---
layout: post
title:  "Phoenix Typed Forms"
date:   2024-03-13 13:16:27 -0400
cover: /assets/phx-banner.png
categories: elixir phoenix ui
---

I've been working with Phoenix for a while now and I'm a big fan of how they handle user input with [forms](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html). What I like about them specifically is that they can be [backed by a changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html). The changeset can compare the user's inputs against your validation rules, and if there are errors it picks them up. Once the changeset converted to a form with `to_form/1`, Phoenix can render those errors back out to the user without needing to add a special handler for each failure case. A nice thing about changesets is that you can create them from an ecto schema, effectively turning them into typed struct validators. The changeset can then automatically cast the inputs into the correct types based off the schema, saving you from having to write a bunch of boilerplate code to do it yourself.

This effectively means that changeset-backed forms are "typed forms" - You can guarantee the types of your form inputs, and have it cast user input for you automatically, and automatically show the user whenever something goes wrong. Combine this with LiveView, and you've got a super responsive UI that can give the user immediate feedback on their forms without needing to submit beforehand, and at the end of it all the frontend has that data packed neatly into a struct it can do whatever it wants with.

But getting here wasn't straightforward for me, I had to do a lot of reading and experimenting to figure out how to create these typed forms. I'm going to share what I've learned with you, and show you how you can easily make your own changeset-backed forms that can be used with LiveView. 

## Typed Forms with Ecto

Let's say we want to make a form for a checkout page. We'll use the simplest case possible and assume we're just asking the user for a quantity to buy. Our schema looks like this:

{% highlight elixir %}
defmodule CheckoutForm do
  use TypedEctoSchema
  import Ecto.Changeset

  @primary_key false
  typed_embedded_schema do
    field(:qty, :integer)
  end

  def changeset(%__MODULE__{} = existing, attrs) do
    fields = __MODULE__.__schema__(:fields)

    cast(existing, attrs, fields)
  end
end
{% endhighlight %}

Rendering a form is simple, we can take the result of our changeset and pass it to the form component in the heex:

{% highlight elixir %}
# Set up the socket
assign(socket, changeset: %CheckoutForm{} |> CheckoutForm.changeset(%{qty: 1}))
{% endhighlight %}

{% highlight html %}
<!-- Render the form in the heex -->
<.form
  :let={f}
  for={@changeset}
  phx-change="form_updated"
>
{% endhighlight %}

Here, a form is being created on the fly (named `f`) for the changeset we pass into this component. Now this is pretty much exactly what I want to get out of these forms, we pass in a `@changeset` to the form that we can get from our `changeset/2` function, and the form handles the rest. It's very simple. So of course this approach is actually discouraged in the [Phoenix docs](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#form/1-using-the-for-attribute)! Phoenix says this approach is bad for two reasons:

> 1. LiveView can better optimize your code if you access the form fields using `@form[:field]` rather than through the let-variable `form`
> 
> 2. Ecto changesets are meant to be single use. By never storing the changeset in the assign, you will be less tempted to use it across operations

I don't have much to add to the first point. `:let` seems convenient but I don't know enough about how Phoenix works under the hood to say if something is performant or not. But I can see the second point being a problem - The changeset *result* is meant to be a single use object, and it's not really a good idea to be storing temporary things in your LiveView socket assigns. But, I want to use my changeset *function* to validate my form whenever it updates, because the actual act of validation doesn't change between form updates - that would just be a really weird user experience.

But there has to be a way we can use changesets to power our forms. How can we avoid these pitfalls? Well if we knew what changeset function to apply on itself, we wouldn't ever have a need to pass around the changeset - When our form updates in LiveView, we can call the changeset function in our event handler. Then we should be able to take that changeset, pass it into `to_form/1`, and assign it as our `@form`, so we can use `for={@form}` in our heex. So our LiveView event handler looks like this:

{% highlight elixir %}
def handle_event("form_updated", %{"checkout_form" => form_params}, socket) do
  changeset = CheckoutForm.changeset(%CheckoutForm{}, form_params)

  {:noreply, assign(socket, form: to_form(changeset))}
end
{% endhighlight %}

And our heex looks like this:

{% highlight html %}
<.form
  for={@form}
  phx-change="form_updated"
>
  <!-- Adding in input field -->
  <.input
    field={@form[:qty]}
    type="number"
    step="1.0"
    label="Qty"
    min="0"
    required
  />
</.form>
{% endhighlight %}

But our values don't seem to be showing up in the form. What's going on? Well, when `to_form/1` is called, it copies over data from the changeset to the form struct, but it doesn't seem to pick up the changes! We need to commit the changes first to the changeset before we can convert it to a form. We can do this by applying the `:new` action to the changeset. But we should remember that `apply_action/2` can actually fail. So our LiveView code becomes this:

{% highlight elixir %}
def handle_event("form_updated", %{"checkout_form" => form_params}, socket) do
  form = %CheckoutForm{}
  |> changeset(form_params)
  |> apply_action(:new)
  |> case do
    {:ok, val} -> changeset(val, %{}) |> to_form()
    {:error, cs} -> to_form(cs)
  end

  {:noreply, assign(socket, form: form)}
end
{% endhighlight %}

And we can tweak the heex to show us the `data` of the form. The data is coming from the changeset as a result of the action we applied to it:

{% highlight elixir %}
<.form
  for={@form}
  phx-change="form_updated"
>
  <.input
    field={@form[:qty]}
    type="number"
    step="1.0"
    label="Qty"
    min="0"
    value={@form.data.qty}
    required
  />
</.form>
{% endhighlight %}

We perform the double `changeset/2` call to update the form with the new data. This is because the `to_form/1` function doesn't care about the incoming `changes` of the changeset, just the `data` that's already there in the field. So our first call to `changeset/2` is gets the changes, but since we haven't applied the actions to get our struct into `data`, `to_form/2` doesn't care about them. I'm not 100% sure so if you know more or have a better solution, please let me know!

And you might also notice, we don't need to specifcally handle errors! We can just turn the changes into a form, and the input component will know how to render the errors out to the user for that field.

## Typed Forms with Macros

So our event handler is doing way too much. Let's break it up. It would be better if the form was responsible for owning all of it's changeset functionality:

{% highlight elixir %}
defmodule CheckoutForm do
  use TypedEctoSchema
  import Ecto.Changeset

  @primary_key false
  typed_embedded_schema do
    field(:qty, :integer)
  end

  def changeset(%__MODULE__{} = existing, attrs) do
    fields = __MODULE__.__schema__(:fields)

    cast(existing, attrs, fields)
  end

  # Added for convenience
  def new_form(), do: new_form(%{})
  def new_form(attrs), do: update_form(attrs)

  def update_form(attrs) do
    case apply_changeset(attrs) do
      {:ok, val} -> changeset(val, %{}) |> to_form()
      {:error, cs} -> to_form(cs)
    end
  end

  defp apply_changeset(attrs) do
    %__MODULE__{}
    |> changeset(attrs)
    |> apply_action(:new)
  end
end

# LiveView
def handle_event("form_updated", %{"checkout_form" => form_params}, socket) do
  form = CheckoutForm.update_form(form_params)

  {:noreply, assign(socket, form: form)}
end
{% endhighlight %}

Now our handler is way cleaner. But hold on, wouldn't this behavior also be able to be used for other forms? We could have a `UserForm`, `ProductForm`, etc. and they would all have the same behavior, just our schema would change. But since nothing in our behavior specifically depends on anything in the schema, we could use a macro to inject this behavior into our other form modules. 

Elixir macros are a lot different than the C style preprocessor macros you might be familiar with. In fact, they're more like Lisp macros - They can take code as input and produce code as output. We don't need to use a macro here to add syntax or create a DSL, we can just use them to inject some common bits of code we need into all our form modules. Teasing a macro apart from our module is trivial with a `use` macro:

{% highlight elixir %}
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset

      @primary_key false

      def changeset(%__MODULE__{} = existing, attrs) do
        fields = __MODULE__.__schema__(:fields)

        cast(existing, attrs, fields)
      end

      def new_form(), do: new_form(%{})
      def new_form(attrs), do: update_form(attrs)

      def update_form(attrs) do
        case apply_changeset(attrs) do
          {:ok, val} -> changeset(val, %{}) |> to_form()
          {:error, cs} -> to_form(cs)
        end
      end

      defp apply_changeset(attrs) do
        %__MODULE__{}
        |> changeset(attrs)
        |> apply_action(:new)
      end
    end
  end
end

defmodule CheckoutForm do
  use PhoenixTypedForm

  typed_embedded_schema do
    field(:qty, :integer)
  end
end
{% endhighlight %}

Wow! Look at how small our form module becomes! And our LiveView code doesn't need to change. We've cleanly separated our behavior from our schema, and now all we have to do to create more forms is to use our `PhoenixTypedForm` module. And that's the simplest version of our macro, there's a lot we can do from here to make it better and cover more use cases.

## Extending the macro

A real frontend is going to require more functionality than just simple validation. I can think of a few off the top of my head:

1. Default values - Creating a `new_form/0` will give you an entirely blank, empty form but chances are we can autofill some data for the user depending on the screen. We should be able to support a set of `default_values` that will be used to fill in the form if the developer doesn't provide them.
  
2. Custom changesets - We're not gonna get very far by just validatng types! Think about the example form with name and age - should age be negative? Probably not! We should be able to override the default `changeset/2` with a custom one that can add more complex validation rules.

3. Runtime constraints - The final issue is a bit complex. Let's reconsider our checkout form. Let's say we want quantity to both be positive, and less than the remaining stock. The problem with this is our macro runs at compile time, so how could we handle constraints that change over time or from user to user? We should be able to support a set of optional runtime `constraints` that can be used to validate the form's input against even more complex rules.

### Default values

Let's enable the macro to have a `default_values` option passed to it:

{% highlight elixir %}
defmodule PhoenixTypedForm do
  defmacro __using__(opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset

      @primary_key false

      Module.put_attribute(
        __MODULE__,
        :default_values,
        Keyword.get(unquote(opts), :default_values, %{})
      )
      
      def new_form(), do: new_form(%{})

      def new_form(attrs) do
        @default_values |> Map.merge(attrs) |> update_form()
      end

      # ...
    end
  end
end

defmodule CheckoutForm do
  use PhoenixTypedForm, default_values: %{qty: 1}

  typed_embedded_schema do
    field(:qty, :integer)
  end
end
{% endhighlight %}

`Module.put_attribute/3` is a way to set a module attribute at compile time. We're using it to store the `default_values` that are passed to the macro. We then merge these with the `attrs` passed to `new_form/1` to create the initial form object. Now if we call `MyForm.new_form()`, we'll get a form object with the default values filled in.

### Custom changesets

It would be awesome if we could override our default `changeset/2` with a custom one. We can do this with `defoverride`, but we'll need to change up how our macro looks a bit:

{% highlight elixir %}
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import unquote(__MODULE__)

      @primary_key false
    end
  end

  defmacro def_typed_form(opts \\ []) do
    quote location: :keep do
      Module.put_attribute(
        __MODULE__,
        :default_values,
        Keyword.get(unquote(opts), :default_values, %{})
      )

      def changeset(existing, attrs)

      def changeset(%__MODULE__{} = existing, attrs) do
        fields = __MODULE__.__schema__(:fields)

        cast(existing, attrs, fields)
      end

      defoverridable changeset: 3

      def new_form(), do: new_form(%{}, [])

      def new_form(attrs) do
        @default_values |> Map.merge(attrs) |> update_form()
      end

      def update_form(attrs) do
        case apply_changeset(attrs) do
          {:ok, val} -> changeset(val, %{}) |> to_form()
          {:error, cs} -> to_form(cs)
        end
      end

      defp apply_changeset(attrs) do
        %__MODULE__{}
        |> changeset(attrs, constraints)
        |> apply_action(:new)
      end
    end
  end
end

defmodule CheckoutForm do
  use PhoenixTypedForm

  typed_embedded_schema do
    field(:qty, :integer)
  end

  # Macro brings in the default changeset...
  def_typed_form(default_values: %{qty: 1})

  # ...and we override it here
  def changeset(%__MODULE__{} = existing, attrs) do
    existing
    |> changeset(attrs)
    |> validate_number(:qty, greater_than: 0)
  end
end
{% endhighlight %}

### Runtime constraints

So how would we support a max quantity constraint? We need to allow the `changeset/2` function to support a custom `constraints` argument, and then allow `new_form` and `update_form` to pass them in. Then we can override our `changeset/2` to use the constraints. It'll look really similar to the previous example:

{% highlight elixir %}
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import unquote(__MODULE__)

      @primary_key false
    end
  end

  defmacro def_typed_form(opts \\ []) do
    quote location: :keep do
      Module.put_attribute(
        __MODULE__,
        :default_values,
        Keyword.get(unquote(opts), :default_values, %{})
      )

      def changeset(existing, attrs, constraints \\ [])

      def changeset(%__MODULE__{} = existing, attrs, []) do
        fields = __MODULE__.__schema__(:fields)

        cast(existing, attrs, fields)
      end

      defoverridable changeset: 3

      @spec new_form() :: Phoenix.HTML.Form.t()
      def new_form(), do: new_form(%{}, [])

      @spec new_form(map(), Keyword.t()) :: Phoenix.HTML.Form.t()
      def new_form(attrs, constraints \\ []) do
        @default_values |> Map.merge(attrs) |> update_form(constraints)
      end

      @spec update_form(map(), Keyword.t()) :: Phoenix.HTML.Form.t()
      def update_form(attrs, constraints \\ []) do
        case apply_changeset(attrs, constraints) do
          {:ok, val} -> changeset(val, %{}, constraints) |> to_form()
          {:error, cs} -> to_form(cs)
        end
      end

      defp apply_changeset(attrs, constraints \\ []) do
        %__MODULE__{}
        |> changeset(attrs, constraints)
        |> apply_action(:new)
      end
    end
  end
end

defmodule MyForm do
  use PhoenixTypedForm

  # Accept qty
  typed_embedded_schema do
    field(:qty, :integer)
  end

  def_typed_form(default_values: %{qty: 1})

  # Override the default changeset - Accepts max_qty constraint
  def changeset(%__MODULE__{} = existing, attrs, max_qty: max_qty) do
    existing
    |> changeset(attrs)
    |> validate_number(:age, greater_than: 0, less_than: max_qty)
  end
end
{% endhighlight %}

Now, if your page fetched the remaining stock from the server on load, you could pass it into the form when you create or update the form. If the user tries to submit a quantity greater than the remaining stock, the form will show an error. Now, you could also keep changing your max_qty constraint as the user interacts with the page, and the form show an error whenever that constraint is violated. But we talked about this earlier, the actual act of validation really shouldn't change between form updates because it would be a really weird user experience.

## Putting it all together - The Phoenix Typed Form hex package

I've taken the sum of these ideas then tested, documented, and published them as a hex package called `phoenix_typed_form`. You can find it [here](https://hex.pm/packages/phoenix_typed_form). 

We currently use this in production at [Belay](https://www.withbelay.com/). I hope you find it as useful as we do!
