# README
This is a starting package to initiate a ruby on rails project with the use of webpacker to separate frontend from rails, utilizing docker containers. It is required to familiarize with docker and its concept before using it.


## Credit
* jetthoughts for creating an updated project just in time, and well documentation on the readme. https://github.com/jetthoughts/vuejs-rails-5-starterkit

* soulmates-ai made a fabulous post on medium and their github demonstrating how to tweak ruby on rails with webpacker to work with docker.
https://medium.com/soulmates-ai/running-a-rails-app-with-webpacker-and-docker-8d29153d3446

## Dependencies
* docker
* docker-compose

follow offical guide for proper installation: [INSTALL DOCKER](https://docs.docker.com/install/)

## Versions
* ruby 2.5.3
* rails 5.2.1
* nodejs 10
* yarn
* webpacker 4
* vuejs 2

## TL;DR
Download the demo project
```bash
git clone https://github.com/cjchen-griton/rubyonrails5-vuejs.git
```
Build the image
```bash
#go to project
cd rubyonrails5-vuejs

#build image
docker-compose build
```
once build, run container
```bash
#run docker container
docker-compose up
```
check localhost:3000, open up console and you shall see vue configuration running.

## STEP BY STEP Tutorial from no files, file preparation & explanation.
### 1. Create Dockerfile to form a base image.
Pull base ruby image from [ruby's dockerhub](https://hub.docker.com/_/ruby/)
```dockerfile
FROM ruby:2.5.3
```
Install dependencies, notice we use PPA (personal package archive) to ensure nodejs is above version 6 since the original version that came along ruby image (debian distro) was 4 [digitalocean tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-16-04).
```dockerfile
RUN apt-get -y update && \
      apt-get install --fix-missing --no-install-recommends -qq -y \
        build-essential \
        vim \
        wget gnupg \
        git-all \
        curl \
        ssh \
        postgresql-client libpq5 libpq-dev libssl1.0-dev -y && \

      #install nodejs 10
      wget -qO- https://deb.nodesource.com/setup_10.x  | bash - && \
      apt-get install -y nodejs && \

      #insall yarn
      wget -qO- https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
      echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \

      # clean up
      apt-get update && \
      apt-get install yarn && \
      apt-get clean && \
      rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```
Finally, we copy all files into the image and run `bundle install` to install the gems.
```dockerfile
RUN mkdir /myapp
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp
```
Final Dockerfile
```Dockerfile
FROM ruby:2.5.3

RUN apt-get -y update && \
      apt-get install --fix-missing --no-install-recommends -qq -y \
        build-essential \
        vim \
        wget gnupg \
        git-all \
        curl \
        ssh \
        postgresql-client libpq5 libpq-dev libssl1.0-dev -y && \
      wget -qO- https://deb.nodesource.com/setup_10.x  | bash - && \
      apt-get install -y nodejs && \
      wget -qO- https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
      echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
      apt-get update && \
      apt-get install yarn && \
      apt-get clean && \
      rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN mkdir /myapp
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp
```

### 2. Create docker-compose.yml file.
According to [webpacker's docker README](https://github.com/rails/webpacker/blob/master/docs/docker.md), in order to make webpack server to work in docker, we need to expose its service port as a new service outside of container in docker-compose.yml file. We will comment out webpacker service first to allow initial project struture with webpacker being built, this part will be later uncomment to run webpacker service together.

```yml
version: '3'
services:
  db:
    image: postgres:10.3
    volumes:
    - ./tmp/db:/var/lib/postgresql/data
#  webpacker:
#    build: .
#    env_file:
#      - '.env.docker'
#    command: ./bin/webpack-dev-server
#    volumes:
#      - .:/myapp
#    ports:
#      - '3035:3035'
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
    - .:/myapp
    ports:
    - "3000:3000"
    depends_on:
    - db
#    - webpacker
```

## 3. Create environment file for docker-compose.yml to read
Create a file `.env.docker`, it contains configuration for webpack-dev-server. Put this document under the root folder beside your `docker-compose.yml` file (or anywhere as long the location aligns within the `docker-compose.yml` file). We'll change the content when its ready for production.

.env.docker
```docker
NODE_ENV=development
RAILS_ENV=development
WEBPACKER_DEV_SERVER_HOST=0.0.0.0
```

## 4. Gemfile & Gemfile.lock
Create two files, a `Gemfile` and empty `Gemfile.lock` for ruby on rails to initialize contents for your project.
```ruby
#Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 5.2.1'
```
create empty _Gemfile.lock_
```bash
$ touch Gemfile.lock
```

## 5. Create the app project
There are a few step for a new project. We'll run docker-compose on web service allowing ruby on rails to create the struture and files for us first and resolve other competancy issue of webpacker later.

run a container to install new project with webpacker and vuejs.
```bash
$ docker-compose run web rails new . --force --webpack=vue --database=postgresql
```

## 6. Create docker container services
You'll notice all files are under root permission of docker since it is created by the container itself, you'll need to change the files permission to edit them everytime you allow app within the container (e.g. rails) create new files in your project. Do this on a new terminal.

1. Change file owner permission
```bash
$ sudo chown -R $USER:$USER .
```
2. Rebuild docker services
```bash
$ docker-compose build
```
3. Connect database by adjusting `config/database.yml` contents
```yml
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # http://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: db
  username: postgres
  password: [PASSWORD]

development:
<<: *default
database: myapp_development

test:
<<: *default
database: myapp_test
```
## 7. configure development environment for webpacker

The webpacker service will display an error message when you start all services since its not fully install yet. Edit your `Gemfile` by adding the gem for webpacker. Follow webpacker's [README](https://github.com/rails/webpacker/blob/master/README.md) file to install.

Enable unsafe-eval rule for webpacker-dev-server.

add the following to `config/initializers/content_security_policy.rb` at the end.
```rb
Rails.application.config.content_security_policy do |policy|
  if Rails.env.development?
    policy.script_src :self, :https, :unsafe_eval
    policy.connect_src :self, :https, 'http://localhost:3035', 'ws://localhost:3035'
  else
    policy.script_src :self, :https
  end
end
```
Uncomment `system('bin/yarn')` in `bin/setup` and `bin/update` to install new modules

Change file owner permission
```bash
$ sudo chown -R $USER:$USER .
```
Uncomment webpacker service in `docker-compose.yml`
```yml
version: '3'
services:
  db:
    image: postgres:10.3
    volumes:
    - ./tmp/db:/var/lib/postgresql/data
  webpacker:
    build: .
    env_file:
      - '.env.docker'
    command: ./bin/webpack-dev-server
    volumes:
      - .:/myapp
    ports:
      - '3035:3035'
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
    - .:/myapp
    ports:
    - "3000:3000"
    depends_on:
    - db
    - webpacker
```

upgrade webpacker to 4
change gem
```Gemfile
gem 'webpacker', '>= 4.0.x'
```
rebuild container to fetch gem
```bash
docker-compose build
```
Create database
```bash
$ docker-compose run web rails db:create db:migrate
```
or enter the service container directly to run the command.
```bash
$ docker exec -it [container_id] bash

#container
$ rails db:create db:migrate
```
Initiate docker services
```bash
$ docker-compose up
```

2. Install dependencies.
```bash
$ ./bin/setup
```

```bash
bundle update webpacker
bin/rails webpacker:install
bin/rails webpacker:install:vue
```

4. Upgrade packages by yarn
```bash
yarn upgrade
yarn add @rails/webpacker@next
yarn upgrade webpack-dev-server --latest
yarn install
```

5. Enable Vue.js single components
Install `@vue/babel-preset-app`
```bash
yarn add @vue/babel-preset-app
```

6. Replace `@babel/preset-env` with `@vue/app` in `.bablerc`


4. adjust `config/webpacker.yml` file for webpacker service in docker container.
```yml
#change under development server settings
dev_server:
  https: false
  host: webpacker
  port: 3035
  public: localhost:3035
  hmr: true
```

7. Edit `app/views/layout/application.html.erb` to render js files under `app/javascript`
replace this part
```
<%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
<%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
<%= javascript_pack_tag 'hello_vue' %>
<%= stylesheet_pack_tag    'hello_vue', media: 'all' %>
```
with
```
<!-- <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %> -->
<!-- <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %> -->
<%= javascript_pack_tag 'hello_vue' %>
<%= stylesheet_pack_tag    'hello_vue', media: 'all' %>
```

create a home#index controller and view for vue's demo app
```
rails g controller home index
```
edit `config/route.rb`
```
get 'home/index'
```

```
root 'home#index'
```

Reboot docker container service.

Go to your browser and check localhost:3000
