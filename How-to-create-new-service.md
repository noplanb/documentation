# How to create new service

## Gems

You may install some common gems globally.
(for rvm):

```shell
rvm use 2.2.2@global
gem install rails rails-api rollbar newrelic_rpm rspec-rails simplecov guard-rspec pry-rails better_errors shoulda-matchers
```

## Rails application

If your service is API service, I recommend use [rails-api gem](https://github.com/rails-api/rails-api) and use dash `-` for naming:

```shell
rails-api new -JT zazo-service
```

If you will not use ActiveRecord:

```shell
rails-api new -OJT zazo-service
```

## Configuration

We use [figaro fork](https://github.com/fny/figaro/tree/elastic-beanstalk-support) with EB support (Gemfile):

```ruby
gem 'figaro', github: 'fny/figaro', branch: 'elastic-beanstalk-support'
```

1. Make `bundle install`.
2. Create `config/application.yml` with ENV params required for application. Minimum are: `secret_key_base`, `rollbar_access_token`, `newrelic_license_key`.
3. Copy `config/application.yml` to `config/application.yml.example` without actual tokens and passwords and add it to Git.
4. To use application configuration (not secrets) add `settingslogic` gem and `config/settings.yml` with `app_name` and `version` parameters (look on *zazo-notification* service) and add it to Git.
5. Commit and push changes.

## RSpec & SimpleCov

Add this code to `spec/rails_helper.rb` before environment loading:

```ruby
reports_dir = ENV['WERCKER_REPORT_ARTIFACTS_DIR'] || File.expand_path('../../tmp', __FILE__)

if ENV.key?('coverage') || ENV.key?('CI')
  require 'simplecov'
  SimpleCov.coverage_dir File.join(reports_dir, 'coverage')
  SimpleCov.start :rails
end
```

And add HTML formatter in `RSpec.configure` block:

```ruby
if ENV.key?('CI')
  config.add_formatter :html, File.join(reports_dir, 'rspec', 'rspec.html')
end
```

Try run `rspec` on local box.

## Github

1. Create private repository under **noplanb** organization and push code to it.
2. Add repo to *ServerNonMainApiRW* team.
3. Add repo to Slack Github integration.

## Rollbar

Make sure `rollbar` gem is in your `Gemfile` and you made `rails g rollbar::install`.

1. Connect Github repo to [Rollbar](http://rollbar.com).
2. Copy access token to your `rollbar_access_token` config (application.yml).
3. Add project to Backend team.
4. Copy notification settings for new project from _tbm-server_.

## NewRelic RPM
Make sure `newrelic_rpm` in your `Gemfile`.

Create `config/newrelic.yml` as in *zazo-notification* service.

## Docker

1. If you don't use db, add `Dockerfile` like this:

```Dockerfile
FROM zazo/rails

EXPOSE 80
```

Otherwise:

```Dockerfile
FROM zazo/rails

EXPOSE 80
CMD rake db:migrate && foreman start
```

If you need assets compilation add this RUN commands after `FROM`:
```Dockerfile
RUN rake assets:precompile
RUN chown www-data:www-data -R /usr/src/app
```

**Notice**. All `RUN` commands will execute during build process and will NOT have environment variables set by EB console. Therefore, commands requres environment to be set, MUST be defined in `CMD`.
If you have many commands, put them in `bin/start.sh` shell script and set `CMD bin/start.sh`. ENV will be set on `docker run` stage during deploy to EB.

For convinient running docker commands on local box, you may copy `lib/tasks/docker.rake` file from *zazo-notification* (and other).

Try build and run container on local box.

2. Create `Dockerrun.aws.json` like this:

```json
{
  "AWSEBDockerrunVersion": 1,
  "Ports": [
    {
      "ContainerPort": "80"
    }
  ],
  "Volumes": [
  ],
  "Logging": "/usr/src/app/log"
}
```

3. Commit and push changes.

## AWS Elasticbeanstalk

1. Create application with same name as repo with *Generic Docker* stack.
2. Create *production* environment names with abbreviated service name and `-production`, eg. `zazo-service` => `zs-production`.
3. Do `eb init`.
3. You may set application environment variables to EB using figaro:
```shell
figaro eb:set -e production --eb-env zs-production
```

## Wercker

1. Create `wercker.yml` file in project root.

Simple wercker without DB:

```yaml
# This references the default Ruby container from
# the Docker Hub.
box: zazo/rails
# You can also use services such as databases. Read more on our dev center:
# http://devcenter.wercker.com/docs/services/index.html

# This is the build pipeline. Pipelines are the core of wercker
# Read more about pipelines on our dev center
# http://devcenter.wercker.com/docs/pipelines/index.html
build:
  steps:
    - asux/bundle-install@1.1.8:
        gemfile: config/Gemfile
        path: /usr/local/bundle
        without: development production
    - script:
        name: create rspec and coverage dir
        code: mkdir -p $WERCKER_REPORT_ARTIFACTS_DIR/rspec $WERCKER_REPORT_ARTIFACTS_DIR/coverage
    - script:
        name: rspec
        code: rspec
  after-steps:
    - asux/pretty-slack-notify:
        webhook_url: $slack_url
        channel: devops
deploy:
  steps:
    - asux/elastic-beanstalk-deploy:
        key: $AWS_ACCESS_KEY_ID
        secret: $AWS_SECRET_ACCESS_KEY
        app_name: $EB_APP
        env_name: $EB_ENV
        region: $AWS_REGION
  after-steps:
    - akelmanson/rollbar-notify:
        access-token: $rollbar_access_token
    - rafaelverger/newrelic-deployment:
        api_key: $newrelic_api_key
        app_name: $app_name ($WERCKER_DEPLOYTARGET_NAME)
    - asux/pretty-slack-notify:
        webhook_url: $slack_url
        channel: devops
```

or with PostgreSQL:

```yaml
# This references the default Ruby container from
# the Docker Hub.
# https://registry.hub.docker.com/_/ruby/
# If you want to use a specific version you would use a tag:
# ruby:2.2.2
box: zazo/rails
# This is the build pipeline. Pipelines are the core of wercker
# Read more about pipelines on our dev center
# http://devcenter.wercker.com/docs/pipelines/index.html
build:
  services:
    - id: postgres
      tag: 9.4
      env:
        POSTGRES_PASSWORD: secret888zazo
        POSTGRES_USER: postgres

  # Steps make up the actions in your pipeline
  # Read more about steps on our dev center:
  # http://devcenter.wercker.com/docs/steps/index.html
  steps:
    - asux/bundle-install@1.1.8:
        gemfile: config/Gemfile
        path: /usr/local/bundle
        without: development production
    - script:
        name: db setup
        code: rake db:create db:migrate RAILS_ENV=test
    - script:
        name: create rspec and coverage dir
        code: mkdir -p $WERCKER_REPORT_ARTIFACTS_DIR/rspec $WERCKER_REPORT_ARTIFACTS_DIR/coverage
    - script:
        name: rspec
        code: rspec
  after-steps:
    - asux/pretty-slack-notify:
        webhook_url: $slack_url
        channel: devops
deploy:
  steps:
    - asux/elastic-beanstalk-deploy:
        key: $AWS_ACCESS_KEY_ID
        secret: $AWS_SECRET_ACCESS_KEY
        app_name: $EB_APP
        env_name: $EB_ENV
        region: $AWS_REGION
  after-steps:
    - akelmanson/rollbar-notify:
        access-token: $rollbar_access_token
    - rafaelverger/newrelic-deployment:
        api_key: $newrelic_api_key
        app_name: $app_name ($WERCKER_DEPLOYTARGET_NAME)
    - asux/pretty-slack-notify:
        webhook_url: $slack_url
        channel: devops
```

and make sure `config/database.yml` looks like:

```yaml
default: &default
  adapter: postgresql
  encoding: utf8
  database: <%= ENV['POSTGRES_ENV_POSTGRES_DATABASE'] || Figaro.env.db_name %>
  username: <%= ENV['POSTGRES_ENV_POSTGRES_USER'] || Figaro.env.db_username %>
  password: <%= ENV['POSTGRES_ENV_POSTGRES_PASSWORD'] || Figaro.env.db_password %>
  host: <%= ENV['POSTGRES_PORT_5432_TCP_ADDR'] || Figaro.env.db_host %>
  port: <%= ENV['POSTGRES_PORT_5432_TCP_PORT'] || Figaro.env.db_port %>
  pool: <%= Figaro.env.db_pool || 10 %>

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default
```

2. Create application from Github repo under **Zazo** organization with Docker-enabled stack.
3. Configure environment variables there including AWS tokens for *wercker* IAM account and `EB_APP` with EB application name for service (`zazo-service`).
```yaml
AWS_ACCESS_KEY_ID: AKIAIRII2GGTOOZ75XQA
AWS_SECRET_ACCESS_KEY: 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
AWS_REGION: us-west-1
```
4. Add *production* target and set `EB_ENV` with EB environment (`zs-production`).
5. Try build and deploy.

# API Blueprint documentation

There is good practice to keep API documentation recent in [API Blueprint fromat](https://apiblueprint.org).
You may want integrate it with [Apiary](http://apiary.io) service and connect github repo with it.
In this case, documentation will sync to `apiary.apib` under project root in *master* branch.
Also, you may copy `lib/tasks/aglio.rake` from *zazo-notification* task to generate static HTML file with API documentation.
