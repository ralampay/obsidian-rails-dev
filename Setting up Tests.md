# Disable Default Tests

Disable default tests in rails by modifying `config/initializers/generators.rb` with the following content:

```ruby
Rails.application.config.generators do |g|
	g.test_framework nil
end
```
# Gems

For basic testing setup, the following gems will be used:
* `factory_bot_rails`: For creating stubs
* `faker`: For generating values
* `rspec-rails`: For test suite

In the `Gemfile`, make sure to have the gems setup under `:test` group:

```ruby
group :test do
	gem "factory_bot_rails"
	gem "faker"
	gem "rspec-rails"
end
```

Install the `rspec` helpers:

```bash
bundle exec rails generate rspec:install
```

# Creating a Basic Test: User Login

To go through the different components of testing, we'll create a test for user login. 

## Setup User Model

A user model will have the following attributes:
* `first_name`
* `last_name`
* `username`
* `email`
* `user_type`
* `password_hash`

Create a model for the user:

```bash
rails g model User first_name:string last_name:string username:string email:string password_hash:string user_type:string
```

Make sure that the migration file puts null constraints on the required fields:

```ruby
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users, id: :uuid do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :username, null: false
      t.string :email, null: false
      t.string :password_hash, null: false
      t.string :user_type, null: false

      t.timestamps
    end
  end
end
```

To setup proper values for `password_hash`, we'll need to have the `bcrypt` gem installed in our `Gemfile`:

```ruby
gem "bcrypt"
```

We'll also be using `jwt` for authentication so let's go ahead and add that gem:

```ruby
gem "jwt"
```

A salt value will be needed by the system so have a valid salt in `.env`:

```bash
SALT="\$2a\$12\$l0Y5QFpO4Y.SdXwtIInh4O"
```

Generating a proper salt value can be done via the console:

```ruby
BCrypt::Engine.generate_salt
```
## Setup User Factory

We can setup a factory to create user stubs by putting it under `spec/factories/user_factory.rb` with the following code:

```ruby
FactoryBot.define do
  factory :user do
    email { Faker::Internet.email }
    username { Faker::Internet.username(specifier: 10) }
    first_name { Faker::Name.first_name }
    last_name { Faker::Name.last_name }
    user_type { "admin" }
    password_hash do
      secret = ENV['SALT']
      BCrypt::Engine.hash_secret('password', secret)
    end
  end
end
```

## Setup Test Utilities

We can create helpers under `app/helpers`. For example, here's `app/helpers/api_helpers.rb` to provide a set of methods that will be used for authenticating a user:

```ruby
module ApiHelpers
  def build_jwt_header(token)
    { 'Authorization': "Bearer #{token}" }
  end
  
  def generate_password_hash(password)
    secret = ENV['SALT']
    BCrypt::Engine.hash_secret(password, secret)
  end

  def decode_jwt(token)
    JWT.decode(token, Rails.application.secret_key_base)
  end
end
```

This allows us to set a `jwt` token header, generate a password using `bcrypt` and decode a token.

## Setup Login Fail Test if No Parameters are Passed

If no parameters are passed to endpoint `POST /login`, http status should return status `:unprocessable_entity` and array of error messages stating that `username` and `password` is required.

```ruby
require 'rails_helper'

RSpec.describe 'Login' do
  include ApiHelpers

  let(:user) { FactoryBot.create(:user) }
  let(:api_url) { "/login" }

  describe "POST /login", type: :request do
    context "invalid calls" do
      it "returns error on no parameters passed" do
        post api_url

        expect(response).to have_http_status(:unprocessable_entity)

        payload = JSON.parse(response.body)

        expect(payload["username"]).to eq(["required"])
        expect(payload["password"]).to eq(["required"])
      end
    end
  end
end
```

## Running Tests Continuously

We can watch our codebase and re-rerun the test suite if changes are made via the `retest` gem. Install first as a global tool:

```bash
gem install retest
```

To continuously test, have a session that runs:

```bash
retest 'bundle exec rspec spec'
```