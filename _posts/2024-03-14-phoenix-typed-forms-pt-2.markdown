---
layout: post
title:  "Phoenix Typed Forms Pt 2 - Macros"
date:   2024-03-14 14:16:27 -0400
cover: /assets/phx-banner.png
categories: elixir phoenix ui
---

This is a continuation of my [previous post](https://surrsurus.github.io/elixir/phoenix/ui/2024/03/13/phoenix-typed-forms-pt-1.html) where I go over the different ways to create a form in [Phoenix](https://www.phoenixframework.org/). In this post, we're going to take the changeset-backed form behavior we built and turn it into a macro that can be used across all of our form modules.

## Elixir Macros

If you've used preprocessor macros in C or C++ before, you might be a little wary of macros in general. But there's not much to be worried about - Elixir macros are more similar to Lisp macros. Macros are functions that are executed at compile-time and transform Elixir code before it's executed. They are a way to extend the language itself and introduce new syntax or behaviors. They're a lot like regular elixir functions, but they produce code as output. They're a great way to reduce boilerplate and make your code DRY-er, and that's exactly what we're gonna use them for. We've identified that all of our forms are going to want the same `changeset/2` function, so now we can use a macro to inject that behavior into all of our form modules.

## The Phoenix Typed Form Macro

So let's take a look at our form module from the previous post:

{% highlight elixir %}
defmodule Order do
  use Ecto.Schema

  @primary_key false
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

If our schema ever changes, we'll never need to change our `changeset/2` function. It's always going to be the same. Think about a totally different form, like a user form. Our changeset is still going to want to pull the fields from the schema and cast data against the fields' types. So our first macro is going to be a simple one that injects this behavior into our form modules:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use Ecto.Schema
      use Phoenix.Component
      import Ecto.Changeset
      # Will bring in the `def_typed_form` macro to the caller module
      import unquote(__MODULE__)
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

We're going to call this macro `PhoenixTypedForm`, because our schema is going to define the types we expect to get from the form, and the changeset will enforce it. 

So let's talk about the syntax of macros. When we `use` a macro, we're implicitly calling the `__using__` function in the macro module. This is where we're going to inject the behavior we want into our form modules. So we just want to inject some default behavior, which in our case is importing the modules we need. We can also use the use block to import the macro module to make `def_typed_form` available to the caller. We're having that macro be invoked so that `__MODULE__` will be able to reference the type created by the Ecto schema. Since macros are compile time, the `use Ecto.Schema` macro won't have executed by the time we try to see what type `__MODULE__` is.

The `quote` block is where we define the code we want to inject. The `location: :keep` option is a way to tell the compiler to keep the line numbers of the code we're injecting the same as the line numbers of the macro. This is useful for debugging - if something blows up in the macro the Elixir runtime will let us know where. So in summary what this code will do is bring in our `changeset/2` function to our form module. Our form module gets a lot simpler:

{% highlight elixir %}
# order.ex
defmodule Order do
  use PhoenixTypedForm

  @primary_key false
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

And that's it! Our form module is now a lot simpler, and our LiveView code doesn't need to change. We've cleanly separated our behavior from our schema, and now all we have to do to create more forms is to use our `PhoenixTypedForm` module. This is really nice because our `Order` becomes a record of what we expect to be getting from the forms, the macro enforces it, and the LiveView just has to manage the form state. Each piece doesn't have to care about what the other looks like or what they're doing. But that `handle_event` is clearly doing too much. Similarly to our changeset function, we'll probably be copying and pasting that everywhere and that's not ideal. But also because of this function, we need to import `Ecto.Changeset` into our LiveView. This hints that we should be moving this functions into our form module, which means we should be extending our macro.

### Some Minor Improvements

We can clean up our code a little bit by adding in `new_form/0` and `update_form/1` functions to our macro. This will remove the need for our LiveViews to be brining in changeset functionality.

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use Ecto.Schema
      import Ecto.Changeset
      import unquote(__MODULE__)
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

We've turned our mount and event handler into one line functions. This is a lot cleaner and easier to read than what we had before, and we've also removed the need to import `Ecto.Changeset` into our LiveView.

## Extending the Macro Further

So we've got a macro to help us create forms, but a real frontend is going to require more functionality than just simple validation. I can think of a few off the top of my head:

1. Default values - Creating a `new_form/0` will give you an entirely blank, empty form but chances are we can autofill some data for the user depending on the screen. We should be able to support a set of `default_values` that will be used to fill in the form if the developer doesn't provide them.
  
2. Custom changesets - We're not gonna get very far by just validatng types! Think about the example form with name and age - should age be negative? Probably not! We should be able to override the default `changeset/2` with a custom one that can add more complex validation rules.

3. Runtime constraints - The final issue is a bit complex. Let's reconsider our checkout form. Let's say we want quantity to both be positive, and less than the remaining stock. The problem with this is our macro runs at compile time, so how could we handle constraints that change over time or from user to user? We should be able to support a set of optional runtime `constraints` that can be used to validate the form's input against even more complex rules.

### Default Values

Let's enable the macro to have a `default_values` option passed to it:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import unquote(__MODULE__)
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

# order.ex
defmodule Order do
  use PhoenixTypedForm

  @primary_key false
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

`Module.put_attribute/3` is a way to set a module attribute at compile time. We're using it to store the `default_values` that are passed to the macro. We then merge these with the `attrs` passed to `new_form/1` to create the initial form object. Now if we call `MyForm.new_form()`, we'll get a form object with the default values filled in. We also still have the ability to optionally pass these in to `new_form/1` at runtime if we so desire, and they'll override the defaults.

### Custom Changesets

It would be awesome if we could override our default `changeset/2` with a custom one. We can do this with Elixir's `defoverride`:

{% highlight elixir %}
# phoenix_typed_form.ex
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import unquote(__MODULE__)
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

### Runtime Constraints

So how would we support a max quantity constraint? We need to allow the `changeset/2` function to support a custom `constraints` argument, and then allow `new_form` and `update_form` to pass them in. Then we can override our `changeset/2` to use the constraints. It'll look really similar to the previous example:

{% highlight elixir %}
defmodule PhoenixTypedForm do
  defmacro __using__(_opts) do
    quote location: :keep do
      use TypedEctoSchema
      import Ecto.Changeset
      import unquote(__MODULE__)
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

Now, if your page fetched the remaining stock from the server on load, you could pass it into the form when you create or update the form. If the user tries to submit a quantity greater than the remaining stock, the form will show an error. Now, you could also keep changing your max_qty constraint as the user interacts with the page, and the form show an error whenever that constraint is violated. But we talked about this earlier, the actual act of validation really shouldn't change between form updates because it would be a really weird user experience.

## Putting it all together - The Phoenix Typed Form hex package

I've taken the sum of these ideas then tested, documented, and published them as a hex package called `phoenix_typed_form`. You can find it [here](https://hex.pm/packages/phoenix_typed_form). 

We currently use this in production at [Belay](https://www.withbelay.com/). I hope you find it as useful as we do!
