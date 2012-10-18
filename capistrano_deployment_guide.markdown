# Brief Deployment Overview

The following document gives a brief overview of our staging server environment and how to deploy the rails application to a new server.

## Server Details

- Amazon Micro instance 
- Ubuntu 12.04 Precise Pangolin
- ruby 1.9.3p194
- Rails 3.2.6
- nginx version: nginx/1.2.1 installed at `/opt/nginx/`
- nginx config file: `/opt/nginx/conf/nginx.conf`
- ip: `50.112.121.91`

## Deploy Process Overview

### 1. ssh into the system:

You will need to get ssh keys for the server for this. Amazon provides a .pem file that you should move to `~/.ec2/dev.pem`. You also need to set the permission level to be only readable by you:

`$ chmod 600 ~/.ec2/dev.pem`

and then you can ssh into the server with the following command:

`$ ssh -i ~/.ec2/dev.pem ubuntu@<server_dns>`

### 2. Generate ssh keys for checking out the source code:

`$ ssh-keygen -t rsa`

This will create two files: `~/.ssh/idrsa` and `~/.ssh/idrsa.pub`. You need to add the public key to your github account to allow the server to checkout the source code. Log into your github.com account and then navigate to: _account settings / SSH keys / Add SSH Key_Give the key a name and paste the output of `$ cat ~/.ssh/idrsa.pub` into _key_ field. 

Also, you will need to add github.com as a trusted host or your deploy will bounce the first time you try to checkout the code. The easiest way to do this is to try to ssh in:

`$ ssh github.com`

and then answer yes when prompted.


### 3. Install Dependencies 

Use apt-get:

`$ sudo apt-get install curl gcc make git-core libcurl4-openssl-dev apache2-prefork-dev libapr1-dev libaprutil1-dev libxslt-dev libxml2-dev`

(If fails, edit /etc/apt/sources.list to enable third party packages)

### 4. Install RVM:

[RVM](https://rvm.io//) is a command line utility for managing multiple ruby environments on the same machine. It is not a hard dependency of our project but it is HIGHLY RECOMMENDED and the rest of this guide will assume you have it installed.

The following command will download and run the RVM install script:

`$ curl -L https://get.rvm.io | bash -s stable`

Again, if you are from the future you should check the [installation docs](https://rvm.io/rvm/install/) to ensure this command is current (LAST UPDATED July 5, 2012).

### 5. Log out and then log back in

You can end your ssh session with `$ exit`. Once you have logged back in, if you enter the command:

`$ type rvm | head -1`

It should print:

`rvm is a function`

### 6. Install a ruby interpreter

`$ rvm install 1.9.3`

and set it as the system default:

`$ rvm use 1.9.3 --default`


### 7. Install mysql server 

You will be asked to generate a password, pick a strong one and write it down:

`$ sudo apt-get install mysql-server`

log in: 

`$ mysql -u root -p`

create a db:

`> CREATE DATABASE something DEFAULT CHARACTER SET utf8`

create a mysql user:

`> CREATE USER 'something_user'@'localhost' IDENTIFIED BY 'some_pass';`

grant privelages to the user on the created db:

`> GRANT ALL PRIVILEGES ON something.* TO 'something'@'localhost';`

__WRITE DOWN THE DATABASE NAME, USER NAME AND PASSWORD YOU JUST CREATED__

### 8. Install Phussion Passenger:

 Passenger is an Nginx module that allows rails to integrate with Nginx easily. More information is available on thier [site](http://www.modrails.com/documentation/Users%20guide%20Nginx.html). Install the gem in the global gemset and run the installer for NGINX.

 First get write permission for the nginx install dir:

 `$ sudo chmod 777 /opt`

`$ rvm gemset use global`

`$ gem install passenger`

`$ gem install bundler`

`$ passenger-install-nginx-module`

The set up process will check for missing dependencies and walk you through a couple extra steps. It will also offer to install NGINX for you, DO IT.

Start NGnix

`$ /opt/nginx/sbin/nginx`

now browse the server to make sure it works

### 9. Edit nginx config and add these lines.

`server {
  listen 80;
  server_name www.yourhost.com;
  root /somewhere/public;
  passenger_enabled on;
}`

### 10. Create a directory to deploy to

You also need to allow the deploy user write permssions:

`$ sudo mkdir /var/www`
`$ sudo chown ubunutu /var/www`

### 11. On the local machine set up capistrano

_Note: this step only needs to be done once per project, not for every server_

Add `gem 'capistrano'` to the project's `Gemfile` and then `$ bundle`. Now _capify_ the project:

`$ cd [project root dir]`
`$ capify .`

### 12. Edit capfile

_Note: this step only needs to be done once per project, not for every server_

When you _capified_ the project, capistrano created a `config/deploy.rb` file and a `Capfile` for you. These config files tell capistrano how to deploy your app. You need to fill in the details for your particular set up. 

In the `Capfile` you need to uncomment this line:

`load 'deploy/assets'`

`config/deploy.rb` is a little more complex. Here is a nice set of defaults to get you started:

```
require "bundler/capistrano"
require "rvm/capistrano"

server "50.112.121.91", :web, :app, :db, :primary => true

set :application, "application_name"
set :user, "ubuntu"
set :deploy_to, "/var/www/application_name"
set :deploy_via, :remote_cache
set :use_sudo, false

set :rvm_ruby_string, '1.9.3@application_name'
set :rvm_type, :user

set :scm, "git"
set :repository, "git@github.com:[repo-path]"
set :branch, "master"

default_run_options[:pty] = true
ssh_options[:forward_agent] = true
ssh_options[:keys] = [File.join(ENV["HOME"], ".ec2", "dev.pem")]

after "deploy", "deploy:cleanup" # keep only the last 5 releases

namespace :deploy do

  task :start do; end
  task :stop do; end
  task :restart, roles: :app, except: {no_release: true} do
    run "touch #{deploy_to}/current/tmp/restart.txt"
  end

  task :setup_config, roles: :app do
    run "mkdir -p #{shared_path}/config"
    # create a database.yml from the mysql example
    put File.read("config/mysql_example_database.yml"), "#{shared_path}/config/database.yml"
    
    # create a api_keys.yml from the example
    put File.read("config/example_api_keys.yml"), "#{shared_path}/config/api_keys.yml"

    puts "Now edit the config files in #{shared_path}."
  end
  after "deploy:setup", "deploy:setup_config"

  task :symlink_config, roles: :app do
    run "ln -nfs #{shared_path}/config/database.yml #{release_path}/config/database.yml"
    run "ln -nfs #{shared_path}/config/api_keys.yml #{release_path}/config/api_keys.yml"
  end
  after "deploy:finalize_update", "deploy:symlink_config"

  desc "Make sure local git is in sync with remote."
    task :check_revision, roles: :web do
    unless `git rev-parse HEAD` == `git rev-parse origin/master`
    puts "WARNING: HEAD is not the same as origin/master"
    puts "Run `git push` to sync changes."
    exit
    end
  end
  before "deploy", "deploy:check_revision"
end
```

### 13. Push the code onto the server for the first time.

To do this simply run the cap setup command:

`$ cap deploy:setup`

### 14. Now edit the config files

If you succeded at step 13 it will have finished with the following line of output:

`Now edit the config files in /var/www/application_name/shared`

That is exactly what you need to do next. SSH onto the server and fill in `/var/www/application_name/shared/config/database.yml` with the details of your production database and drop our LinkedIn api keys into `/var/www/application_name/shared/config/api_keys.yml`. Remember the stuff you wrote down in step 7?

_explanation: this step is for security purposes. We need to manage these config files securely because they contain sensitive information, thus we cannot check them into source control and push them up to github._

### 15. Run the deploy script for the first time.

End your ssh session, cross your fingers and run:

`$ cap deploy:cold`

### 16. Hit the url for your server. 

It should take a few seconds to load the first page, but if you don't get an error message you are done!!!

From now on you can deploy your app with:

`$ cap deploy`

and you can revert a bad deploy with:

`$ cap deploy:rollback`
