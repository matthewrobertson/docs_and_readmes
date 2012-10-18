# Installation Guide

## Purpose:

This document contains instructions for setting up a Ruby on Rails development on Mac OSX.

## High Level Overview:

The following steps are covered in more depth below:

1. Install dependencies
2. Install RVM
3. Install a ruby interpreter
4. Get the source code
5. Install Rails
6. Create the database
7. Run the code
8. Run the test suite

## Installing Dependencies:

### XCODE Command Line Utilities:

The ruby interpreter is written in C. The ruby version manager (RVM) compiles interpreters from source during installation. It is also common for Ruby Gems to include "native extensions" written in C or C++ that also need to be compiled during installation. For these reasons a compatible C compiler is required to install ruby. Currently the best option is to install the __Command Line Tools for Xcode__ that are available [here](https://daw.apple.com/cgi-bin/WebObjects/DSAuthWeb.woa/wa/login?appIdKey=d4f7d769c2abecc664d0dadfed6a67f943442b5e9c87524d4587a95773750cea&path=%2F%2Fdownloads%2Findex.action). In order to download the installer you need to sign in with your apple ID.

### Homebrew:

[Homebrew](http://mxcl.github.com/homebrew/) is an easy to use utility to install Unix utilies that are not included in OSX. Many Ruby Gems hold these libraries as dependencies, so it is a good idea to install Homebrew. Currenly the following command will download and run the Homebrew installation script, but you should check the [installation documentation](https://github.com/mxcl/homebrew/wiki/installation) to ensure it is up to date (LAST UPDATED July 5, 2012).

`/usr/bin/ruby <(curl -fsSkL raw.github.com/mxcl/homebrew/go)`

### SQLite

We are using [SQLite](http://www.sqlite.org/) as a development database. The installer can be downloaded [here](http://www.sqlite.org/download.html) or you can use homebrew:

`$ brew install sqlite`

### Git

We are using [Git](http://git-scm.com/) as a source control system. Many libraries will install it for you, so it is a good idea to check if you have it before installing. In a terminal type the command:

`$ git --version`

If you get a `command not found` error you can use Homebrew to install git:

`$ brew install git`

## Installing the Ruby Version Manager (RVM)

[RVM](https://rvm.io//) is a command line utility for managing multiple ruby environments on the same machine. It is not a hard dependency of our project but it is HIGHLY RECOMMENDED and the rest of this guide will assume you have it installed.

The following command will download and run the RVM install script:

`$ curl -L https://get.rvm.io | bash -s stable`

Again, if you are from the future you should check the [installation docs](https://rvm.io/rvm/install/) to ensure this command is current (LAST UPDATED July 5, 2012).

## Install a Ruby Interpreter

Upon completion of the RVM installation, a bunch of info will be dumped to the console about your system and any missing requirements it detected. If you cleared your console or this information did not print, you can get it back with:

`$ rvm requirements`

Do your best to sift through this info and make sure you are not missing anything really important (we are going to be installing `ruby 1.9.3`). If it warns you about XCode versions or tells you to install the OSX GCC compiler just ignore it, last I checked the Command Line Utilities for XCode worked fine but were not detected by RVM (LAST UPDATED JULY 5, 2012). Also, the beauty of RVM is that it makes it pretty hard to screw things up, so if you are unsure there is never any harm in trying to install an interpreter and letting the install fail.

Install version 1.9.3 of ruby:

`$ rvm install 1.9.3`

This will download the source and compile it which usually takes a few minutes, so now is a good time to grab a coffee. Once it completes you can set it to be used as your system’s default with:

`$ rvm use 1.9.3 --default`

and then check that it worked with:

`$ ruby -v`

(you can get more information about what you can do with rvm via `$ rvm usage`).

## Get the Source Code

Use git to check out the source code from its repository on github:

`$ git clone git@github.com:[REPOSITORY]`

_note: in order for this to work, you will first have to [configure ssh keys for github](https://help.github.com/articles/generating-ssh-keys)_

## Install Rails

The next step is to install rails and all of the gems we are using in the project. Rails uses a dependency management system called [Bundler](http://gembundler.com/) that makes this very simple. Here are the steps:

1. Create an RVM gemset to sandbox all of the gems we are about to install: `$ rvm gemset create application_name`
2. Tell RVM to start using that gemset: `$ rvm gemset use application_name`
3. Install bundler: `$ gem install bundler`
4. Switch into the application root directory: `$ cd application_name`
5. Trust the `.rvmrc`: the first time you switch into the project root you need to type `Y` to do this (more below)
6. Use bundler to install all the gems listed in the `Gemfile`: `$ bundle install --without production`

_`.rvmrc` explained: we have included a file in the root directory of the project with a special name: `.rvmrc`. When RVM encounters one of these files it runs its contents. All that ours does is set your environment to use `ruby 1.9.3` and the gemset called `application_name`. This prevents you from forgetting to switch and installing gems in the wrong gemsets._

## Create API Key

Copy the api key config

`$ cp config/example_api_keys.yml config/api_keys.yml`

## Create the Database

Our application uses a database and you need to create it to be able to run the code. The first step is to create the config file that tell rails about the database you are using. We do not check this file into source control because it is configuration specific and there are security concerns in production. However, we did create a sample file that mimics the content you need in yours. To copy this contents into the actual config file you can run this command from the project root:

`$ cp config/sqlite_example_database.yml config/database.yml`

Now rails will know how to connect to your sqlite installation so you can use it to create the development databases:

`$ rake db:create:all`

and get the shemas up to date:

`$ rake db:migrate`

and then seed the database: 

`$ rake seed`

## Run the Code

OK if you made it this far you should be ready to run the application. Rails comes with a lightweight development HTTP server that can be started on port 3000 with the command:

`$ rails server`

After a couple seconds you will see a dump of some info that says everything is up and running. Open a web browser and navigate to [http://localhost:3000](http://localhost:3000) and you should see the application's landing page. If it works congratulations you are ready to start working!!! 

## Run the Test Suite

We are using the [rspec testing framework](http://rspec.info/). However, before you start, you need to set up the test database with this command.

`$ rake db:test:prepare`

Rails uses a different database for the `test` and `development` environments because often tests will create and destroy data when they run.

Once that is done you can run the test suite with:

`$ rake spec`

You can also use a tool called [Guard](https://github.com/guard/guard/) to watch for filesystem changes and automatically run the corresponding specs.
First you need to install PhantomJS to enable guard to test javascript tests:

`$ brew install PhantomJS`

and then you can run guard:

`$ bundle exec guard`

To stop guard from running type 'quit' in the console. Guard will also try to run the javascript test suite. In order to get this working you need to install a headless js interpretter like this:

`$ brew install phantomjs`

## Next Steps:

The Ruby on Rails [getting started guide](http://guides.rubyonrails.org/getting_started.html) provides a nice overview of how things work in rails. A lot of the content overlaps with this guide.
