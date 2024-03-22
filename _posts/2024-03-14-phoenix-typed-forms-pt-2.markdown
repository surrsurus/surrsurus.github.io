---
layout: post
title:  "Phoenix Typed Forms Pt 2 - Macros"
date:   2024-03-14 14:16:27 -0400
cover: /assets/phx-banner.png
categories: elixir phoenix ui
---

This is a continuation of my [previous post](https://surrsurus.github.io/elixir/phoenix/ui/2024/03/13/phoenix-typed-forms-pt-1.html) where I go over the different ways to create a form in [Phoenix](https://www.phoenixframework.org/). In this post, we're going to take the changeset-backed form behavior we built and turn it into a macro that we can use across all our form modules. Then, we're going to extend the macro to support default values, custom changesets, and runtime constraints.

## Elixir Macros

If you've used preprocessor macros in C or C++ before, you might be a little wary of macros in general. There's not much to worry about - Elixir macros are more akin to Lisp macros than C macros. Elixir macros are functions that run at compile-time and transform Elixir code. They're used to extend the language itself and introduce new syntax or behaviors. Macros work a lot like regular elixir functions, but they produce code as output - they're a great way to reduce boilerplate and make your code DRY-er. That's exactly what we're going to use them for.

## The Phoenix Typed Form Macro

So let's take a look at our form module from the previous post:

{% highlight elixir %}
# order.ex
defmodule Order do
  use Ecto.Schema

  schema "order" do
    field(:order_id, :string)
    field(:qty, :integer)
  end

  @type t() :: %__MODULE__{
    __meta__: Ecto.Schema.Metadata.t(),
    qty: integer(),
    order_id: String.t()
  }

  def changeset(%__MODULE__{} = existing, attrs) do
    fields = __MODULE__.__schema__(:fields)

    cast(existing, attrs, fields)
  end
end
{% endhighlight %}

If our schema ever changes, we'll never need to update our `changeset/2` function. It's going to work for any schema, because it's not caring about the individual fields. This is a great candidate for a macro. Imagine if we have more forms in our application - we could have a login form for instance. It would still want the same changeset function, and our macro would allow us to reuse the same code across both form modules. We can create a macro that will inject the `changeset/2` function into our form module. Here's what our macro will look like:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use Ecto.Schema
      use Phoenix.Component
      import Ecto.Changeset
      import PhoenixTypedForm
    end
  end

  defmacro def_typed_form(_opts) do
    quote location: :keep do
      use Ecto.Schema
      import Ecto.Changeset

      def changeset(%__MODULE__{} = existing, attrs) do
        fields = __MODULE__.__schema__(:fields)

        cast(existing, attrs, fields)
      end
    end
  end
end
{% endhighlight %}

We're going to call this macro `PhoenixTypedForm`, because our schema is going to define the types we expect to get from the form, and the changeset will enforce it. We'll give it the `Phoenix` prefix because that adheres to [Hex.pm's naming guidelines](https://hex.pm/docs/publish) since we're providing functionality on top of the existing Phoenix forms.

So let's talk about the syntax of macros. We've got two different macros here - `__using__` and `def_typed_form`. When we `use` a macro, we're implicitly calling the `__using__` function in the macro module. You've seen this before, when we `use Ecto.Schema` we're not bringing in the `Ecto.Schema` module, we're executing a macro. In our case, we're using the `__using__` block to inject some common imports. We can also use the `__using__` block to import the `PhoenixTypedForm` module to make `def_typed_form` available to the caller. This means we don't need to `use PhoenixTypedForm` and also `import PhoenixTypedForm` at the same time.

The reason why we don't put all of `def_typed_form` into the use block is because of compile time constraints. Our macros run at compile time, and we're referencing the `__MODULE__` struct that `use Ecto.Schema` creates for us before we've given it a chance to actually create it. We'll see a `Order.__struct__/0 is undefined` error pop up if we try to do this. So the idea is we're letting the module invoke the macro so that `__MODULE__` will be able to reference the struct.

The `quote` block inside our macros is where we define the code we want to inject. The `location: :keep` option is a way to tell the compiler to keep the line numbers of the code we're injecting the same as the line numbers of the macro. This is useful for debugging - if something blows up in the macro the Elixir runtime will let us know where. So in summary what this code will do is bring in our `changeset/2` function to our form module. Our form module gets a lot simpler:

{% highlight elixir %}
# order.ex
defmodule Order do
  use PhoenixTypedForm

  schema "order" do
    field(:order_id, :string)
    field(:qty, :integer)
  end

  @type t() :: %__MODULE__{
    __meta__: Ecto.Schema.Metadata.t(),
    qty: integer(),
    order_id: String.t()
  }

  # Bring in our changeset
  def_typed_form()
end
{% endhighlight %}

Now we can use our `Order` module in our LiveView:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView
  import Ecto.Changeset

  def mount(_params, _session, socket) do
    order_changeset = Order.changeset(%Order{}, %{})
    form = to_form(order_changeset)

    {:ok, assign(socket, form: form)}
  end

  def handle_event("form_updated", %{"order_form" => form_params}, socket) do
    form = %Order{}
    |> changeset(form_params)
    |> apply_action(:new)
    |> case do
      {:ok, val} -> changeset(val, %{}) |> to_form()
      {:error, cs} -> to_form(cs)
    end

    {:noreply, assign(socket, form: form)}
  end
end
{% endhighlight %}

And that's it! Now that we've cleanly separated our behavior from our schema, all we have to do to create more forms is to use our `PhoenixTypedForm` module. This is nice because our `Order` becomes a record of what we expect to be getting from the forms, the macro enforces it, and the LiveView only has to manage the form state. Each piece doesn't have to care about what the other looks like or what they're doing, and they each have a clear responsibility.

Taking a step back, we can see that `handle_event` is doing too much. We're creating a new form, applying the changeset, converting it to a form, and handling errors all in one function. Similarly to our changeset function, we'll be copying and pasting that behavior everywhere and that's not ideal. Another thing to note is that because of this function, we need to import `Ecto.Changeset` into our LiveView, which is a definite code smell - this hints that we should be moving this into the macro.

### Some Minor Improvements

We can clean up our code a little bit by adding in `new_form/0` and `update_form/1` functions to our macro. This will remove the need for our LiveViews to be brining in changeset functionality.

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use Ecto.Schema
      import Ecto.Changeset
      import PhoenixTypedForm
    end
  end

  defmacro def_typed_form(_opts) do
    quote location: :keep do
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
{% endhighlight %}

And there it is, the trick is that new forms and updated forms are basically the same thing since we create them with the changeset. The difference is that we update forms with new parameters from the user. We've also given the ability to create a form with default options by calling `new_form/1`. Now our LiveView code is a lot cleaner:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView
  use Phoenix.Component

  def mount(_params, _session, socket) do
    {:ok, assign(socket, form: Order.new_form())}
  end

  def handle_event("form_updated", %{"order_form" => form_params}, socket) do
    {:noreply, assign(socket, form: Order.update_form(form_params))}
  end
end
{% endhighlight %}

We've turned our mount and event handler into one line functions. This is a lot cleaner and easier to read than what we had before, and we've also removed the need to import `Ecto.Changeset` into our LiveView. Our behavior is moving into our macro, and as a result our LiveView keeps getting simpler.

## Extending the Macro Further

So we've got a macro to help us create forms, but a real frontend is going to require more functionality than simple validation. I can think of a few off the top of my head:

1. Default values - Creating a `new_form/0` will give you an blank, empty form but chances are we can autofill some data for the user. For instance, an order might have a default quantity of one. We should be able to support a set of `default_values` that's used to fill in the form if the developer doesn't provide them.

2. Custom changesets - We're not gonna get very far by only validating types! Should an order's quantity be negative? Probably not! We should be able to override the default `changeset/2` with a custom one that can add more complex validation rules.

3. Runtime constraints - The final issue is a bit complex. Let's reconsider our checkout form. Let's say we want quantity to both be positive, and less than the remaining stock. The problem with this is our macro runs at compile time, and our stock is always changing. How can we prevent the user from buying more inventory than we have? We should be able to support a set of optional runtime `constraints` that's used to validate the form's input against even more complex rules.
4. 
### Default Values

Let's enable the macro to have a `default_values` option passed to it:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import PhoenixTypedForm
    end
  end

  defmacro def_typed_form(opts \\ []) do
    quote location: :keep do
      Module.put_attribute(
        __MODULE__,
        :default_values,
        Keyword.get(unquote(opts), :default_values, %{})
      )
      
      def new_form(), do: new_form(%{})
      def new_form(attrs), do: @default_values |> Map.merge(attrs) |> update_form()

      # ...
    end
  end
end
{% endhighlight %}

{% highlight elixir %}
# order.ex
defmodule Order do
  use PhoenixTypedForm

  schema "order" do
    field(:order_id, :string)
    field(:qty, :integer)
  end

  @type t() :: %__MODULE__{
    __meta__: Ecto.Schema.Metadata.t(),
    qty: integer(),
    order_id: String.t()
  }

  def_typed_form(default_values: %{qty: 1})
end
{% endhighlight %}

We're using some new macro features here, namely `Module.put_attribute/3` and `unquote`. `Module.put_attribute/3` is a way to set a module attribute at compile time. An attribute is a piece of metadata associated with a module, and is accessible at runtime. They can be used as constants, but also as annotations and temporary storage during compilation. We're using it as a constant to store the `default_values` that get passed to the macro. We then merge these with the `attrs` passed to `new_form/1` to create the initial form object. Now if we call `MyForm.new_form()`, we'll get a form object with the default values filled in. We also still have the ability to optionally pass these in to `new_form/1` at runtime if we so desire, and they'll override the defaults.

We use the `unquote` function within macros to inject dynamic values into the generated code. When you're defining a macro and want to include a value that is determined at runtime, you can use unquote to splice that value directly into the code. Here we're using it to inject our `opts`, which is a keyword list of options that the user can pass to the `def_typed_form` macro. We're using the `unquote` function to inject the value of `opts` into the generated code. Our `default_values` is an option that the user can pass to the macro.

### Custom Changesets

It would be awesome if we could override our default `changeset/2` with a custom one. We can do this with Elixir's `defoverridable`:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import PhoenixTypedForm
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
      def new_form(attrs), do: @default_values |> Map.merge(attrs) |> update_form()


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
{% endhighlight %}

{% highlight elixir %}
# order.ex
defmodule Order do
  use PhoenixTypedForm

  schema "order" do
    field(:order_id, :string)
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

`defoverridable` is our new keyword, but it's not macro specific. `defoverridable` marks the given function as able to be overridden. Overriden functions are lazily evaluated, which means that the function that gets called is determined at runtime.

This is a way to allow the user to override the default `changeset/2` with a custom one. All you need to do is define your own changeset after you bring in the `def_typed_form`. We're also using the `validate_number` function from `Ecto.Changeset` to add a custom validation rule to our changeset. Now, if the user tries to submit a negative quantity, the form will show an error. So we've given the ability to have a default changeset, but also the ability to override it with a custom one for more complex validation rules.

### Runtime Constraints

So how would we support a max quantity constraint? It's likely that our max quantity would change over time as we sell off inventory and get more deliveries. What we want, is we want to allow the `changeset/2` function to support a custom `constraints` argument, and then allow `new_form` and `update_form` to pass them in. Then we can override our `changeset/2` to use the constraints we're giving in the moment. It'll look similar to the previous example:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import PhoenixTypedForm
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

      def new_form(), do: new_form(%{}, [])
      def new_form(attrs, constraints \\ []), do: @default_values |> Map.merge(attrs) |> update_form(constraints)

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
{% endhighlight %}

{% highlight elixir %}
# order.ex
defmodule Order do
  use PhoenixTypedForm

  schema "order" do
    field(:order_id, :string)
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

You can see we've added the optional list of `constraints` to our functions. Now, if your page fetched the remaining stock from the server on load, you could pass it into the form when you create or update the form. If the user tries to submit a quantity greater than the remaining stock, the form will show an error. You could also keep changing your `max_qty` constraint as the user interacts with the page, and the form show an error whenever that constraint is violated. But we talked about this earlier, the actual act of validation shouldn't change between form updates because it would be a weird user experience.

## Putting it all together - The Phoenix Typed Form hex package

I've taken the sum of these ideas then tested, documented, and published them as a hex package called `phoenix_typed_form`. You can find it [here](https://hex.pm/packages/phoenix_typed_form). 

We currently use this in production at [Belay](https://www.withbelay.com/). I hope you find it as useful as we do!
