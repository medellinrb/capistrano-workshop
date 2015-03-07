# Rails Deployments with Capistrano

Sample app for Capistrano workshop


## Ubuntu 14.04 - Capistrano - Nginx + Passenger

## 1. Create droplet (don´t forget ssh keys)

```bash
ssh-keygen -t rsa -C "lerolero@gmail.com"
pbcopy < ~/.ssh/bandolero_rsa.pub
```

## 2. SSH your server

`ssh root@104.236.202.145 -i ~/.ssh/bandolero_rsa`

### Add deployer user

```bash
groupadd admin
useradd deployer -G admin -m -s /bin/bash
passwd deployer
```

## 3. Install all the things

### Ruby & dependencies

```bash
sudo apt-get update
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

# Qt and xvfb-run for Capybara Webkit
sudo apt-get install -y libqtwebkit-dev xvfb

# ImageMagick and Rmagick
sudo apt-get install -y imagemagick libmagickwand-dev

# SQLite, Git and Node.js
sudo apt-get install -y libsqlite3-dev git nodejs
```

### RVM

```bash
sudo apt-get install libgdbm-dev libncurses5-dev automake libtool bison libffi-dev
# Download keys if needed
# gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
curl -L https://get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm
rvm install 2.2.0
rvm use 2.2.0 --default
ruby -v
```

### Nginx and Passenger (from Phusion repository)

```bash
# Add HTTPS support for APT if needed
# sudo apt-get install apt-transport-https

# Download Phusion PGP keys
gpg --keyserver keyserver.ubuntu.com --recv-keys 561F9B9CAC40B2F7
gpg --armor --export 561F9B9CAC40B2F7 | sudo apt-key add -

# Add phusion repositories
sudo sh -c "echo 'deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main' >> /etc/apt/sources.list.d/passenger.list"
sudo chown root: /etc/apt/sources.list.d/passenger.list
sudo chmod 600 /etc/apt/sources.list.d/passenger.list

# Install nginx and passenger
sudo apt-get update
sudo apt-get install nginx-full passenger
```

## 4. Configure

### Nginx config

```bash
# sudo vim /etc/nginx/nginx.conf` and uncomment as needed:
passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /usr/bin/ruby;
# passenger_ruby /home/deployer/.rvm/wrappers/ruby-2.2.0/ruby; # If using rvm
# passenger_ruby /home/deployer/.rbenv/shims/ruby; # If using rbenv
```

### Database

```bash
# sudo apt-get install mysql-server mysql-client libmysqlclient-dev
sudo apt-get install postgresql postgresql-contrib libpq-dev

sudo su - postgres
createuser --pwprompt
exit

# sudo sudo -u postgres psql -1 -c "CREATE USER pgadmin WITH PASSWORD 'secret';"
# sudo sudo -u postgres psql -1 -c "ALTER USER pgadmin WITH SUPERUSER;"
# sudo sudo -u postgres psql -1 -c "CREATE DATABASE awesomeapp_prod;"
```

## 5. Capistrano

### Gems

```ruby
group :development do
  # ....

  gem 'capistrano'
  gem 'capistrano-rails'
  gem 'capistrano-bundler'
  # gem 'capistrano-rbenv'
  gem 'capistrano-rvm'
  gem 'capistrano-passenger'
end

bundle install
bundle --binstubs
cap --install
```

### Config

#### Capfile
```ruby
# Load DSL and set up stages
require 'capistrano/setup'

require 'capistrano/deploy'
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano/passenger'

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }

```

#### config/deploy.rb
```ruby
lock '3.4.0'

set :application, 'awesomeapp'
set :repo_url, 'git@github.com:Mardiniii/scrapping-workshop.git'

# Default branch is :master
# ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp

set :deploy_to, '/home/deployer/awesomeapp'
set :scm, :git
set :linked_files, fetch(:linked_files, []).push('config/database.yml', 'config/secrets.yml')
set :linked_dirs, fetch(:linked_dirs, []).push('log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system')
set :keep_releases, 5

namespace :deploy do

  # Not needed if using capistrano-passenger gem/recipe
  # desc 'Restart application'
  # task :restart do
  #   on roles(:app), in: :sequence, wait: 5 do
  #     execute :touch, release_path.join('tmp/restart.txt')
  #   end
  # end

  after :publishing, 'deploy:restart'
  after :finishing, 'deploy:cleanup'
end

```

#### config/deplot/production.rb

`server '104.236.76.72', user: 'deployer', roles: %w{app db web}`

## 6. Latest server configs

### Tell nginx about capistrano app folder

#### /etc/nginx/sites-enabled/default

```bash
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        # server_name mydomain.com;
        server_name localhost;
        passenger_enabled on;
        rails_env    production;
        root         /home/deployer/awesomeapp/current/public;

        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}
```

### Create Linked files

#### /home/deployer/awesomewapp/shared/config/database.yml

```
pg: &pg
  adapter: postgresql
  encoding: unicode
  username: pgadmin
  password: secret
  host: localhost
  pool: 5

production:
  <<: *pg
  database: awesomeapp_prod
```

#### /home/deployer/awesomewapp/shared/config/secrets.yml

```
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

## 7. Deploy

```bash
cap production deploy
cap production deploy:restart
cap -T
```
