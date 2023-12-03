# Create New Project

```
rails new projectname --api --database=postgresql
```

* `--api`: Creates a project in API mode
* `--database=postgresql`: Uses PostgreSQL as database

# Configuring Database

Make sure you have `dotenv-rails` in your `Gemfile`:

```
gem "dotenv-rails"
```

User settings for interacting with PostgreSQL will be stored under `.env` which should be ignored by `git` so make sure the following is found in `.gitignore`:

```
.env
```

**NOTE:**
* In Rails 7.1.2, `/.env` is already ignored.

Create a new template file called `.dist-env` which will be part of the repository. Include the database user parameters there:

```
PG_USERNAME=developer
PG_PASSWORD=password
PG_HOST=localhost
```

Add the following default parameters for the user to be used for accessing PostgreSQL database under `config/database.yml`:

```yaml
default: &default
	adapter: postgresql
	encoding: unicode
	pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
	username: <%= ENV.fetch("PG_USERNAME") %>
	password: <%= ENV.fetch("PG_PASSWORD") %>
	host: <%= ENV.fetch("PG_HOST") %>
```

Create the database to test if all things are working:

```bash
bundle exec rails db:create
```
# Making CORS Work

Cross Origin Resource Sharing has to be enabled for local testing. To enable it, first uncomment or add the following gem to the `Gemfile`:

```
gem "rack-cors"
```

Make sure that you have a file in `config/initializers.cors.rb` with the following content:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"

    resource "*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

# Using `uuid` as Primary Key

Enable `pgcrypto` extension in PostgreSQL by creating a migration file with the following:

```ruby
def change
	enable_extension 'pgcrypto'
end
```

In the file (create if not present) `config/initializers/generators.rb`, set `ActiveRecord` to use `uuid` as primary keys when creating a model:

```ruby
Rails.application.config.generators do |g|
	g.orm :active_record, primary_key_type: :uuid
end
```