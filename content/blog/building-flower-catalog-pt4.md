+++
title = "Building a Flower Catalog with Flask, Part 4"
date = 2023-06-16T11:05:45-07:00
draft = true
[taxonomies]
tags = ["Flask", "Python", "web", "sql", "tailwindcss"]
+++

# Adding a Shopping Cart

The final feature I wanted to implement before deploying is a shopping cart.
Granted, we have no plans to sell anything just yet, so this was more for my own edification.
And I stopped short of providing an actual checkout with payment and shipping forms.
So in this post, I explain adding routes that work with Flask's `session` global variable to track an anonymous user's shopping cart.
It takes into account the available stock and, with the help of Jinja2 templating, integrates into the website's front end.

# The cart blueprint

In `src/flower_store/cart.py`, I created a new Flask Blueprint called `cart` and added routes for the shopping cart page (`"/cart"`), adding to it (`"/cart/add/<flower_id>"`), updating quantities (`"/cart/update/<flower_id>"`), and removing flowers from the cart (`"/cart/remove/<flower_id>"`).
Later, I'll go over the checkout and submit routes.

## The `session` variable

There are multiple ways to implement a cart.
Since I don't need user accounts, I opted to keep it anonymous and cookie-based rather than served up by the database.
Flask provides a global variable called `session`, which is a `dict` assigned to a session served up to a web browser.
Once I wrapped my head around its availability to my web app (much like `g` in templates and `current_app`), it made the implementation easy.
There's no need to declare it in the app factory or anywhere else.
Flask takes care of it behind the scenes.

## `"/cart"`

Accordingly, the first thing that needs to be done when the server receives a GET request for the shopping cart is to see whether it exists yet:

```python
@bp.route("/cart", methods=["GET"])
def cart():
    # First check that the cart has been instantiated.
    if "cart" not in session or not session["cart"]:
        return render_template("cart.html", title="Shopping Cart", cart=None)
```

The if statement does that first, and then, if the `cart` key does exist in `session`, it checks whether it's falsey.
Since `session["cart"]` can only ever be a `dict` (barring any mistaken changes I might make in the future!), I could make this stricter by changing it to `or session["cart"] == {}`.
The looser `not` check works here.
Then, it renders the shopping cart's template, `cart.html` with `None` passed as the `cart` parameter.

Any code beyond that conditional will have at least one item in the cart.
The next block evaluates those items, ensuring they're still in stock and, if they are, then ensuring their quantity in the cart does not exceed the available stock.

```python
    # def cart() ...
    messages = _check_cart()
    if messages:
        for message in messages:
            flash(message)
```

And the helper function:

```python
def _check_cart() -> list[str] | None:
    messages = []
    for flower_id in session["cart"]:
        flower = Flower.query.filter(Flower.id == flower_id).first()
        if flower is None or flower.stock < 1:
            del session["cart"][flower_id]
            messages.append(f"{flower.name} is no longer in stock.")
        elif session["cart"][flower_id] > flower.stock:
            messages.append(f"{flower.name} stock has been reduced to {flower.stock}.")
            session["cart"][flower_id] = flower.stock

    session.modified = True
    return messages if len(messages) > 0 else None
```

In addition to looking up flowers in the database and comparing some integers, I take advantage of Flask's usage of Click and `flash` some messages.
These can be retrieved within the html template and shown to the user.
This provides the user with some feedback about the changes automatically made to their cart.

Note the penultimate line in `_check_cart`: `session.modified = True`.
This alerts Flask to a change made to a mutable data structure in the `session` dict.
Otherwise, changes will not persist between operations.

## Add to cart

Before I get to the rest of the `cart` view function, let's take a look at the `add` view function.
This will help make sense of the former.
`add` comprises three blocks:

1. ensure that the flower is in stock
2. creates the cart if necessary
3. add the flower to the cart or increase its quantity already in the cart

```python
@bp.route("/cart/add/<flower_id>", methods=["POST"])
def add(flower_id):
    """Adds one flower of `flower_id` to the shopping cart."""
    # Ensure that the flower is in stock.
    flower = Flower.query.filter(Flower.id == flower_id).first()
    if flower is None or flower.stock < 1:
        flash("That flower is out of stock.")
        return redirect(url_for("catalog.flower", flower_id=flower_id))

    # Check whether the shopping cart has already been declared in this session.
    if "cart" not in session:
        # Create the shopping cart in the session.
        session["cart"] = defaultdict(int)

    # Add the flower to it.
    quantity = int(request.form.get("quantity"))
    if flower_id not in session["cart"]:
        session["cart"][flower_id] = quantity
    else:
        session["cart"][flower_id] += quantity
        # If the above pushes the cart quantity over the amount in stock, then
        # it will be corrected when going to the cart.

    session.modified = True

    return redirect(url_for("cart.cart"))
```

Technically, the first check should be unnecessary for two reasons.
First, flowers without at least one in stock do not have an "Add to cart" form rendered in their template.
Second, the user cannot manually enter a route to add to the cart because this route only accepts POST requests.

Next, we see that the `session["cart"]` data structure is a dictionary.
Specifically, it's a `defaultdict` from Python's built-in `collections` module.
By passing it `int` at instantiation, this means that when trying to access a non-existent key, the `defaultdict` will create the key and set its initial value to `0`.
This takes care of having to worry about causing a `KeyError` when manipulating the cart.

Last, there's an `input` field in the `full_flower.html` template that takes a number between 1 and the number of that flower in stock.
It is associated with the "Add to cart" button since both are contained within a `form` element.

```html
<form action="{{ url_for('cart.add', flower_id=flower.id) }}" method="post">
  <input
    class="w-12 py-1 pl-2 pr-1 border-gray-300 rounded-lg focus:border-blue-300 mr-4"
    name="quantity"
    type="number"
    value="1"
    min="0"
    max="{{ flower.stock }}"
  />
  <button class="rounded-md bg-pink-200 p-2">Add to cart</button>
</form>
```

This value is retrieved via Flask's `request` object (`request.form.get('quantity')`).
(Note the `input` element's `name` attribute.)

As explained in [Part 1](@/blog/building-flower-catalog-pt1.md), I'm using TailwindCSS to style the website's front end.

## Back to the cart

Now for the rest of the `cart` view function.
This snippet loops over the IDs in the cart dictionary, which are mapped to the quantity added to the cart.

```python
    flowers_in_cart = []
    total: float = 0
    for id in session["cart"]:
        flower = Flower.query.filter(Flower.id == id).first()
        quantity = session["cart"][id]
        flowers_in_cart.append(
            (
                flower.id,
                flower.name,
                quantity,
                "${:.2f}".format(flower.price * quantity),
                flower.stock,
            )
        )
        total += flower.price * quantity

    return render_template(
        "cart.html",
        title="Shopping Cart",
        cart=flowers_in_cart,
        total="${:.2f}".format(total),
    )
```

To the `flowers_in_cart` list, I append a tuple containing all the data necessary for rendering the shopping cart and supporting its features.
This tuple includes the total cost of each flower.
I added a `price` attribute to the `Flower` model, made if of type `Float` with 5 characters, and, for the sample version of the site, set all flowers to have a price of `35.00`.
I do two things with that price: each flower has its total prepared as a string with a dollar sign; but I also keep a running total of the float value, which gets passed to the template as well.

## Remove and Update

The user may also manipulate the items in their cart thanks to two more routes in the `cart` blueprint: one to update the quantity, another to remove.
Both of these are "POST" requests made via forms in the cart.

Each item listed in the cart has a "Remove" button:

```html
<!-- Remove -->
<td class="p-3 text-sm text-gray-800">
  <form action="{{ url_for('cart.remove', flower_id=id) }}" method="post">
    <button class="rounded-md bg-gray-400 p-1">Remove</button>
  </form>
</td>
```

Pressing it calls the remove route's view function with that flower's id.

```python
@bp.route("/cart/remove/<flower_id>", methods=["POST"])
def remove(flower_id):
    """Removes flower with `flower_id` from cart."""
    try:
        del session["cart"][flower_id]
        session.modified = True
        flower_name = Flower.query.filter(Flower.id == flower_id).first().name
        flash(f"{flower_name} removed from cart.")
        return redirect(url_for("cart.cart"))
    except KeyError:
        flash("Invalid item number.")
        return redirect(url_for("cart.cart"))
```

Three things happen:

1. The entry in the `session["cart"]` dict for that flower is deleted.
2. A message is flashed indicating this change.
3. The shopping cart is reloaded.

The try/except block _should be_ unnecessary because only flowers already in the cart will have a "Remove" button, and also because this route only accepts POST requests—that is, entering the route manually in the browser's URL bar as a GET request will be rejected.
However, I chose to keep it just in case something down the line of development were to break that assumption.

The "Quantity" form of the cart template is similar to its counterpart in `full_flower.html` used by the `add` route:

```html
<!-- Quantity -->
<td class="p-3 text-sm text-gray-800">
  <form
    class="flex flex-row"
    action="{{ url_for('cart.update', flower_id=id) }}"
    method="post"
  >
    <input
      class="w-12 py-1 pl-2 pr-1 border-gray-300 rounded-lg focus:border-blue-300 mr-4"
      name="quantity"
      type="number"
      value="{{ quantity }}"
      min="0"
      max="{{ stock }}"
    />
    <button class="p-1 rounded-md bg-green-300 bg-opacity-50">Update</button>
  </form>
</td>
```

And the view function:

```python
@bp.route("/cart/update/<flower_id>", methods=["POST"])
def update(flower_id):
    """Updates the quantity in the shopping cart."""
    new_quantity = int(request.form.get("quantity"))

    if new_quantity == 0:
        # Remove from cart.
        return redirect(url_for("cart.remove", flower_id=flower_id))

    if session["cart"][flower_id] == new_quantity:
        # Quantity is the same.
        return redirect(url_for("cart.cart"))

    # Update to new quantity.
    session["cart"][flower_id] = new_quantity
    session.modified = True
    flash("Cart updated.")
    return redirect(url_for("cart.cart"))

```

If the quantity is updated to `0`, then it redirects to the remove view function.
This is nice because the flashed message is more specific.
If the quantity is unchanged, then it returns to the cart without making any adjustments.
While it would be nice to have that check at the front end, handling it here is easiest, and the performance hit here is trivial.
With those edge cases handled, the value for the correct key in the cart is modified, a generic "Cart updated." message is flashed, and the cart is reloaded.

## Checkout

Finally, I implemented a minimal "checkout" route.
Since we're not prepared to sell anything, I decided to include only a simple take on this feature.
Clicking the checkout button takes the user to a `checkout.html` template that offers to buttons: a submit button and a return-to-cart button.

The submit route adjusts the stock of flowers in the database:

```python
@bp.route("/cart/submit", methods=["POST"])
def submit():
    """Submits the order, clears the cart in the session, and reduces stock."""
    # Go through cart, reducing stock of flowers.
    for id, quantity in session["cart"].items():
        flower = Flower.query.filter(Flower.id == id).first()
        flower.stock -= quantity

    db.session.commit()

    # Clear the shopping cart.
    session["cart"].clear()

    flash("Order submitted!")
    return redirect(url_for("cart.cart"))
```

# Wrapping Up

That's nearly everything worth mentioning for the shopping cart, at least until integrating a payment and shipping solution.
If you look at the full [cart.html template](https://github.com/DavidRambo/flower-store/blob/main/src/flower_store/templates/cart.html), you will see the following structure for the content block:

1. Check for messages flashed from the Flask app.
2. If something other than `None` was passed to the template's `cart` variable, then render a table to display items in the cart. Beneath that table, show the total cost of the shopping cart and provide a button for checking out.
3. Otherwise, print a "Your cart is empty." paragraph.

The shopping cart table is another aspect of the site that puts TailwindCSS to good use.
Tweaking the width, padding, and text alignment values took some time to get it all looking as I wanted.
The most challenging portion was adjusting both the `<td>`level utility classes and those of the `<input>` field for the "Quantity" column.

That's pretty much a wrap on this Flower Catalog/Shop project.
I still aim to deploy it, which will get a write-up of its own.
At that point, I will also add proper error reporting, including emails to alert me to them.
Thanks for reading, and I hope it has been useful.
