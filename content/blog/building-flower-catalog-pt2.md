+++
title = "Building a Flower Catalog with Flask, Part 2"
date = 2023-06-08
draft = false
[taxonomies]
tags = ["Flask", "Python", "web", "sql", "tailwindcss", "htmx"]
[extra]
toc=true
+++

This is part 2 of a series of posts about a flower catalog I am
buillding with Flask. In [Part 1](@/blog/building-flower-catalog-pt1.md), I discussed
the routes and templates behind the catalog of flowers in the database.
In this part, I go into how I implemented search in two different ways:
first with SQLAlchemy queries, and second with Elasticsearch. Both
approaches rely on HTMX to handle the frontend. I begin with the view
functions, including the SQL query implementation of search, then show
how HTMX serves as a go-between for an input text field and displaying
search results, and finally I show how I tweaked this to use
Elasticsearch.

# Search Routes

The search implementation uses two routes: \"/search\" and
\"/search~results~\". Both of these are in `catalog.py`, so the
`@bp.route` decorator refers to that blueprint.

```python
@bp.route("/search", methods=["GET"])
def search():
    return render_template("search.html", title="Search")


@bp.route("/search_results", methods=["POST"])
def search_results():
    search_term: str = request.form.get("search")

    if not len(search_term):
        return render_template("results.html", flower_ids=None)

    page = request.args.get("page", 1, type=int)

    query = Flower.query.filter(Flower.name.ilike("%" + search_term + "%"))

    flowers_sorted = query.order_by(Flower.name).paginate(
        page=page, per_page=current_app.config["PER_PAGE"], error_out=False
    )

    return render_template("results.html", flowers=flowers_sorted.items)
```

The first renders the `search.html` template, and the second handles the
actual search query. As we will see when we look at the templates, the
`search.html` template contains a text input form whose behavior is
powered by HTMX. That input form sends a POST request to
`/search_results`, which handles the query and renders it through the
`results.html` template.

The first line in the `search_results()` view function gets whatever the
user has entered into the input field with the name of \"search\". Here
we come across the first major difference between this HTMX-powered
approach compared with using Flask-WTF\'s FlaskForm as a base class to
create a form for searching ([as Miguel Grinberg does in his
tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvi-full-text-search)).
With FlaskForm, one can use its `validate()` method to check whether
anything was submitted. In my implementation, I simply check the length
of the string received from the form. If it\'s zero, then the results
page gets rendered blank. If it\'s non-zero, then the view function
proceeds.

The SQLAlchemy version of the search query proceeds similarly in
structure to the catalog view function. After all, both return a
`render_template` call to `results.html`. The difference is in the
query. Since we don\'t want all Flower table entires, we use
`Flower.query.filter()`, passing it an argument that compares the name
field in the Flower table\'s rows to the `search_term`, optionally
preceded and/or succeeded by arbitrary text. Using `Flower.name.ilike`
rather than `.like` makes it case insensitive.

The last tweak I make to the query results is to arrange them in
alphabetical order by name (this is also done in the `catalog` view
function).

# HTMX Search Input

Check out the template for this:

```html
{% extends "base.html" %} {% block header %} Search {% endblock header %} {%
block content %}
<div class="text-center my-4 px-10 md:mx-30">
  <input
    type="text"
    name="search"
    hx-post="/search_results"
    hx-trigger="keyup changed delay:250ms"
    hx-indicator=".htmx-indicator"
    hx-target="#flower-results"
    placeholder="Search..."
  />
</div>
<div id="flower-results">{% include "results.html" %}</div>
{% endblock content %}
```

It has two parts: the input and the output. The `input` tag is framed by
a `div` that has some TailwindCSS: center the content, add some vertical
margin and horizontal padding, and add horizontal margin on larger
screens. The `input` field has a few typical HTML attributes along with
four HTMX attributes:

- `hx-post`: This sets the route to take when performing a POST
  request. It\'s our search~results~ view function.
- `hx-trigger`: This sets the event that triggers that `hx-post`
  request. It looks for a change (`changed`) in keyboard input
  (`keyup`), and uses a 250ms timer (`delay`) to determine when a
  change has stopped.
- `hx-target`: The target HTML element labeled by `id` attribute. The
  output is identified by `hx-target` as having an
  `id="flower-results"`.

Whenever the text in the input field has changed, and no further changes
have ocurred for a quarter of a second, HTMX sends a POST request to
`/search_results`. This request calls our view function defined to
handle that route, which retrieves the text in the input field with a
`name="search"`.

Another HTMX attribute I would like to use is `hx-indicator`, which can
be used to trigger CSS transitions. I haven\'t been able to get it to
work on my `flower_results` `div` tag. What I\'d like to do is have the
opacity of the content in the `results.html` template ease in.

# Improving Search

## A Problem Case

I think that\'s a decent solution to such a small-scale search feature.
However, aside from letter case, queries need to be exact. This is a
major annoyance for flowers whose names reference their origin. For
example, let\'s say you wanted to search for \"KA\'s Cloud\". Typing in
\"ka\" would net you all flowers with \"ka\" anywhere in their name.
Typing \"ka cloud\" would lose all results as soon as that space is
entered in lieu of an apostrophe.

## Python Alternatives

Since this database will likely remain small, and all I want to search
for the moment are the names of flowers, I could have gone a pure Python
route. For instance, I could have added an attribute to my Flask app
that holds a mapping of all the flower names and their IDs, updating it
when the database changes. This data structure could then be analyzed
using regular expressions or a fuzzy search library. Heck, I could feed
the list of names to `fzf` using the os module\'s `os.popen()` function.

## Elasticsearch

Instead, I opted to use Elasticsearch, mostly because it\'s a good tool
to know how to use for larger scale full-text search. Grinberg shows how
to integrate it into his tutorial Flask project. There are a few changes
since he wrote that (e.g. \"document\" replacing \"body\" and Homebrew
no longer serving Elasticsearch for those on MacOS). And mostly I
followed Grinberg\'s tutorial. Getting it to work on my Linux system
required a bit of additional legwork because of the certification file.
Grinberg shows how to set up a basic Elasticsearch service over http,
but it no longer works with http, requiring a secure https server
instead. It ended up being straightfoward thanks to the `elasticsearch`
python package and `py-dotenv`. Add the path to the `.crt` file to
`.env` and retrieve it as a flask app config attribute.

I made two significant changes to how Grinberg implements Elasticsearch:
fuzzy search instead of multi-match, and HTMX instead of a custom
`SearchForm` class based on Flask-WTF\'s `FlaskForm` class.

Here\'s the modified query function:

```python
def query_index(index, query):
    if not current_app.elasticsearch:
        return
    search = current_app.elasticsearch.search(
        index=index,
        body={"query": {"match": {"name": {"query": query, "fuzziness": "AUTO"}}}},
    )
    ids = [int(hit["_id"]) for hit in search["hits"]["hits"]]
    return ids, search["hits"]["total"]["value"]
```

The `search` method\'s `body` kwarg uses \"match\" instead of
\"multi-match\" in order to make use of fuzzy matching. The difference
between these two query types is that `match` can only query a single
field, whereas `multi-match` can query many fields. Without that change,
its search results were stricter than using SQL\'s `ilike`.

I used Grinberg\'s `SearchableMixin` class verbatim and had my `Flower`
SQLAlchemy model class inherit it. The above `query_index` function is
wrapped by that class\'s own `search` method. This is what gets called
by the `catalog.search_results` view function now:

```python
    query, _ = Flower.search(search_term)
```

The above line replaces the SQLAlchemy query with the `ilike` method.
The thrown away return value is the number of results.

# What\'s next

That\'s it for search. I would like to try implementing Opensearch at
some point. For now, this works exactly as I wanted.

[Next](@/blog/building-flower-catalog-pt3.md), I discuss how Flask-Admin provides a ready-made solution for
frontend database manipulation.
