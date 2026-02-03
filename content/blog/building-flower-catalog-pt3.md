+++
title = "Building a Flower Catalog with Flask, Part 3"
date = 2023-06-09
draft = true
[taxonomies]
tags = ["Flask", "Python", "web", "sql", "tailwindcss", "htmx"]
[extra]
toc=true
+++

# Setup

In this third part about building a simple flower catalog with Flask, I
go over my use of
[Flask-Admin](https://flask-admin.readthedocs.io/en/latest/) to provide
a frontend interface to the database. To include the Flask-Admin
extension, you do the typical thing with your app\'s `__init__.py` file:

```python
# import it
from flask_admin import Admin

# instantiate it
admin = Admin()

# initialize it as part of the app in the app factor function
def create_app(test_config=None):
        app = Flask(__name__, instance_relative_config=True)

    # ...
    # Setup admin app context and ModelViews
        from flower_store.models import Flower
        from flower_store.admin_views import MyAdminIndexView, FlowerView

        admin.init_app(app, index_view=MyAdminIndexView())
        admin.add_view(FlowerView(Flower, db.session))
```

Flask-Admin integrates with existing database models. In
`admin_views.py`, I define versions of both the `AdminIndexeView` class
`ModelView` class. We see that the latter is passed both the `Flower`
model and the database session. At first, I followed the basic example
provided by the documentation. One little addition I made has to do with
how I set up users for my app:

```python
class FlowerView(ModelView):
    # form_base_class = SecureForm()
    create_modal = True
    edit_modal = True

    def is_accessible(self):
        return current_user.is_authenticated and current_user.is_admin

    def inaccessible_callback(self, name):
        return redirect(url_for("login"))


class MyAdminIndexView(AdminIndexView):
    def is_accessible(self):
        return current_user.is_authenticated and current_user.is_admin
```

Since I may want to implement accounts as a feature in the future, I
decided to setup a basic Users table with an `is_admin` attribute that
has a SQL type of `Boolean`. For now, I manage this entirely from the
command line.

## Users and Logging In

Since adding users to a database and using Flask-Login seem to be a
standard pair of elements in Flask tutorials, I won\'t go into much
detail. I mostly repurposed code from [Grinberg\'s
tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-v-user-logins).
At the moment, since I don\'t want user accounts as a feature, none of
this is displayed through the website.

As mentioned before, my `User` class for Flask-SQLAlchemy has the
following attribute:

```python
is_admin = db.Column(db.Boolean, default=False)
```

And I have a function I expose to my flask shell environment that
creates such a user for testing purposes:

```python
# See Part 1 for all imports.

from dotenv import load_dotenv
from flower_store.models import User

def register(app):
    # ...

    @app.cli.command("setup-admin")
    def setup_admin():
        """Creates an admin user account."""

        # Ensure `user` table exists
        if not db.inspect(db.engine).has_table("user"):
            print("User table does not exist. Run `flask db upgrade`")
            return

        load_dotenv()
        admin = os.environ.get("ADMIN")
        admin_pwd = os.environ.get("ADMIN_PWD")

        if not admin_pwd:
            print("There is no admin user password in the local environment.")
            return

        test_admin = User(
            username=admin,
            email="davidrambo@mailfence.com",
            is_admin=True,
        )
        # Check for existing "admin" user and delete if it exists.
        user = User.query.filter_by(username=test_admin.username).first()
        if user:
            db.session.delete(user)

        test_admin.set_password(admin_pwd)
        db.session.add(test_admin)
        db.session.commit()
```

As with the command that populates the flower table, this is run
via `flask setup-admin`.

Both logging in and out have to be done by entering the appropriate
routes (\"/login\" and \"/logout\"). Both routes\' view functions are in
[auth/routes.py](https://github.com/DavidRambo/flower-store/blob/main/src/flower_store/auth/routes.py).
The one for logging in is basically the same as what Grinberg provides,
except it redirects to the admin view if the user logging in is an
admin:

```python
if user.is_admin:
    return redirect("admin")
```

It\'s a nice convenience.

## Flask-Admin\'s Template

Once you\'ve created a user with admin privileges and logged in as that
user, Flask-Admin\'s frontend can be accessed by going to the \"/admin\"
route. To customize it through Flask\'s templating system, Flask-Admin
exposes its templates behind the scenes. Here\'s a minimal change I
made:

```html
{% extends 'admin/master.html' %} {% block body %}
<p>To log out, add "/logout" to the URL.</p>
{% endblock body %}
```

# Uploading images

To upload images, I added an additional field to my `FlowerView` class:

```python
    form_extra_fields = {
        "image_file": ImageUploadField(
            "Image",
            base_path=IMAGE_PATH,
            namegen=image_rename,
            # thumbnail_size=(125, 125, True),
            max_size=(500, 500, True),
        )
    }
```

I adapted this from [an
example](https://github.com/flask-admin/flask-admin/blob/master/examples/forms-files-images/app.py)
provided by Flask-Admin and [the
documentation](https://flask-admin.readthedocs.io/en/latest/api/mod_form_upload/?highlight=PIL#flask_admin.form.upload.ImageUploadField)
for `ImageUploadField`. Initially, I tried to use Flask-Admin\'s
`form_overrides` variable, which didn\'t work. My thinking was that the
form, by default, is a text input box, since `Flower.image_file` takes a
string. This code snippet I arrived at works perfectly. I\'ll break it
down:

- \"image_file\" names the field in the `Flower` model to be
  manipulated.

- `ImageUploadField` levies `Pillow` (a fork of `PIL`, the Python
  Image Library) to perform image manipulation.

- \"Image\" names the field in the Admin view.

- base_path is where you want the file to be saved. `IMAGE_PATH` is
  defined at the module level to get the directory path:

  ```python
  IMAGE_PATH = os.path.join(os.path.dirname(__file__), "static/flower_imgs")
  ```

- namegen defines how the file name is to be generated. I wrote a
  little function called `image_rename` in
  [utils.py](https://github.com/DavidRambo/flower-store/blob/main/src/flower_store/utils.py).
  It takes a tip from Corey Schafer\'s tutorial and adds some hex
  characters to the name provided by the user. It also ensures that
  the length is within the limit imposed by the `Flower` model\'s
  `image_file` field (30).

# Automating asset deletion

The last thing I wanted to implement is a SQLAlchemy hook to delete
images associated with flowers in the event of their deletion. Back in
the app factory, I added the following code beneath the admin setup:

```python
    from sqlalchemy.event import listens_for
    from flower_store.admin_views import IMAGE_PATH

    @listens_for(Flower, "after_delete")
    def del_image(mapper, connection, target):
        if target.image_file:
            # Delete image
            try:
                os.remove(os.path.join(IMAGE_PATH, target.image_file))
            except OSError:
                pass
```

Nice!
