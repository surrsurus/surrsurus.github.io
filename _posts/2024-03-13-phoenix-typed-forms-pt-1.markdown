---
layout: post
title:  "Phoenix Typed Forms - Introduction"
date:   2024-03-13 13:16:27 -0400
cover: /assets/phx-banner.png
categories: elixir phoenix ui
---

I've been working with [Elixir](https://elixir-lang.org/) for a couple years now, and in that time I've built a handful of web applications with it - all of them powered by [Phoenix](https://www.phoenixframework.org/). I've used a few different frontend frameworks in the past, and Phoenix (specifically [LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html)) has been my favorite so far. I've been able to build some really complex UIs with it, with a lot less code than I would have needed with a more common framework, and I find I'm able to reason about LiveViews a lot easier than I am able to reason about React + Redux.

But one of the Phoenix concepts I've had problems with is forms. I thought forms were pretty simple - You've got fields you want the user to fill out, you want to validate that data before moving forward, and then you want to do something with that data or tell the user they did something wrong. But in practice, I've found that forms can cause some pain if you aren't familiar with them. I'd like to talk about that pain, and how I've been able to make forms a lot easier to work with in Phoenix.

## What's Phoenix?

Let's take a second to introduce Phoenix if you aren't familiar. Phoenix is a powerful web framework written in Elixir, a functional programming language built on the Erlang VM. Elixir has a great concurrency model it inherits from Erlang - the concept is that each process is a lightweight 'unit of execution' managed by the Erlang VM. Processes are incredibly easy to spin up and tear down, so much so that a common way of handling failure in Elixir is to just let a process die and have it's supervisor (a process that monitors other processes) spin it back up. Phoenix takes advantage of this with [LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) where each user gets their own websocket connection to a LiveView process in the server. And since your users are connected with a websocket, you can push 'live' updates to the client in real time. This is a really powerful model for building web applications, and I'm a really big fan of it. Now back to forms.

## Phoenix Forms

Phoenix provides a `Phoenix.Component.form/1` that can be used to render forms. It's a pretty simple component - you pass in a `Phoenix.HTML.Form` struct, and it will render the form for you. Getting the form struct is also trivial - Phoenix gives us a `to_form/1` that can convert a map or an Ecto changeset into a form struct. Effectively, forms are split into two types - map-backed forms and changeset-backed forms. What's the difference? Ecto changesets are a bit more complicated than maps so let's start with map-backed forms.

### Map-Backed Forms

Say we have a schema for an order in our database. A super basic version of this schema would look like this:

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

Here, we're using [Ecto](https://hexdocs.pm/ecto/Ecto.html) to define a schema for our order. Ecto is powerful and we'll get more into it later, but you won't see exactly why that is just from this tiny little schema. Now, we want to create a form to capture an order from a Phoenix web app, and then save them off to our database.

The most basic way to create a form is to use a map to represent the fields you want to capture. If we wanted to create a map-backed form for our order schema, we'd start with a map that looks like this:

{% highlight elixir %}
order_map = %{"id" => "", "qty" => ""}
{% endhighlight %}

A map is just a key-value collection. Now, we'll pass this into `to_form/1` to get a form struct. From experience, `to_form/1` won't care about the values of the map, just the keys, so they can just be blank, so we can pass this map into `to_form/1` and it will give us a form struct. Let's put this together in a LiveView:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    order_map = %{"id" => "", "qty" => ""}
    form = to_form(order_map)

    {:ok, assign(socket, form: form)}
  end
end
{% endhighlight %}

Our `mount/3` is called whenever someone loads the page for the first time, or on refresh. So whenever users come to our page we'll generate the form, and with our form struct in the socket, we can render the form in our HEEx:

{% highlight elixir %}
<!-- order_live.html.heex -->
<.form
  for={@form}
  phx-change="form_updated"
>
  <.input field={@form[:field]} />
</.form>
{% endhighlight %}

HEEx is just Elixir's version of html templates. If you've used React, this is effectively JSX for Elixir. You've got an HTML template you can embed code in, and you can call components you can pass properties into, except elixir calls these properties "assigns", because they get 'assigned' to your websocket connection with the LiveView. Your browser can then piece together the HTML and the assigns from the websocket to render the page. Phoenix will automatically pair a template with a LiveView if they have the same name, so `order_live.html.heex` is the template for the `order_live.ex` LiveView.

So we've manually defined what fields our form has, created a form struct from the map, and rendered the form in our LiveView. But there are a few problems with this approach. Our form data (`form_params`) is going to come back as a map where the keys are the same as our `order_map` but the values are the strings the user entered in the form. Because of this, we have to manually handle all of the type validation and any casting ourselves. Whenever our form changes, an event is fired named `form_updated` that we'll need to write a handler for:

{% highlight elixir %}
# order_live.ex
defmodule OrderLive do
  use Phoenix.LiveView

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
      # Insert into db - `Repo` comes from Ecto
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

And this is the biggest downside to using a map-backed form - you're left to handle all of the type validation yourself since all your fields come back as strings. Imagine if this form ever gets more fields, this function is just going to keep getting bigger and bigger. There's also a more subtle downside to using map-backed forms - we need to leak the internal structure of our database schema to the frontend:

{% highlight elixir %}
order_map = %{"id" => "", "qty" => ""}
{% endhighlight %}

This is a revealing line of code that to me signals we're headed down the wrong path. We're duplicating the knowledge of our database schema in our frontend code. If our schema ever changes, our form will break instantly and we'll need to update code in a lot of places to accommodate that change. A small change to the schema might mean a lot more work for us.

So how could we make map-backed forms better? There might be a way to just pull out the fields from the struct and use that to create the map we back the form with. We might even be able to use `Map.from_struct/1` to make things easy. But this doesn't completely solve the issue of leaking the schema - we still need to know the types to cast to before we can insert into the database. What if I could define my form's types and validation rules in one place, and have the form handle the rest for me? Ecto has some more tricks up it's sleeve that can help us out here, and that's where our changeset-backed form comes in.

### Changeset-backed forms

#### What's a Changeset?

I've been sidestepping around this, but let's talk about Ecto changesets. So remember when I mentioned Ecto is powerful? Here's where we get to see some of that. We created an order earlier that looks like this:

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

And our problem is that we want to insert new orders into our database by letting users submit their order via a form. As I've said before problem with that is we'll get back all of our data as strings, and we'll need to convert them to the correct types before we can insert them into the database. So what if we could check the set of changes against the types we define in the schema *before* we try to insert them all into a struct?

This is exactly what a changeset is for - a changeset takes in the struct you want to change and the list of changes you want to apply to it. It then compares the changes to the validation rules you've set up, and if there are any errors it picks them up. It can even try to cast types for you. When you try to `apply_action/2` to the changeset, it will either return the struct with the changes applied, or an error. So now our form becomes a lot more powerful - we can automatically cast and validate our form inputs, and then handle the changeset errors if any. You create changesets by writing changeset functions in your schema module. Here's what a super simple validation changeset would look like in our `Order` module:

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

All this changeset does is cast `attrs` to the types defined by the schema. If anything doesn't match or can't be cast, the changeset will error on that field. And that's it!

A cool feature of Phoenix forms is that you can convert a changeset directly to a form - even if that changeset has errors! This is actually great, because the form component can use the errors to show the user what they did wrong, directly inline with the field that caused the error. Changesets can be created directly from the schema, effectively turning them into typed struct validators. The changeset can then automatically cast the inputs into the correct types based off the schema, saving you from having to write a bunch of boilerplate code to do it yourself.

#### Do I Even Need a Database?

If you're not familiar with Ecto you might think that it's specifically meant to handle databases, and if we wanted to make our other forms changeset-backed, we would need to create schemas, changesets, and database tables for the data to live in. But that's not the case!

Ecto supports [embedded schemas](https://hexdocs.pm/ecto/embedded-schemas.html), which means you aren't obligated to use Ecto with a database. Instead, your data can live in memory so you can get all of the validation and casting benefits of the changeset, without needing to insert that data into a database. You can use Ecto as purely a data validation layer. This means all you need is a schema and a changeset to power your forms.

#### Creating a Changeset-Backed Form

So we have our order schema, and we have a changeset function that can create a changeset to validate and cast our form inputs. But how do we use this in our LiveView? Rendering a form is simple, we can take the result of our changeset and pass it to the form component in the HEEx:

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
>
  <.input field={@form[:field]} />
</.form>
{% endhighlight %}

Here, a form is being created on the fly (named `f`) for the changeset we pass into this component. Now this is pretty much exactly what I want to get out of these forms, we pass in a `@changeset` to the form that we can get from our `changeset/2` function, and the form handles the rest. It's very simple. So of course this approach is actually discouraged in the [Phoenix docs](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#form/1-using-the-for-attribute)! The docs say this approach is bad for two reasons:

> 1. LiveView can better optimize your code if you access the form fields using `@form[:field]` rather than through the let-variable `form`
> 
> 2. Ecto changesets are meant to be single use. By never storing the changeset in the assign, you will be less tempted to use it across operations

I don't have much to add to the first point. `:let` seems convenient but I don't know enough about how Phoenix works under the hood to say if something is performant or not. But I can see the second point being a problem - The changeset *result* is meant to be a single use object, and it's not really a good idea to be storing temporary things in your LiveView socket assigns. But, I want to use my changeset *function* to validate my form whenever it updates, because the actual act of validation doesn't change between form updates - that would just be a really weird user experience.

But there has to be a way we can use changesets to power our forms. How can we avoid these pitfalls? 

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
{% endhighlight %}

And our HEEx looks like this:

{% highlight html %}
<!-- order_live.html.heex -->
<.form
  for={@form}
  phx-change="form_updated"
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
</.form>
{% endhighlight %}

But our values don't seem to be showing up in the form. What's going on? Well, when `to_form/1` is called, it copies over data from the changeset to the form struct, but it doesn't seem to pick up the changes! We need to commit the changes first to the changeset before we can convert it to a form. We can do this by applying the `:new` action to the changeset. But we should remember that `apply_action/2` can fail. So our LiveView code becomes this:

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
{% endhighlight %}

And we can tweak the HEEx to show us the `data` of the form. The data is coming from the changeset as a result of the action we applied to it:

{% highlight elixir %}
<!-- order_live.html.heex -->
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

And you might also notice, we don't need to specifically handle errors! We can just turn the changes into a form, and the input component will know how to render the errors out to the user for that field.

## Conclusion

So we've seen how we can use changesets to power our forms in Phoenix. We can use the changeset to validate and cast our form inputs, and then our form can render any errors if present in the changeset. Best of all, we can do this in a way that lets us be performant and keep our frontends uncoupled with our database schemas. I would definitely advise you to jump right for the changeset-backed forms, and avoid the map-backed forms. Changeset-backed forms still work even when you don't have a database or want to use it for your forms.

This is awesome, but we could go further - any other forms we make are going to want that same method to validating and casting inputs. We don't want to copy around the same behavior everywhere, if the behavior for forms needs to be updated or have a feature added, we'd have to change it in a lot of places. Thankfully Elixir has a great way to handle this - macros.

In the next post, I'll talk about how we can extract this behavior into a macro we can use across all of our forms. 
