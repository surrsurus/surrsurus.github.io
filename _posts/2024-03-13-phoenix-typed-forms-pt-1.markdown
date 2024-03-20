---
layout: post
title:  "Phoenix Typed Forms - Introduction"
date:   2024-03-13 13:16:27 -0400
cover: /assets/phx-banner.png
categories: elixir phoenix ui
---

For the past few years I've been building web applications with [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html). Before this, I was a React + Redux developer and I found Phoenix easy to pick up. I've been able to build complex web applications faster and with less code than I typically would need in React.

That's not to say everything I've encountered in Phoenix is straightforward. I've struggled with forms, which surprised me. Forms seemed simple from the docs but proved challenging in practice. They involve collecting and validating user data, and that can be quite tricky to do right.

Before continuing, I'm going to assume you have a basic understanding of Phoenix LiveView, Ecto, and Elixir. In this article I'll be talking about how to create forms in Phoenix, and how we can leverage [changesets](https://hexdocs.pm/ecto/Ecto.Changeset.html) to make our forms more powerful. I'll also talk about how we can do this in a way that's performant and keeps our frontend uncoupled with our database schemas. In the next article, we'll go over how we can extract this behavior into a macro we can use across all our forms.

## Phoenix Forms

Phoenix provides a `Phoenix.Component.form/1` component that renders forms. All we need to do is give it a `Phoenix.HTML.Form` struct, and it will render the form. Phoenix also gives us a way to get that form struct - it gives us a `to_form/1` method that can convert a map or an Ecto changeset into a form struct. So forms are split into two types - map-backed forms and changeset-backed forms. What are the major differences?

### Map-Backed Forms

> **Warning: Don't Use Map-Backed Forms**
>
> If you're skimming this article for how to properly do forms in Phoenix, don't use map-backed forms. I'm going to talk about them to show their flaws and why you shouldn't use them. If you're looking for the right way to do forms in Phoenix, skip to [Performant Changeset-Backed Forms](https://surrsurus.github.io/elixir/phoenix/ui/2024/03/13/phoenix-typed-forms-pt-1.html#performant-changeset-backed-forms).
{: .block-danger }

Let's start by creating a hypothetical form we want users to fill out. Say we want to let users place orders for a product. We'll want to record our orders to the database so we can keep track of them. A basic order schema would look like this:

{% highlight elixir %}
# order.ex
defmodule Order do
  use Ecto.Schema

  schema "order" do
    field(:order_id, :string) # The id of the order
    field(:qty, :integer) # The quantity of the order
  end

  @type t() :: %__MODULE__{
    __meta__: Ecto.Schema.Metadata.t(),
    qty: integer(),
    order_id: String.t()
  }
end
{% endhighlight %}

The most basic way to create a form is to use a map to represent the fields we want to capture. If we wanted to create a map-backed form for our order schema, we'd start with a map that looks like this:

{% highlight elixir %}
order_map = %{"id" => "", "qty" => ""}
{% endhighlight %}

Now, we'll pass this into `to_form/1` to get a form struct. From experience, `to_form/1` won't care about the values of the map, just the keys, so they can be blank. So we can pass this map into `to_form/1` and it will give us a form struct. Let's put this together in a LiveView:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView
  use Phoenix.Component

  def mount(_params, _session, socket) do
    order_map = %{"id" => "", "qty" => ""}
    form = to_form(order_map)

    {:ok, assign(socket, form: form)}
  end
end
{% endhighlight %}

Our `mount/3` executes whenever someone loads the page for the first time, or on refresh. When users come to our page we'll generate the form, and with our form struct in the socket, we can render it in our HEEx:

{% highlight elixir %}
<!-- order_live.html.heex -->
<.form
  for={@form}
  phx-change="form_updated"
  phx-submit="form_submitted"
>
  <.input field={@form[:field]} />
  <button type="submit">Submit</button>
</.form>
{% endhighlight %}

So we've manually defined what fields our form has, created a form struct from the map, and rendered it out. We'll get the form data back in our event handlers, one for when the form updates and the other for when the user submits the form.

There are a few problems with this approach. Our form data will come back to us as a map with the same keys as `order_map`, but the values are strings the user entered in the form. Because of this, we have to manually handle all the type validation and any casting ourselves. We'll need to write event handlers to do this:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView
  use Phoenix.Component

  def mount(_params, _session, socket) do
    order_map = %{"id" => "", "qty" => ""}
    form = to_form(order_map)

    {:ok, assign(socket, form: form)}
  end

  def handle_event("form_updated", %{"form" => form_params}, socket) do
    # Convert the qty to an integer
    qty = String.to_integer(form_params["qty"])

    # Validate the qty
    socket = if is_integer(qty) && qty > 0 do
      socket
    else
      # Show the user an error
      put_flash(socket, :error, "Qty must be a positive integer")
    end

    {:noreply, socket}
  end

  def handle_event("form_submitted", %{"form" => form_params}, socket) do
    # Validate the qty... again
    socket = if is_integer(qty) && qty > 0 do
      # Insert into db with Ecto
      Repo.insert(%Order{order_id: form_params["id"], qty: qty})

      socket
    else
      # Show the user an error
      put_flash(socket, :error, "Qty must be a positive integer")
    end

    {:noreply, socket}
  end
end
{% endhighlight %}

So as our user types into the form, our `form_updated` event handler will be called. We'll need to convert the `qty` to an integer, and then validate it. If it's not an integer or it's less than zero, we'll show an error to the user. Then, when the user submits the form, `form_submitted` is called. It does the same validation as `form_updated`, and then inserts the order into the database.

Let's take a step back. Our frontend got complicated fast. We're taking the business concept of an order and representing it as a map and using that to build a form:

{% highlight elixir %}
order_map = %{"id" => "", "qty" => ""}
{% endhighlight %}

We're duplicating the knowledge of the schema and letting it's structure leak into our frontend, which is coupling them. Our frontend needs to implicitly know the types of the fields in the schema, so it can map the form data. Then, the frontend is also doing it's own validation of the data - in multiple places. We also then have to insert our order or show the user errors if possible. 

That's a lot of responsibilities for a LiveView, on top of it's usual responsibilities such as tracking state, handling events, and rendering the UI. This is just a mess, we balled everything up and threw it into the frontend code. There is a complete lack of context separation. If our schema ever changes in the slightest, our form will break and we'll need to update code in many places to accommodate that change. 

The problem with maps is that they are entirely un-opinionated, and you're left to solve all of the challenges above as a one-off each time. But changesets provide a framework that, while it takes time to learn, give us a clean strategy for separating these concerns, and enabling code reuse.

### Changeset-backed forms

#### Ecto isn't Just for Databases

You might be accustomed to thinking that Ecto is the database library for Elixir. That's not entirely true. Yes, it does interact with databases, but it's also good at mapping and validating data. Ecto has the changeset concept, which is a way to validate and cast data. You typically use it to validate data before you insert it into the database, but it's not limited to that.

> **Do I Even Need a Database?**
>
> You could use Ecto entirely without a database! Ecto supports [embedded schemas](https://hexdocs.pm/ecto/embedded-schemas.html). With an embedded schema, data can live in memory instead. This means you can use Ecto as purely a data validation layer. All you really need is a schema and a changeset to power your forms.
{: .block-tip }

So let's make a changeset for our order. We created an order earlier that looks like this:

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
end
{% endhighlight %}

As I said earlier, we'll get back all our user's form data as strings. We'll need to cast them to the correct types before we can insert them into the database. Ecto makes it easy to do this with a changeset. We can define a changeset function that will take our form data and `cast/3` it to the correct types. Here's what that looks like:

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

All this changeset does is cast `attrs` to the types defined by the fields of the schema. If anything doesn't match or can't be cast, the changeset will error on that field. And that's it!

As we talked about earlier, we can convert a changeset directly to a form - even if that changeset has errors! This is great, because the form component can use the errors to show the user what they did wrong, inline with the field that caused the error. Changesets can be created directly from the schema, effectively turning them into typed struct validators. The changeset can then cast the inputs into the correct types based off the schema, saving you from having to write a bunch of boilerplate code to do it yourself.

#### Creating a Changeset-Backed Form

So we have our order schema, and we have a changeset function that can create a changeset to validate and cast our form inputs. How do we use this in our LiveView? Phoenix makes rendering a form easy, we can take the result of our changeset and pass it to the form component in the HEEx:

{% highlight elixir %}
# order_live.ex
def mount(_params, _session, socket) do
  # Start with an empty order
  changeset = Order.changeset(%Order{}, %{})

  {:ok, assign(socket, changeset: changeset)}
end
{% endhighlight %}

{% highlight html %}
<!-- order_live.html.heex -->
<.form
  :let={f}
  for={@changeset}
  phx-change="form_updated"
  phx-submit="form_submitted"
>
  <.input field={@form[:field]} />
  <button type="submit">Submit</button>
</.form>
{% endhighlight %}

Here, a form is being created on the fly (named `f`) for the changeset we pass into this component. Now this is pretty much exactly what I want to get out of these forms, we pass in a `@changeset` to the form that we can get from our `changeset/2` function, and the form handles the rest. So of course this approach is actually discouraged in the [Phoenix docs](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#form/1-using-the-for-attribute)! The docs say this approach is bad for two reasons:

> 1. LiveView can better optimize your code if you access the form fields using `@form[:field]` rather than through the let-variable `form`
> 
> 2. Ecto changesets are meant to be single use. By never storing the changeset in the assign, you will be less tempted to use it across operations

I don't have much to add to the first point. `:let` seems convenient but I don't know enough about how Phoenix works under the hood to say if something is performant or not. I can see the second point being a problem - The changeset *result* is meant to be a single use object, and it's not really a good idea to be storing temporary things in your LiveView socket assigns. So what we really want is to use our changeset *function* to validate the form whenever it updates, because the actual act of validation doesn't change between form updates - that would just be a really weird user experience.

There has to be a way we can use changesets to power our forms.

## Performant Changeset-Backed Forms

If we knew what changeset function to apply on our forms, we wouldn't ever have a need to pass around the changeset - When our form updates in LiveView, we can call the changeset function in our event handler. Then we should be able to take that, pass it into `to_form/1`, and assign it as our `@form`, so we can use the `for={@form}` attribute in our HEEx instead of `:let`. So our LiveView event handler looks like this:

{% highlight elixir %}
# order_live.ex
def handle_event("form_updated", %{"order" => form_params}, socket) do
  # Still creating the changeset...
  changeset = Order.changeset(%Order{}, form_params)

  # ...but then we can convert it to a form
  {:noreply, assign(socket, form: to_form(changeset))}
end

def handle_event("form_submitted", %{"order" => form_params}, socket) do
  # Still creating the changeset...
  changeset = Order.changeset(%Order{}, form_params)

  # ... so we can insert it into the database
  case Repo.insert(changeset) do
    {:ok, _} -> {:noreply, socket}
    {:error, changeset} -> {:noreply, assign(socket, form: to_form(changeset))}
  end
end
{% endhighlight %}

And our HEEx looks like this:

{% highlight html %}
<!-- order_live.html.heex -->
<.form
  for={@form}
  phx-change="form_updated"
  phx-submit="form_submitted"
>
  <!-- Fleshing out the input field -->
  <.input
    field={@form[:qty]}
    type="number"
    step="1.0"
    label="Qty"
    min="0"
    required
  />
  <button type="submit">Submit</button>
</.form>
{% endhighlight %}

However our values don't seem to be showing up in the form. What's going on? Well, when `to_form/1` is called, it copies over data from the changeset to the form struct, but it doesn't seem to pick up the changes! We need to commit the changes first to the changeset before we can convert it to a form. We can do this by applying the `:new` action to the changeset, but we should remember that `apply_action/2` can fail. So our LiveView code becomes this:

{% highlight elixir %}
# order_live.ex
def handle_event("form_updated", %{"order_form" => form_params}, socket) do
  form = %OrderForm{}
  |> changeset(form_params)
  |> apply_action(:new)
  |> case do
    {:ok, val} -> changeset(val, %{}) |> to_form()
    {:error, cs} -> to_form(cs)
  end

  {:noreply, assign(socket, form: form)}
end

def handle_event("form_submitted", %{"order_form" => form_params}, socket) do
  form = %OrderForm{}
  |> changeset(form_params)
  |> apply_action(:new)
  |> case do
    {:ok, val} -> changeset(val, %{}) |> to_form()
    {:error, cs} -> to_form(cs)
  end

  case Repo.insert(form) do
    {:ok, _} -> {:noreply, socket}
    {:error, changeset} -> {:noreply, assign(socket, form: to_form(changeset))}
  end
end
{% endhighlight %}

And we can tweak the HEEx to show us the `data` of the form. The data is coming from the changeset as a result of the action we applied to it:

{% highlight elixir %}
<!-- order_live.html.heex -->
<.form
  for={@form}
  phx-change="form_updated"
  phx-submit="form_submitted"
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
  <button type="submit">Submit</button>
</.form>
{% endhighlight %}

We perform the double `changeset/2` call to update the form with the new data. This is because the `to_form/1` function doesn't care about the incoming `changes` of the changeset, just the `data` that's already there in the field. So our first call to `changeset/2` is gets the changes, but since we haven't applied the actions to get our struct into `data`, `to_form/2` doesn't care about them. I'm not 100% sure so if you know more or have a better solution, please let me know!

And you might also notice, we don't need to specifically handle errors! We can just turn the changes into a form, and the input component will know how to render the errors out to the user for that field.

## Conclusion

So we've seen how we can use changesets to power our forms in Phoenix. We can use the changeset to validate and cast our form inputs, and then our form can render any errors if present in the changeset. Best of all, we can do this in a way that lets us be performant and keep our frontend uncoupled with our database schemas. I would definitely advise you to jump right for the changeset-backed forms, and completely avoid the map-backed forms. Changeset-backed forms still work even when you don't have a database.

This is awesome, but we could go further. Any other forms we make are going to want that same method to validating and casting inputs. We don't want to copy around the same behavior everywhere. If the behavior for forms changes or has a feature added, we'd have to change it in a lot of places. But Elixir has a great way to handle this - macros.

In the next [post](https://surrsurus.github.io/elixir/phoenix/ui/2024/03/14/phoenix-typed-forms-pt-2.html), I'll talk about how we can extract this behavior into a macro we can use across all our forms. 
