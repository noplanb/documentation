# How to create new service

# Rails application

## Gems

Install `rails` and `rails-api` gems globally.
(for rvm):

```shell
rvm use 2.2.2@global
gem install rails rails-api
```

If your service is API service, I recommend use [rails-api gem](https://github.com/rails-api/rails-api) and use dash `-` for naming:

```shell
rails-api new -JT my-service
```