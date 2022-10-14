# Deploying a Flask API to Render

## Learning Goals

- Set up your local environment for deploying with Render.
- Deploy a basic Flask application to Render.

***

## Key Vocab

- **Deployment**: the processes that make an application available for its
  intended use. For web applications, this means moving the application to a
  platform that supports requests from the internet.
- **Developer Operations (DevOps)**: the practices and tools that improve a
  team's ability to develop and deploy applications quickly.
- **PostgreSQL**: an open-source relational database system that provides more
  SQL functionality than SQLite. Unlike SQLite, its data is stored on a server
  rather than in files.
- **Platform as a Service (PaaS)**: a development and deployment platform that
  exists on a wide range of servers with different functionality. PaaS solutions
  reduce maintenance time for a software development team, but can increase
  cost. Some PaaS solutions, such as Render, provide free tiers for small
  applications.

***

## Introduction

In this lesson, we'll be deploying a basic, standalone Flask API application to
Render. We'll give instructions to generate the application from scratch and
talk through the steps to get the code running on a Render server.

In coming lessons, we'll learn how to add more complexity to the application
with a React frontend. Since the setup for a Flask-React application is a bit
trickier, it'll be beneficial to see the setup for Flask alone first. Let's get
started!

***

## Environment Setup

To make sure you're able to deploy your application, you'll need to do the
following:

### Sign Up for a Render Account

You can sign up at for a free account at
[https://dashboard.render.com/][render dashboard]. We recommend signing up using
your GitHub account- this will streamline the process of connecting your
applications to Render later on.

### Install PostgreSQL

Render requires that you use PostgreSQL for your database instead of SQLite.
PostgreSQL (or just Postgres for short) is an advanced database management
system with more features than SQLite. If you don't already have it installed,
you'll need to set it up.

#### PostgreSQL Installation for WSL

To install Postgres for WSL, run the following commands from your Ubuntu
terminal:

```console
$ sudo apt update
$ sudo apt install postgresql postgresql-contrib libpq-dev
```

Then confirm that Postgres was installed successfully:

```console
$ psql --version
```

Run this command to start the Postgres service:

```console
$ sudo service postgresql start
```

Finally, you'll also need to create a database user so that you are able to
connect to the database from Flask. First, check what your operating system
username is:

```console
$ whoami
```

If your username is "ian", for example, you'd need to create a Postgres user
with that same name. To do so, run this command to open the Postgres CLI:

```console
$ sudo -u postgres -i
```

From the Postgres CLI, run this command (replacing "ian" with your username):

```console
$ createuser -sr ian
```

Then enter `control + d` or type `logout` to exit.

[This guide][postgresql wsl] has more info on setting up Postgres on WSL if you
get stuck.

#### Postgresql Installation for OSX

To install Postgres for OSX, you can use Homebrew:

```console
$ brew install postgresql
```

Once Postgres has been installed, run this command to start the Postgres
service:

```console
$ brew services start postgresql
```

Phew! With that out of the way, let's get started on building our Flask
application and deploying it to Render.

***

## Creating a Flask App to Deploy

We'll be following the steps in Render's
["Deploy a Flask App"][render flask] guide, so if you get
stuck and are looking for more assistance, check that guide first.

The first thing we'll need to do is create our new Flask application. Make
sure you're in a non-lab directory, then run:

```console
$ mkdir bird-app && cd $_
$ pipenv install Flask gunicorn psycopg2-binary Flask-SQLAlchemy Flask-Migrate SQLAlchemy-Serializer Flask-RESTful
```

This will set create an application directory and install some valuable
libraries for building a RESTful API. Let's also run the following command to
generate a `requirements.txt` file:

```console
$ pipenv requirements > requirements.txt
```

`requirements.txt` is very similar to a Pipfile- the primary difference here is
that instead of supporting a local virtual environment through pipenv, it
supports the creation of an application environment on a PaaS platform. (There
are some different tools that use this file as well.)

### Creating a PostgreSQL Database on Render

Using SQLite, our database was generated in a file in our application directory.
With PostgreSQL, the database is stored elsewhere- typically on a server
dedicated to databases. Ours will be stored on a server at Render.

From your Render dashboard, click the "New+" button and select PostgreSQL:

![dropdown menu containing static site, web service, private service, background
worker, cron job, postgresql, redis, and blueprint. postgresql is selected.](
https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-render-postgres.png
)

Next, configure your database with a name, database name, user, and timezone.
These can be whichever values you feel are best.

![form with fields name, database, user, region, postgresql version, and datadog
api key](
https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-render-postgres-config.png
)

Finally, create your database. It will expire after 90 days; you can always make
a new one, but there are paid options available as well. For now, select the
free tier:

![payment options for render databases. the free tier is selected. there is a
create database button at the bottom that will allow users to finish creating
their database](
https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-render-postgres-payment.png
)

From here, scroll down in the new database configuration page and copy the
"External Database URL". Modify the protocol to say `postgresql` instead of
`postgres` (SQLAlchemy is picky) and run the following command in your project
root directory:

```console
$ export DATABASE_URI=<External Database URL goes here>
```

Now we're ready to start building our app.

### Building the Demo App

Let's set up our app, models, and migrations to get things started:

```py
# app.py

import os

from flask import Flask, jsonify, make_response
from flask_migrate import Migrate
from flask_restful import Api, Resource

from models import db, Bird

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URI')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.json.compact = False

migrate = Migrate(app, db)
db.init_app(app)

api = Api(app)

class Birds(Resource):

    def get(self):
        birds = [bird.to_dict() for bird in Bird.query.all()]
        return make_response(jsonify(birds), 200)

api.add_resource(Birds, '/birds')

```

```py
# models.py

from flask_sqlalchemy import SQLAlchemy
from sqlalchemy_serializer import SerializerMixin

db = SQLAlchemy()

class Bird(db.Model, SerializerMixin):
    __tablename__ = 'birds'

    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String)
    species = db.Column(db.String)

    def __repr__(self):
        return f'<Bird {self.name} | Species: {self.species}>'

```

Add this data to the `seed.py` file:

```py
# seed.py

from app import app
from models import db, Bird

db.init_app(app)

with app.app_context():

    print('Deleting existing birds...')
    Bird.query.delete()

    print('Creating bird objects...')
    chickadee = Bird(name='Black-Capped Chickadee', species='Poecile Atricapillus')
    grackle = Bird(name='Grackle', species='Quiscalus Quiscula')
    starling = Bird(name='Common Starling', species='Sturnus Vulgaris')
    dove = Bird(name='Mourning Dove', species='Zenaida Macroura')

    print('Adding bird objects to transaction...')
    db.session.add_all([chickadee, grackle, starling, dove])

    print('Committing transaction...')
    db.session.commit()

    print('Complete.')

```

Then run these commands to generate the database and run the migrations and seed
file:

```console
$ flask db init
# => ...
$ flask db revision --autogenerate -m'create table birds'
# => ...
$ flask db upgrade
# => ...
$ python seed.py
# => Deleting existing birds...
# => Creating bird objects...
# => Adding bird objects to transaction...
# => Committing transaction...
# => Complete.
```

> `flask db upgrade` creates a new PostgreSQL database to be associated with your
> application based on the configuration on Render and in `models.py`.
> Unlike with SQLite, the actual database file isn't created in the
> application's folder;
> it lives on Render's servers and will not show up in your directory structure
> at all.

To make sure the app works locally before deploying, run `gunicorn app:app` and visit
[http://localhost:8000/birds](http://localhost:8000/birds).

> **NOTE: gunicorn runs on port 8000 by default. Because this does not conflict
> with any system ports on MacOS, Windows, or Linux, we won't change it here.

***

## Deploying

Now that we've got some working code, it's time to get that code to run on a
Render server! The process of uploading our code to Render is managed by
Git. This makes it easy to deploy new versions using a tool most developers,
including yourself, are already familiar with.

Make a commit to save your changes, then push them to a remote repo:

```console
$ git add .
$ git commit -m 'Initial commit' 
```

Next, you'll need to create repository in GitHub:

![github create repo form. repo is set to public](
https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-github-repo.png
)

...and push your work to the new repo:

```console
$ git remote add <remote_name> <remote_repo_url>
$ git push -u <remote_name> <local_branch_name>
```

***

## Configuring the Environment and Deploying

Head back to [the Render dashboard][render dashboard] and click
"New+" to make a new Web Service. If you connected to GitHub when you signed up,
you should see a list of all your repos! Find your bird API and click "Connect".

Give your application a name and configure it as seen below:

![configuration screen for a web service in render. the root directory is left
blank. environment is Python 3. Region is Ohio (US East). Branch is main. Build
command is "pip install -r requirements.txt". start command is "gunicorn
app:app"](
https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-render-flask.png
)

Click "Create Web Service" to create your web service. (It will fail, though.)

Before our first successful deployment, we need to click on the "Environment"
tab and enter two environment variables:

`PYTHON_VERSION` is `3.8.13`.

`DATABASE_URI` is the External Database URL from your PostgreSQL database. Don't
forget to change the protocol to `postgresql`!

Render will automatically deploy your _working_ application now. Your URL is
located under your app name at the top of the screen. Click on the link,
navigate to `/birds`, and view your work in all its glory!

```json
[
  {
    "id": 1,
    "name": "Black-Capped Chickadee",
    "species": "Poecile Atricapillus"
  },
  {
    "id": 2,
    "name": "Grackle",
    "species": "Quiscalus Quiscula"
  },
  {
    "id": 3,
    "name": "Common Starling",
    "species": "Sturnus Vulgaris"
  },
  {
    "id": 4,
    "name": "Mourning Dove",
    "species": "Zenaida Macroura"
  }
]
```

***

## Adding New Features

Since Render integrates the deploying process with Git, it's straightforward
to add new features to your code and deploy them. Let's start by adding a new
route in `app.py`:

```py
class BirdByID(Resource):
    def get(self, id):
        bird = Bird.query.filter_by(id=id).first().to_dict()
        return make_response(jsonify(bird), 200)

api.add_resource(BirdByID, '/birds/<int:id>')

```

Test your code locally by running `gunicorn app:app` and visiting
[https://localhost:8000/birds/1](https://localhost:8000/birds/1).

After adding this code, make a commit:

```console
$ git add app.py
$ git commit -m 'Added get by ID route'
```

Then, to deploy the changes, push the new code up to GitHub:

```console
$ git push
```

After pushing new code, Render will run through the build process again and
deploy your changes. You don't have to run your migrations again since the
database already exists on the server. You would have to run the migrations if
you created a new migration file.

***

## Conclusion

Congrats on deploying your first Flask app to the world wide web! Understanding
the deployment process and what it takes to run your application on another
computer is an important step toward becoming a full-stack developer. Like
anything new, this process can be daunting the first time you try it, but with
practice and exposure, you'll build confidence over time.

In the next lesson, we'll work on deploying a more complex application with a
Flask API backend and a React frontend, and talk through some of the challenges
of running these two applications together.

***

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. When creating a new Rails app from the terminal, what additional flag do you
   need to use to be able to deploy it on Heroku?
2. What familiar process is used for deploying code to Heroku? How does the
   process differ when you're deploying to Heroku vs. developing code locally?

## Resources

- [Deploy a Flask App - Render][render flask]

[render dashboard]: https://dashboard.render.com/
[postgresql wsl]: https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database#install-postgresql
[render flask]: https://render.com/docs/deploy-flask
