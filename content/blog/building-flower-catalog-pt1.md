+++
title = "Building a Flower Catalog with Flask, Part 1"
date = 2023-06-07T10:36:23-07:00
draft = false
[taxonomies]
tags = ["Flask", "Python", "web", "sql", "tailwindcss", "htmx"]
[extra]
toc=true
+++

# Overview

My wife and I grow a lot of flowers: dahlias, ranunculuses, poppies,
snapdragons, straw flowers, and more. Okay, I mainly move the dirt. My
wife grows the flowers. To practice some backend web development, I set
out to make a website that serves primarily as a catalog of those
flowers and secondarily as a store front. The market for dahlia tubers
and cuttings is intense, with the boutique cuttings going for \$35 a
piece and selling out in seconds.

But before getting to the storefront aspect, I wanted to implement a few
things:

1.  **Search**. Most of the small dahlia shops either lack a search
    function or have it but it\'s broken. So I wanted to try out a
    couple of implementations.
2.  **Admin**. A way to manage the database through the website.
    Unsurprisingly, there\'s already a package that extends Flask for
    this: [Flask-Admin](https://flask-admin.readthedocs.io/en/latest/).
3.  **HTMX**. Since I\'m mainly invested in backend work, I\'d like to
    keep the frontend scripting to a minimum. This is where
    [HTMX](https://htmx.org/) comes in: it provides attributes for use
    within HTML element tags. Cool!
4.  **TailwindCSS**. Despite my focus on the backend, I do still enjoy
    tinkering with CSS and frontend design. I wanted to try out
    [TailwindCSS](https://tailwindcss.com/), which uses utility classes
    to apply CSS in-line with HTML elements.

In this series, I will go over parts of the build process. There are
excellent tutorials for Flask, like [Corey
Schafer\'s](https://youtube.com/playlist?list=PL-osiE80TeTs4UjLw5MM6OjgkjFeUxCYH)
on YouTube and Miguel Grinberg\'s [Flask
Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world).
So I will forego things like templates, routes, and forms. To see how
all that works together in my case, the [code is available on
GitHub](https://github.com/DavidRambo/flower-store).

In this first part, I explain the \"Catalog\" page, which displays the
flowers.

# Flowers in the Database

My approach to managing the flowers mostly follows [Miguel Grinberg\'s
tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-iv-database).
He makes use of an extension he wrote called Flask-Migrate, which
integrates Alembic with Flask, to manage modifications made to the
underlying database models. This made upgrading my SQLite database
straightfoward, something I did a handful of times as I figured out more
fields to add to the Flower table. I also use the Flask-SQLAlchemy
extension to work with SQLAlchemy\'s ORM within Flask.

With the database setup, I need to populate it with some data for
development. For this, I have a pair of functions defined in `cli.py`.
These are both made available through the application's `run.py` script
at the root directory of the project:

```python
from flower_store import cli, create_app, db
from flower_store.models import Flower, User

app = create_app()
cli.register(app)

@app.shell_context_processor
def make_shell_context():
    """When the `flask shell` command runs, it invokes this function and
    registers the dictionary of items returned. Thus, the database instance
    'db' can be accessed in the shell session as 'db', and likewise for the
    SQLAlchemy models 'Flower' and 'User'.
    """
    return {
        "db": db,
        "Flower": Flower,
        "User": User,
    }
```

For this to work, the `FLASK_APP` environment variable needs to be set to
`run.py`. This is done with a line in `.flaskenv` at the project's root
directory.

Any CLI commands I set up using `click` (one of Flask's dependencies) can
now be run as secondary commands with `flask`.

```python
import os
from random import randint, shuffle

from flask import current_app

from flower_store import db
from flower_store.models import Flower


def register(app):
    @app.cli.command("pop-flowers")
    def populate_flowers():
        """Populates the database with Flowers for the sake of development."""

        flowers = [
            "A-Peeling",
            "Bride To Be",
            "Café au Lait",
            "Cheers",
            "Daddy's Girl",
            "Diva",
            "Fluffles",
            "Foxy Lady",
            "Ice Tea",
            "KA's Bella Luna",
            "KA's Blood Orange",
            "KA's Boho Peach",
            "KA's Cloud",
            "KA's Mocha Jake",
            "KA's Mocha Maya",
            "L'Ancress",
            "Lovebug",
            "Mai Tai",
            "Maki",
            "Marshmallow",
            "Maui",
            "Moonstruck",
            "Ranunculus",
            "Snapdragon",
            "Straw flower",
            "Tootles",
        ]
        shuffle(flowers)

        # Make sure `flower` table exists
        if not db.inspect(db.engine).has_table("flower"):
            print("Flower table does not exist. Run `flask db upgrade`")
            return

        # Clear Flower table
        Flower.query.delete()

        for flower in flowers:
            if flower == "Straw flower":
                db.session.add(
                    Flower(
                        name=flower,
                        stock=randint(0, 10),
                        image_file="strawflower_edb8f2b8dc93.png",
                        price=35.00,
                    )
                )
            elif flower == "Ranunculus":
                db.session.add(
                    Flower(
                        name=flower,
                        stock=randint(0, 10),
                        image_file="ranunculus_fc5fbcc0f.jpg",
                        price=35.00,
                    )
                )
            else:
                db.session.add(
                    Flower(
                        name=flower,
                        stock=randint(0, 10),
                        price=float(str(randint(1, 35)) + "." + str(randint(0, 99))),
                    )
                )

            # Update elasticsearch index.
            if current_app.elasticsearch:
                entry = Flower.query.filter_by(name=flower).first()
                current_app.elasticsearch.index(
                    index="flower", id=entry.id, document={"name": flower}, timeout=30
                )

        db.session.commit()
```

The above can be run at the command line with `flask pop-flowers`.

# Templates

Now, on to the website. The \"Catalog\" portion of the website spans
several files:

- src/flower_store/catalog.py
- src/flower_store/templates/catalog.html
- src/flower_store/templates/results.html
- src/flower_store/templates/\_flower.html

The `catalog.py` module comprises the catalog Flask blueprint and
associated routes. I\'ll get to that in the next section. The three
templates are nested via Jinja2 templates.

## catalog.html

This template serves as a kind of landing page for the actual content.
Here it is:

```html
{% extends "base.html" %} {% block header %} Catalog {% endblock header %}
{%block content %} {% include "results.html" %} {% endblock content %}
```

It provides some text that the base layout styles as a title for the page.
Then it is itself extended by the results template.

## results.html

The results template does two things: it serves up flower cards in a CSS
grid and it displays pagination links in a `nav` section. It will be
reused by the catalog, search, and in-stock routes. The whole template
is framed inside an if statement. That way, if there are no flowers
provided by the Flask app\'s `render_template` function, it will skip
over that HTML and instead display some spacing to buffer against the
website\'s footer.

Here\'s the main portion:

```html
{% if flowers is defined and flowers|length>0 %}
<div class="grid sm:grid-cols-2 md:grid-cols-3 2xl:grid-cols-4 gap-10">
  {% for flower in flowers %} {% include "_flower.html" %} {% endfor %}
</div>
```

We\'ll get to where the `flowers` data comes from after going over the
templates. This snippet displays a grid and populates it with
`_flower.html` templated content generated by a `for` loop. Here\'s our
first use of TailwindCSS\'s utility classes:

```html
<div class="grid sm:grid-cols-2 md:grid-cols-3 2xl:grid-cols-4 gap-10"></div>
```

This sets the `div` tag to display as a grid, and to display a certain
number of columns depending on the screen size. These abbreviated
prefixes show how TailwindCSS handles responsive design: \"sm\" for
small, \"md\" for medium, and so forth.

## \_flower.html

Finally, the individual flower card:

```html
<div class="card">
  <a href="{{ url_for('catalog.flower', flower_id=flower.id) }}">
    <img
      class="w-full h-48 lg:h-72 object-cover"
      src="{{ url_for('static', filename='flower_imgs/' + flower.image_file) }}"
    />
    <div class="pt-1">
      <span class="text-md">{{ flower.name }}</span>
    </div>
  </a>
</div>
```

Taking this line-by-line, the first specification we see is a CSS class
called `card`. This is defined in `static/src/main.css` as a set of
extracted Tailwind classes:

```css
.card {
  @apply pb-2 md:w-full text-center bg-white border rounded-md overflow-hidden hover:shadow-lg hover:scale-105 hover:bg-opacity-50 transform ease-out duration-300;
}
```

Typically, [Tailwind\'s \@apply
directive](https://tailwindcss.com/docs/reusing-styles#extracting-classes-with-apply)
is used to render a set of its utility classes more easily repeatable. I
find this to be a funny loop back to regular old CSS. Of course, it\'s
still using Tailwind\'s utility classes, but it does so entirely through
reference to an external stylesheet.

Next, the anchor tag\'s `href` attribute invokes Flask\'s `url_for`
function:

```html
{{ url_for('catalog.flower', flower_id=flower.id) }}
```

Flask\'s `url_for()` function generates a path to the route associated
with \"catalog.flower\". (More on this next.) The second parameter is
passed as the `flower_id` keyword parameter. Remember that the
`_flower.html` template is being called within a `for` loop in
`results.html`. That\'s where the `flower` reference comes from. As
we\'ll see, It references an entry in the database as defined by the
Flask-SQLAlchemy `Flower` class.

It gets used two more times in this template: once to generate a path to
the image to display, and again to display the flower\'s name. Those
attributes: `flower.id`, `flower.image_file`, and `flower.name` are all
attributes in the ORM class that correspond to columns in the Flower
table.

And that\'s a perfect place to switch over to the catalog blueprint.

# Catalog Blueprint

Let\'s take this in reverse, starting where we left off.

## The flower route

At the top of `catalog.py`, a Flask Blueprint is instantiated:

```python
bp = Blueprint("catalog", __name__)
```

This will be used as part of the route decorators and will be registered
in the application factory (the `create_app` function in
`src/flower_store/__init__.py`) via
`app.register_blueprint(catalog.bp)`.

The `_flower` template\'s `url_for` references to `catalog.flower` are
defined as a route in this blueprint:

```python
@bp.route("/catalog/<flower_id>", methods=["GET"])
def flower(flower_id):
    """Individual flower's page."""
    flower = Flower.query.filter_by(id=flower_id).first()

    image_file = url_for("static", filename="flower_imgs/" + flower.image_file)
    return render_template(
"full_flower.html", title=flower.name, flower=flower, image_file=image_file
    )
```

So, the template\'s `url_for` call establishes a route that in turn
calls this function, decorated with a Flask route. The route has a
variable in angled brackets called `flower_id`, which corresponds to the
keyword parameter of the view function `flower`. When clicking on the
link, the `full_flower.html` template is returned along with some
additional data that the template can make use of.

## The catalog route

Moving up the nested hierarchy of templates, `results.html` provides the
individual flower objects by iterating over some data structure
referenced as `flowers`. Recall that the `catalog.html` template wraps
the `results.html` template, passing along `flowers`. (This separation
is so that `results.html` can be used by other routes, namely those for
search and displaying in-stock flowers.) To see where `flowers` comes
from, let\'s take a look at the `/catalog` route:

```python
@bp.route("/catalog", methods=["GET"])
def catalog():
    """The catalog page shows all flowers in the database regardless of
    inventory.
    """
    page = request.args.get("page", 1, type=int)
    flowers = Flower.query.order_by(Flower.name).paginate(
        page=page, per_page=current_app.config["PER_PAGE"], error_out=False
    )
    next_url = (
        url_for("catalog.catalog", page=flowers.next_num) if flowers.has_next else None
    )
    prev_url = (
        url_for("catalog.catalog", page=flowers.prev_num) if flowers.has_prev else None
    )
    return render_template(
        "catalog.html",
        title="Catalog",
        flowers=flowers.items,
        next_url=next_url,
        prev_url=prev_url,
    )
```

In order to handle paginated results in a dynamic fashion, this view
function interacts with a `flask_sqlalchemy.pagination.QueryPagination`
object, referenced by the name `flowers`. The URLs for those pages, if
they exist, are themselves calls to this same view function---hence,
`url_for("catalog.catalog", page=flowers.next_num)`. [As Miguel Grinberg
explains](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-ix-pagination),
the first line of code defines `page` as either the current page (to be
found in the URL: `.../catalog?page=2`, for example) or a default of 1.
This `page` value therefore handles which page of results from the
`Flower.query` call get passed to the template and thus displayed. The
`render_template` function call in turn passes those flowers along as a
list of `Flower` model objects. (This can be confirmed within the
`flask shell` environment, where `type(flowers.items[0])` returns
`<class 'flower_store.models.Flower'>`.)

# Next up

That was by no means exhaustive, but I hope it provides some helpful
explanation of how the lessons from popular Flask tutorials can be
tweaked into new form. Next, I\'ll be going over [how I implemented search](@/blog/building-flower-catalog-pt2.md).
