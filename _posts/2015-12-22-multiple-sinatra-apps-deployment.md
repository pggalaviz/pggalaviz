---
layout: post
title:  "Run multiple Sinatra apps on a single server (CentOS 7)"
author: Pedro G. Galaviz
date:   2017-05-22 12:00:00 -0600
comments: true
tags: [ ruby, sinatra, centos ]
---

There's a considerable lack of information about rack apps deployment other than Ruby on Rails, so I made this small tutorial for deploying Sinatra apps to CentOS 7 servers.

I'll use a [Digital Ocean][dg] droplet as an example but should work anywhere else. These are some of the technologies we'll use:

* CentOS 7 (Operative System)
* Nginx (Front Facing Web Server)
* Unicorn (App Server)
* Ruby 2.2.2 (Programming Language)
* Sinatra (DSL for Rack apps)

## Server Configuration

This is a very basic server configuration, there are multiple steps to secure our server but we won't talk about this on this post.

### Log In

Conect to your server, log as the `root` user using the following command (substitute the highlighted word with your server's public IP address):

{% highlight shell %}
ssh root@SERVER_IP_ADDRESS
{% endhighlight %}

Complete the login process by accepting the warning about host authenticity if it appears, then providing your root authentication (password or private key).

### Create user

This example creates a new user called "pggalaviz", but you should replace it with a user name that you like:

{% highlight shell %}
adduser pggalaviz
{% endhighlight %}

Next, assign a password to the new user (again, substitute "pggalaviz" with the user that you just created):

{% highlight shell %}
passwd pggalaviz
{% endhighlight %}

Enter a strong password, and repeat it again to verify it.
We have a new user account with regular account privileges. However, we need to do administrative tasks.
As root, run this command to add your new user to the wheel group (substitute the highlighted word with your new user):

{% highlight shell %}
gpasswd -a pggalaviz wheel
{% endhighlight %}

Now your user can run commands with super user privileges!

### Public Key Authentication

If you don't have a public key already, generate one by running this command at the terminal of your local machine (your computer not the server)!

{% highlight shell %}
ssh-keygen
{% endhighlight %}

Press 'return key' to accept (don't modify the path) and add a password to your keys (this last step is optional).
If you already had or you just created your keys run this command on your local machine to print your public key (id_rsa.pub):

{% highlight shell %}
cat ~/.ssh/id_rsa.pub
{% endhighlight %}

it should print something like this:

{% highlight shell %}
ssh-rsa AAAAB3N.......zaC1yc2 localuser@machine.local
{% endhighlight %}

Select it all and copy it to the clipboard.
On the server as the root user enter this command to switch to the user we created at the begining of this tutorial:

{% highlight shell %}
su - pggalaviz
{% endhighlight %}

Now we're going to create a folder named .ssh and change its permissions for security reasons:

{% highlight shell %}
mkdir .ssh
chmod 700 .ssh
{% endhighlight %}

Create a new file by running:

{% highlight shell %}
vi .ssh/authorized_keys
{% endhighlight %}

Enter 'insert' mode by pressing `i` and paste your previoulsy copied ssh key. press ESC and `:x` to save changes.

Change the file permissions:

{% highlight shell %}
chmod 600 .ssh/authorized_keys
{% endhighlight %}

and return to the root user by running:

{% highlight shell %}
exit
{% endhighlight %}

Now, you'll be able to login to your server by running something like

{% highlight shell %}
ssh pggalaviz@SERVER_IP_ADDRESS
{% endhighlight %}

### Restrict Root Login

Log in to your server using your user (not root) and open the SSH configuration file by running:

{% highlight shell %}
vi /etc/ssh/sshd_config
{% endhighlight %}

*Probably you'll need to add sudo before the command.*

Look for a line that looks like: `#PermitRootLogin yes`.

Enter insert mode by pressing `i` and edit the line so it looks like `PermitRootLogin no`

**Disabling remote root login is highly recommended on every server!**

Press `:x` and enter to save changes and then run the following command to reload SSH.

{% highlight shell %}
sudo systemctl reload sshd.service
{% endhighlight %}

___

## Install Dependencies

### Install EPEL and update

First install the EPEL package:

{% highlight shell %}
sudo yum install epel-release
{% endhighlight %}

Then you should confirm that all packages are updated by running:

{% highlight shell %}
sudo yum update
{% endhighlight %}

### Install Packages

Install all the packages needed such as gcc, make, git, binutils, etc.

{% highlight shell %}
sudo yum install -y git-core zlib zlib-devel gcc-c++ patch readline readline-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison curl sqlite-devel curl-devel
{% endhighlight %}

### Setting Up a Ruby Environment

We'll use 'rbenv' to install and manage Ruby versions, just run these commands:

{% highlight shell %}
cd
git clone git://github.com/sstephenson/rbenv.git .rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
exec $SHELL

git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bash_profile
exec $SHELL
{% endhighlight %}

you can check installation by running `rbenv`.

If nothing appears then logout from the server, log in again using ssh and run `rbenv`, everything should work now.

Now we'll install Ruby:

{% highlight shell %}
rbenv install -v 2.2.2
{% endhighlight %}

This will install Ruby version 2.2.2, this can take some minutes.
After install let's set the default version our shell will use by running:

{% highlight shell %}
rbenv global 2.2.2
{% endhighlight %}

Finally let's verify ruby was installed properly with this command:

{% highlight shell %}
ruby -v
{% endhighlight %}

After ruby is installed run this command to install gems without documentation and save yourself some time and disk usage:

{% highlight shell %}
echo "gem: --no-document" > ~/.gemrc
{% endhighlight %}

Now lets install some basic gems:

{% highlight shell %}
gem install bundler rack sinatra unicorn
{% endhighlight %}

Whenever you install a new version of Ruby or a gem that provides commands, you should run the rehash sub-command. This will install shims for all Ruby executables known to rbenv, which will allow you to use the executables:

{% highlight shell %}
rbenv rehash
{% endhighlight %}

___

## Create a simple Sinatra App

Just for this tutorial lets create a simple sinatra app.

{% highlight shell %}
cd
mkdir sample_app
cd sample_app
{% endhighlight %}

We'll use Sinatra's modular style, start by creating the following archives and folders inside our sample_app folder:

{% highlight shell %}
assets
  |_ css
      |_app.scss
  |_ js
  |_images
log
public
tmp
  |_pids
  |_sockets
views
  |_ layouts
  |_ home.erb
app.rb
config.ru
Gemfile
unicorn.rb
{% endhighlight %}

Inside our Gemfile lets add:

{% highlight ruby %}
#=> Gemfile

source "https://rubygems.org"

gem 'sinatra'
gem 'sass'
gem 'unicorn'
{% endhighlight %}

Inside the app.rb file lets add:

{% highlight ruby %}
#=> app.rb
ENV['RACK_ENV'] ||= 'development'
$: << File.expand_path('../', __FILE__)

require 'sinatra/base'
require 'sass'

module SampleApp
  class App < Sinatra::Base

    #****************
    #Configuration
    #****************
    set :root, File.dirname(__FILE__)
    configure do
      enable :sessions
    end

    #****************
    #Assets Routes
    #****************

    get '/css/*.css' do
      content_type 'text/css', :charset => 'utf-8'
      filename = params[:splat].first
      scss filename.to_sym, :views => "#{settings.root}/assets/css", :style => :compressed
    end

    #****************
    #Main Routes
    #****************

    get '/' do
      erb :home
    end

  end
end
{% endhighlight %}


Inside the config.ru lets add:

{% highlight ruby %}
#=> config.ru

require 'rubygems'
require 'bundler'

Bundler.require :default, ENV['RACK_ENV'].to_sym

require File.expand_path '../app.rb', __FILE__

run SampleApp::App
{% endhighlight %}

Inside the views/home.erb lets add:

{% highlight html %}
<html>
  <head>
    <meta charset="UTF-8">
    <title>Sample App</title>
    <link rel="stylesheet" type="text/css" href="/css/app.css">
  </head>

  <body>
    <h1>Pedro G. Galaviz</h1>
    <p>If you can see me, it's working!</p>
  </body>
</html>
{% endhighlight %}

Finally on app's directory run:

{% highlight shell %}
bundle install
{% endhighlight %}

## Configure Unicorn

Now we need to configure Unicorn to serve our app.

{% highlight shell %}
cd
cd sample_app
vi unicorn.rb
{% endhighlight %}

And copy this inside the file:

{% highlight shell %}
root = "/home/pggalaviz/sample_app"
worker_processes 2
working_directory root
timeout 30
pid "#{root}/tmp/pids/sample_app.pid"

stderr_path "#{root}/log/unicorn.log"
stdout_path "#{root}/log/unicorn.log"

listen "#{root}/tmp/sockets/sample_app.sock"
{% endhighlight %}

### Create Unicorn Init Script

Now we'll create an init script so we can manage our unicorn server.

{% highlight shell %}
sudo vi /etc/init.d/unicorn_sample_app
{% endhighlight %}

And inside this file we'll add:

```shell
#!/bin/sh

### BEGIN INIT INFO
# Provides:          unicorn
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the unicorn app server
# Description:       starts unicorn using start-stop-daemon
### END INIT INFO

set -e

USAGE="Usage: $0 "

# app settings
USER="pggalaviz"
APP_NAME="sample_app"
APP_ROOT="/home/$USER/$APP_NAME"
ENV="production"

# environment settings
PATH="/home/$USER/.rbenv/shims:/home/$USER/.rbenv/bin:$PATH"
CMD="cd $APP_ROOT && bundle exec unicorn -c unicorn.rb -E $ENV -D"
PID="$APP_ROOT/tmp/pids/$APP_NAME.pid"
OLD_PID="$PID.oldbin"

# make sure the app exists
cd $APP_ROOT || exit 1

sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
  test -s $OLD_PID && kill -$1 `cat $OLD_PID`
}

case $1 in
  start)
    sig 0 && echo >&2 "Already running" && exit 0
    echo "Starting $APP_NAME"
    su - $USER -c "$CMD"
    ;;
  stop)
    echo "Stopping $APP_NAME"
    sig QUIT && exit 0
    echo >&2 "Not running"
    ;;
  force-stop)
    echo "Force stopping $APP_NAME"
    sig TERM && exit 0
    echo >&2 "Not running"
    ;;
  restart|reload|upgrade)
    sig USR2 && echo "reloaded $APP_NAME" && exit 0
    echo >&2 "Couldn't reload, starting '$CMD' instead"
    $CMD
    ;;
  rotate)
    sig USR1 && echo rotated logs OK && exit 0
    echo >&2 "Couldn't rotate logs" && exit 1
    ;;
  *)
    echo >&2 $USAGE
    exit 1
    ;;
esac
```
Check that you change `# app settings` to your app and user data.

Save and exit the file. Now to enable unicorn to start on boot lets run:

{% highlight shell %}
sudo chmod 755 /etc/init.d/unicorn_sample_app
sudo chkconfig --levels 235 unicorn_sample_app on
{% endhighlight %}

Lets start our Unicorn server now:

{% highlight shell %}
sudo service unicorn_sample_app start
{% endhighlight %}

## Configure Nginx

### Install Nginx

First we need to install Nginx to our server. Just run:

{% highlight shell %}
sudo yum install nginx
{% endhighlight %}

next, we'll enable it to start on server boot:

{% highlight shell %}
sudo chkconfig --levels 235 nginx on
{% endhighlight %}

### Configuration

First we'll open the nginx configuration file by running:

{% highlight shell %}
sudo vi /etc/nginx/nginx.conf
{% endhighlight %}

Then change its content to look like the one below.

```nginx
user  pggalaviz;
worker_processes  auto;

error_log  /var/log/nginx/error.log;
pid        /run/nginx.pid;
events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    index   index.html index.htm;

    include /etc/nginx/conf.d/*.conf;
}
```

Check you changed the user to yours. Now let's create our server file:

{% highlight shell %}
sudo vi /etc/nginx/conf.d/sample_app.conf
{% endhighlight %}

and lets add this content:

{% highlight nginx %}
upstream sample_app_x {
  server unix:/home/pggalaviz/sample_app/tmp/sockets/sample_app.sock fail_timeout=0;
}

server {
    listen 80;
    server_name localhost sample_app.com;

    root /home/pggalaviz/sample_app/public;

    try_files $uri/index.html $uri @app;

    location @app {
        proxy_pass http://sample_app_x;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}
{% endhighlight %}

Change to match your user and app's data. Save and exit.
Now let's start our nginx server by running:

{% highlight shell %}
sudo service nginx start
{% endhighlight %}

If everything worked fine you should be able to visit your server IP address or your domain (if you already pointed it to the server) and see our home page.

By following this steps you can deploy any number of apps in the same server, just limited by your server's resources.

## Updating & Adding apps

Let's assume you're using **GIT** as your version control system and your Sinatra app code is living there (Any app).

You can log in to your server and clone the git repo to the folder where your apps will live.

For each app you'll need to create a "unicorn.rb" file inside the app's folder then a "Unicorn init script" and an "Nginx configuration file" on the server as we previously did.

you can start any unicorn process by typing:

{% highlight shell %}
sudo service unicorn_sample_app start
{% endhighlight %}

If you are updating your app first pull the repo code from git to your app's folder and then type:

{% highlight shell %}
sudo service unicorn_sample_app restart
sudo service nginx restart
{% endhighlight %}

And your app should be updated accordingly.

[dg]: https://www.digitalocean.com/?refcode=07dd39adf951
