---
layout: post
title: "Building a message board with Python, Flask and PostgreSQL"
author: "Joakim Lönnegren"
categories: blog
tags: [Web development, Python, Flask]
image:
  feature: building-rboard/feature.png
  teaser: building-rboard/teaser.png
---

I got the urge to do some web development lately so I started building a message board inspired by [Reddit](https://reddit.com). Here's some documentation of the process and design decisions. The source tree can be found on [Github](https://github.com/joalon/rboard). You can try out the development version at [joalon.se/apps/rboard](https://joalon.se/apps/rboard) (no stability guarantees!).


## Project setup
This is a pretty standard [Flask](https://palletsprojects.com/p/flask/) web application backed by [SQLite](https://www.sqlite.org) in development and [PostgreSQL](https://www.postgresql.org/) in production.

```fish
mkdir rboard
cd rboard
python -m venv venv
. venv/bin/activate.fish
```

I ended up installing flask and a couple of related libraries for handling logins and such:

```fish
pip3 install flask flask-login flask-sqlalchemy flask-migrate flask-wtforms
pip3 freeze > requirements.txt
```

The directory overview looks like:

```fish
rboard/
     app.py
     requirements.txt
     venv/
         ...
     rboard/
         __init__.py
         main.py
         board/
              __init__.py
              routes.py
         user/
              __init__.py
              routes.py
         post/
              __init__.py
              routes.py
         models.py
         templates/
              base.html
              main.html
              board.html
              post.html
              login.html
              register.html
         static/
              ...
```

I'll go through the most important files:

app.py
```python
from rboard import make_app

app = make_app()
```

The previous file basically imports the app package and creates an app. Including the app package runs the following file, __init__.py (some imports and blueprint registering omitted for brevity):

```python
import os

from flask import Flask, Blueprint, render_template
...

db = SQLAlchemy()
migrate = Migrate()
...

def make_app():
    app = Flask(__name__)

    if app.config['ENV'] == 'dev':
        print("Starting in dev")
        app.config.from_pyfile('config/dev.cfg')
    ...
    else:
        print("Expected FLASK_ENV to be either 'prod' or 'dev'")
        exit(1)

    db.init_app(app)
    migrate.init_app(app, db)
    ...

    base_bp = Blueprint("base", __name__)
    app.register_blueprint(base_bp)

    from rboard.main import blueprint as main_bp
    app.register_blueprint(main_bp)

    ...

    return app


@login.user_loader
def load_user(user_id):
    return User.query.get(user_id)

from rboard.models import User

```

The user loader specifies how flask-login should search for a User object in the database. Importing the models at the end is done to avoid circular imports since rboard.models depends on the variable `db` to register the SQLAlchemy ORM.


## User handling
All user handling is handled by flask-login and the corresponding user blueprint under the `rboard/user` directory. I'll start with the model from models.py:

```python
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(40), unique=True)
    passwordHash = db.Column(db.String(100))
    joined_at = db.Column(db.DateTime, default=datetime.utcnow)
    posts = db.relationship("Post", backref="user")
    comments = db.relationship("Comment", backref="user")

    def check_password(self, password):
        return check_password_hash(self.passwordHash, password)

    def __eq__(self, other):
        if isinstance(self, other.__class__):
            return self.id == other.id
        return False
```

This model can be seen in its natural habitat in the login endpoint in `rboard/user/routes.py`:

```python
@blueprint.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        redirect(url_for('index'))

    form = LoginForm(request.form)
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        if user is not None and user.check_password(form.password.data):
            login_user(user)
            return redirect(url_for('base.main'))

    return render_template('login.html', title="Sign in", form=form)

class LoginForm(FlaskForm):
    username = StringField('Username', [validators.Length(min=1, max=40)])
    password = PasswordField('Password', [validators.Length(min=3, max=100)])
```

If the user is not authenticated and hits the /login endpoint it will render a jinja template with a login form. The form will be evaluated on a POST-request and will check the password hash against the one stored in the database using the werkzeug `check_password_hash`.


## Creating a board
A central part in this app will be users creating their own boards with a title and a description, that other users can post to. The only interesting part compared to the user model is that a board will have moderators, a many-to-many relationship between users and boards:

```python
moderators = db.Table( "moderators",
     db.Column("moderator_id", db.Integer, db.ForeignKey("user.id")),
     db.Column("board_id", db.Integer, db.ForeignKey("board.id")),
)

class Board(db.Model):
    ...
    moderators = db.relationship("User", secondary=moderators)
    ...
```

This means each board now contains a list of its moderators in `board.moderators`.


## Bootstrap
The UI looked a bit bland, so after the board functionality I spent some time adding bootstrap to spice things up on the front end:

![Bootstrap comparison](/images/building-rboard/bootstrap-comparison.png)


In the `base.html` template I added the bootstrap dependencies:

```html
<head>
...
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstr    ap/4.4.1/css/bootstrap.min.css" integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKt    u6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh" crossorigin="anonymous">

<body>
...

    <script src="https://code.jquery.com/jquery-3.4.1.slim.min.js" integrit    y="sha384-J6qa4849blE2+poT4WnyKhv5vZF5SrPo0iEjwBvKU7imGFAV0wwj1yYfoRSJoZ+n" cro    ssorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/pop    per.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3U    ksdQRVvoxMfooAo" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/js/boot    strap.min.js" integrity="sha384-wfSDF2E50Y2D1uUdj0O3uMBJnjuUD4Ih7YwaYd1iqfktj0U    od8GCExl3Og8ifwB6" crossorigin="anonymous"></script>
</body>
```

I took a look at some of [the Bootstrap examples](https://getbootstrap.com/docs/4.5/examples/) to find some designs I liked. I ended up 'stealing' from `Sign-in`, `navbar static` and `offcanvas`.


## Posting and Commenting
After the overhaul it was time to add some features. Posting and commenting was the trickiest part of the application. Since I decided I wanted both comments to posts as well as comments to other comments I ended up making a recursive query in SQLAlchemy as well as  recursive templating going on the front end.

The posts and comments models ended up looking like:

```python
class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(40))
    text = db.Column(db.String(140))
    posted_at = db.Column(db.DateTime, default=datetime.utcnow)
    author_id = db.Column(db.Integer, db.ForeignKey("user.id"))
    board_id = db.Column(db.Integer, db.ForeignKey("board.id"))

    comments = db.relationship(
        "Comment", backref="post"
    )

class Comment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    text = db.Column(db.String(140))
    posted_at = db.Column(db.DateTime, default=datetime.utcnow)
    post_id = db.Column(db.Integer, db.ForeignKey("post.id"))
    author_id = db.Column(db.Integer, db.ForeignKey("user.id"))
    parent_comment_id = db.Column(db.Integer, db.ForeignKey("comment.id"))

    parent = db.relationship("Comment", backref="comments", remote_side=[id])
```

And a post with comments gets rendered with a pretty sweet recursive macro:

```html
{% raw %}
{% macro print_comments(post) %}
    <ul style="list-style-type:none">
    {% for comment in post.comments recursive %}
        <li>
            <span class="d-block">
            {{ comment.text }}
            </span>
            <div class="media text-muted pt-1">
                Posted by {{ comment.user.username }} at {{ comment.posted_at.s    trftime("%Y-%m-%d %H:%M") }}<pre> - </pre><a href="{{ url_for('post.post_index'    , board_name=board_name, post_id=post.id, reply_to=comment.id) }}">reply</a>
            </div>
            {% if current_user.is_authenticated and reply_form and comment_id =    = comment.id %}
                <form method=post>
                    {{ reply_form.csrf_token }}
                    {{ reply_form.body }}
                    <input type=submit value=Comment>
                </form>
            {% endif %}
            <br>

            {% if comment.comments %}
                <ul style="list-style-type:none">
                    {{ loop(comment.comments) }}
                </ul>
            {% endif %}
        </li>
    {% endfor %}
    </ul>
{% endmacro %}
{% endraw %}
```


This macro gets called in the template where the comments should be rendered. It creates an `<ul>` (unordered list) and populates it with an `<li>` (list item) for each comment. If the comment has comments (`if comment.comments`) it does the recursive step `loop`, which calls itself with the argument. Credit to [this](https://stackoverflow.com/questions/42684484/jinja2-recursion-over-python-dictionary-and-sets/42726002) post on Stackoverflow about writing recursive jinja.

Here's another resource for making self-referential tables: [docs.sqlalchemy.org](https://docs.sqlalchemy.org/en/13/orm/self_referential.html).


## Deployment
When starting this project I wanted to try out hosting it on kubernetes with [Minikube](https://minikube.sigs.k8s.io/docs/). I started by building a Docker image:

```Dockerfile
FROM centos:8

RUN yum install -y python3 python3-pip

RUN useradd -d /opt/rboard --shell /bin/bash rboard
USER rboard
WORKDIR /opt/rboard

COPY --chown=rboard rboard ./rboard
COPY --chown=rboard requirements.txt app.py entrypoint.sh /opt/rboard/
RUN pip3 install --user -r requirements.txt

ENV FLASK_APP rboard
ENV FLASK_ENV dev
ENV PATH "/opt/rboard/.local/bin:${PATH}"

EXPOSE 5000

ENTRYPOINT ["./entrypoint.sh"]
```

Where entrypoint.sh is a simple bash script:

```bash
#!/bin/sh

if [ -z $FLASK_ENV ]; then
    echo "No FLASK_ENV variable set"
    exit 1
elif [[ "$FLASK_ENV" == "dev" ]]; then
    flask db init
    flask db migrate
    flask db upgrade
elif [[ "$FLASK_ENV" == "prod" ]]; then
    flask db upgrade
else
    echo "Expected FLASK_ENV to be one of \"prod\" or \"dev\""
    exit 1
fi

flask run --host=0.0.0.0
```

To deploy it to minikube on my laptop I had do enable the registry addon, which deploys a registry on minikube which I could push container images to. To actually push the images I had to setup an ssh tunnel for port 5000 (the default registry port), before building, tagging and pushing the image.

```fish
minikube start --vm-driver=kvm2

minikube addon enable registry

ssh -L 5000:(minikube ip):5000 -N -f localhost
     # where:
     # -L local tunnel
     # -f run in background
     # -N don't start a shell on remote

# Build or tag image
docker build -t localhost:5000/rboard

# Push to remote registry
docker push localhost:5000/rboard

# Pull the image to the minikube vm
minikube ssh -c "docker pull localhost:5000/rboard"

```

When Minikube can pull the image successfully I could deploy it using a yaml file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: rboard-deployment
spec:
    selector:
         matchLabels:
            app: rboard
    replicas: 1
    template:
         metadata:
            labels:
                app: rboard
         spec:
                containers:
                    - name: rboard
                       image: localhost:5000/rboard
                       ports:
                            - containerPort: 5000
```

The following commands then start a development version:

```
kubectl apply -f dev-deployment.yaml
kubectl expose deployment rboard-deployment --type=NodePort
kubectl get service rboard-deployment
```

![Finished product](/images/building-rboard/finished.png)

## Summary
I had a generally good experience working with the Flask framework and PostgreSQL. I struggled some with setting up the app in kubernetes mainly with how to get the docker image to the minikube registry but otherwise it was a pretty smooth experience.

