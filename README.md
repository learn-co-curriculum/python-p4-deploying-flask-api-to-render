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

### Sign Up for a Heroku Account

You can sign up at for a free account at
[https://signup.heroku.com/devcenter][heroku signup].

### Download the Heroku CLI Application

Download the [Heroku CLI][heroku cli]. This will let you run commands from your
terminal to deploy and interact with your application on Heroku.

**For OSX users**, you can use Homebrew to install the CLI:

```console
$ brew tap heroku/brew && brew install heroku
```

**For WSL users**, run this command in the Ubuntu terminal:

```console
$ curl https://cli-assets.heroku.com/install.sh | sh
```

If you run into issues installing, check out the [Heroku CLI][heroku cli]
downloads page for more options.

After downloading, **log in to Heroku** via the CLI in the terminal:

```console
$ heroku login
```

This will open a browser window to log you into your Heroku account. After
logging in, close the browser window and return to the terminal. You can run
`heroku whoami` in the terminal to verify that you have logged in successfully.

### Install the Latest Ruby Version

Verify which version of Ruby you're running by entering this in the terminal:

```console
$ ruby -v
```

Make sure that the Ruby version you're running is listed in the [supported
runtimes][] by Heroku. At the time of writing, supported versions are 2.6.8,
2.7.4, or 3.0.2. Our recommendation is 2.7.4, but make sure to check the site
for the latest supported versions.

If it's not, you can use `rvm` to install a newer version of Ruby:

```console
$ rvm install 2.7.4 --default
```

You should also install the latest versions of `bundler` and `rails`:

```console
$ gem install bundler
$ gem install rails
```

### Install Postgresql

Heroku requires that you use PostgreSQL for your database instead of SQLite.
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
connect to the database from Rails. First, check what your operating system
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

Phew! With that out of the way, let's get started on building our Rails
application and deploying it to Heroku.

## Creating a Rails App to Deploy

We'll be following the steps in the
[Heroku Rails Deploying Guide][heroku rails deploying guide], so if you get
stuck and are looking for more assistance, check that guide first.

The first thing we'll need to do is create our new Rails application. Make
sure you're in a non-lab directory, then run:

```console
$ rails new bird-app --api --minimal --database=postgresql
```

This will set up our app to run in API mode, with the minimum dependencies
needed, and with Postgresql as the database.

Next, we'll need to configure our `Gemfile.lock` file to support the same OS as
Heroku, which runs Ubuntu. This way, regardless of what OS you're using in
development, `bundler` will be able to install the same gems on Heroku using any
Ubuntu-specific gem dependencies.

`cd` into the app, and run this command:

```console
$ bundle lock --add-platform x86_64-linux --add-platform ruby
```

This will add additional platforms to your `Gemfile.lock` file that will allow
the necessary dependencies to be installed after you deploy your app.

## Building the Demo App

Next, let's set up up a migration, model, route, and controller so that
we have some data to display in our application:

```console
$ rails g resource Bird name species
```

Add this data to the `db/seeds.rb` file:

```rb
Bird.create!(name: 'Black-Capped Chickadee', species: 'Poecile Atricapillus')
Bird.create!(name: 'Grackle', species: 'Quiscalus Quiscula')
Bird.create!(name: 'Common Starling', species: 'Sturnus Vulgaris')
Bird.create!(name: 'Mourning Dove', species: 'Zenaida Macroura')
```

Then run this command to generate the database and run the migrations and seed
file:

```console
$ rails db:create db:migrate db:seed
```

> `rails db:create` creates a new Postgresql database to be associated with your
> application based on the configuration in the `config/database.yml` file.
> Unlike with SQLite, the actual database file isn't created in the `db` folder;
> it lives elsewhere in your file system, depending on your Postgresql
> configuration. If you have problems with this step, see the
> **Troubleshooting** section below.

Finally, edit the `app/birds_controller.rb` file and add an `index` action:

```rb
  # GET /birds
  def index
    birds = Bird.all
    render json: birds
  end
```

To make sure the app works locally before deploying, run `rails s` and visit
[http://localhost:3000/birds](http://localhost:3000/birds).

## Deploying

Now that we've got some working code, it's time to get that code to run on a
Heroku server! The process of uploading our code to Heroku is managed by Git.
This makes it easy to deploy new versions using a tool most developers,
including yourself, are already familiar with.

Make a commit to save your changes:

```console
$ git add .
$ git commit -m 'Initial commit'
```

Next, you'll need to create an application on Heroku:

```console
$ heroku create
```

This command will generate a new application in your Heroku account, and
configure a new remote repository where you can push up your code. You can
confirm the remote repository was created successfully by running:

```console
$ git config --list --local | grep heroku
remote.heroku.url=https://git.heroku.com/aqueous-sierra-44713.git
remote.heroku.fetch=+refs/heads/*:refs/remotes/heroku/*
```

Now, deploying your code is as simple as using `git push` to upload the changes
from your repository to Heroku:

```console
$ git push heroku main
```

> Note: depending on your Git configuration, your default branch might be named
> `master` or `main`. You can verify which by running
> `git branch --show-current`. If it's `master`, you'll need to run
> `git push heroku master` instead.

You've successfully pushed up your code!

## Building and Migrating

After pushing up your code, you'll notice a flurry of activity in the terminal.
This indicates that Heroku is in the process of building your application, by:

- Setting up a Ruby environment to run your code in
- Installing the gems for your project with `bundle install`
- Running `rails server` to start up your server

If the build process succeeds, your app is live and online!

Before visiting the site, let's also set up the database. Remember, the database
is a separate application from your Rails application. Thankfully, we can get
it set up easily by running a few Rake commands on the server.

To migrate and seed the database on the server, run:

```console
$ heroku run rails db:migrate db:seed
```

When you prefix any command with `heroku run`, it will run that command on the
server where your application was deployed. This command is very useful for
troubleshooting: you can even run `heroku run rails c` to open a Rails console
on the server!

You can now visit the site in the browser by running `heroku open`. Note that,
because there is no root path (`'/'`) defined in our routes, you will see a
Page Not Found error when the app opens.

Navigate to the `/birds` endpoint and verify that you are able to see an array
of JSON data for all the birds in the database. If you aren't able to, check out
the troubleshooting section below, or the
[troubleshooting guide on Heroku][troubleshooting guide on heroku].

## Adding New Features

Since Heroku integrates the deploying process with Git, it's straightforward
to add new features to your code and deploy them. Let's start by adding a new
controller action in the `BirdsController`:

```rb
def show
  bird = Bird.find(params[:id])
  render json: bird
rescue ActiveRecord::RecordNotFound
  render json: "Bird not found", status: :not_found
end
```

Test your code locally by running `rails s` and visiting
[http://localhost:3000/birds/1](http://localhost:3000/birds/1).

After adding this code, make a commit:

```console
$ git add app/controllers/birds_controller.rb
$ git commit -m 'Added show action'
```

Then, to deploy the changes, push the new code up to Heroku:

```console
$ git push heroku main
```

After pushing new code, Heroku will run through the build process again and
deploy your changes. You don't have to run your migrations again since the
database already exists on the server. You would have to run the migrations if
you created a new migration file.

## Troubleshooting

If you ran into any errors along the way, here are some things you can try to
troubleshoot:

- If you're on a Mac and got a server connection error when you tried to run
  `rails db:create`, one option for solving this problem for Mac users is to
  install the Postgres app. To do this, first uninstall `postgresql` by running
  `brew remove postgresql`. Next, download the app from the
  [Postgres downloads page][] and install it. Launch the app and click
  "Initialize" to create a new server. You should now be able to run
  `rails db:create`.

- If you're using WSL and got the following error running `rails db:create`:

  ```txt
  PG::ConnectionBad: FATAL:  role "yourusername" does not exist
  ```

  The issue is that you did not create a role in Postgres for the default user
  account. Check [this video](https://www.youtube.com/watch?v=bQC5izDzOgE) for
  one possible fix.

- If your app failed to deploy at the build stage, make sure your local
  environment is set up correctly by following the steps at the beginning of
  this lesson. Check that you have the latest versions of Ruby and Bundler, and
  ensure that Postgresql was installed successfully.

- If you deployed successfully, but you ran into issues when you visited the
  site, make sure you migrated and seeded the database. Also, make sure that
  your application works locally and try to debug any issues on your local
  machine before re-deploying. You can also check the logs on the server by
  running `heroku logs`.

For additional support, check out these guides on Heroku:

- [Deploying a Rails 6 App to Heroku][heroku rails deploying guide]
- [Rails Troubleshooting on Heroku][troubleshooting guide on heroku]

## Conclusion

Congrats on deploying your first Rails app to the world wide web! Understanding
the deployment process and what it takes to run your application on another
computer is an important step toward becoming a full-stack developer. Like
anything new, this process can be daunting the first time you try it, but with
practice and exposure, you'll build confidence over time.

In the next lesson, we'll work on deploying a more complex application with a
Rails API backend and a React frontend, and talk through some of the challenges
of running these two applications together.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. When creating a new Rails app from the terminal, what additional flag do you
   need to use to be able to deploy it on Heroku?
2. What familiar process is used for deploying code to Heroku? How does the
   process differ when you're deploying to Heroku vs. developing code locally?

## Resources

- [Deploying a Rails 6 App to Heroku][heroku rails deploying guide]
- [Rails Troubleshooting on Heroku][troubleshooting guide on heroku]

[heroku signup]: https://signup.heroku.com/devcenter
[heroku cli]: https://devcenter.heroku.com/articles/heroku-cli#download-and-install
[supported runtimes]: https://devcenter.heroku.com/articles/ruby-support#supported-runtimes
[postgresql wsl]: https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database#install-postgresql
[heroku rails deploying guide]: https://devcenter.heroku.com/articles/getting-started-with-rails6
[troubleshooting guide on heroku]: https://devcenter.heroku.com/articles/getting-started-with-rails6#troubleshooting
[postgres downloads page]: https://postgresapp.com/downloads.html
