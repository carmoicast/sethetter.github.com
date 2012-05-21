---
layout: post
title: Testing User Authentication in Rails with RSpec
categories: ruby, ruby on rails, testing, rspec
---
Test driven development is something I've been meaning to integrate into my workflow for some time now. I know this is the situation with a lot of developers out there, so I'm writing a series of posts outlining how to develop a basic application in a test driven format. The application I'll be building is a personal journaling application that will employ the use of Rails for a back end, and Backbone.js on the front end interaction.

I plan to build and test the front end Backbone application in a test driven manner as well and will be writing that post (or posts) following this one. The project will be hosted in my [Github account](http://github.com/sethetter) for those who wish to reference the source code. Now let's get started.

# Creating Our Project

I'm calling the application DayLog. Rails will handle server side requests by returning JSON data of the resources requested. It will also handle user authentication and session management, which will be the focus of this first post in the series on DayLog.

We'll start by creating our project.

{% highlight sh %}
$ rails new daylog
{% endhighlight %}

Now we want to open up our Gemfile so we can add our RSpec dependency. We only need RSpec in our development and test environments, so create a new group for that gem. We'll also be adding in `bcrypt-ruby` for password encryption on our Users.

{% highlight ruby %}
# Gemfile

group :development, :test do
  gem 'rspec-rails', '~> 2.0'
end

gem 'bcrypt-ruby', '~> 3.0.0'
{% endhighlight %}

With that added we want to run `bundle` so rubygems can install all our dependencies. Once `rspec-rails` is installed on the system we want to install our spec files and folder in our rails project and removing one we don't need for this particular project:

{% highlight sh %}
$ rails g rspec:install
$ rm spec/helpers/users_helper_spec.rb
{% endhighlight %}

# Creating our User Resource

First things first, run our rails generator to create a user resource.

{% highlight sh %}
$ rails g resource user
{% endhighlight %}

Open up our migration file located in `db/migrate/` and add the following attributes.

{% highlight ruby %}
# db/migrate/#####_create_users.rb

class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.string :username
      t.string :password_digest

      t.timestamps
    end
  end
end
{% endhighlight %}

All we need for our user resource is a username and a password to log in with. Rails handily provides us with a convenience method for password encryption that requires us to have a `password_digest` attribute on our user resource

We'll look at how that works in a second. Now run `$ rake db:migrate` from the command line so we can add our resource to the database. We are now ready to write our first test.

# Testing Attribute Validation

This test will simply see if we can set a user's username attribute. Open up `spec/models/user_spec.rb`:

{% highlight ruby %}
# spec/models/user_spec.rb

require 'spec_helper'

describe User do
  it "allows access to username attribute" do
    user = User.new(username: 'seth')
    user.username.should eq('seth')
  end
end
{% endhighlight %}

If we run `rake spec` from the terminal we should see one failing test. Now let's make it pass. In `app/models/user.rb` we need to make our username attribute accesible.

{% highlight ruby %}
# app/models/user.rb

class User < ActiveRecord::Base
  attr_accessible :username
end
{% endhighlight %}

Running `rake spec` again should show us our first test passing with flying colors. Next we need to be able to access password and password_confirmation attributes. Our test goes in the same describe block as before:

{% highlight ruby %}
# spec/models/user_spec.rb

it "allows access to the password and password confirmation attributes" do
  user = User.new(
    username: 'seth',
    password: 'something',
    password_confirmation: 'something'
  )
  user.password.should eq('something')
  user.password_confirmation.should eq('something')
end
{% endhighlight %}

In our user model file (`app/models/user.rb`) we need to add is the aforementioned convenience method, `has_secure_password`. This creates all the password encryption and authentication methods on our user model that we need for creating and logging in users. The only other thing we need is to make the `password` and `password_confirmation` attributes accessible the same way we did for our `username` attribute.

{% highlight ruby %}
# app/models/user.rb

class User < ActiveRecord::Base
  attr_accessible :username, :password, :password_confirmation
  has_secure_password
end
{% endhighlight %}

Running our `rake spec` command again should show our second test passing. Our next few tests will make sure our server validates the presence of the username, password, and password_confirmation attributes upon user creation.

{% highlight ruby %}
# spec/models/user_spec.rb

it "validates presence of username attribute" do
  user = User.new(
    password: 'something',
    password_confirmation: 'something'
  )
  user.save
  user.should have(1).errors_on(:username)
end

it "validates uniqueness of username attribue" do
  user = User.new(
    username: 'seth',
    password: 'something',
    password_confirmation: 'something'
  )
  user.save
  user2 = User.new(
    username: 'seth',
    password: 'something',
    password_confirmation: 'something'
  )
  user2.should have(1).errors_on(:username)
end

it "validates match of password and confirmation attributes" do
  user = User.new(
    username: 'seth',
    password: 'somethingelse',
    password_confirmation: 'something'
  )
  user.save
  user.should have(1).errors_on(:password)
end

it "validates presence of password_confirmation attribute" do
  user = User.new(
    username: 'seth',
    password: 'something'
  )
  user.save
  user.should have(1).errors_on(:password_confirmation)
end
{% endhighlight %}

These four tests will all fail as of now, to make them pass update your user model file with the following:

{% highlight ruby %}
# app/models/user.rb

class User < ActiveRecord::Base
  attr_accessible :username, :password, :password_confirmation
  has_secure_password

  validates_presence_of :username, :password, :password_confirmation
  validates_uniqueness_of :username
end
{% end highlight %}

# Testing User Authentication

We have created our user resource and tested the validation of our attributes, including the match between password and password_confirmation upon creation. The next thing we need to test is that our password is successfully encrypted to password_digest, and that our `authenticate` method works as it should.

{% highlight ruby%}
# spec/models/user_spec.rb

context "with a successfully created user" do
  before(:each) do
    @user = User.create(
      username: 'seth',
      password: 'something',
      password_confirmation: 'something'
    )
  end

  it "encrypts password upon creation" do
    @user.password_digest.should_not be_empty
  end

  it "returns user upon authentication" do
    @logged_in = @user.authenticate("something")
    @logged_in.id.should eq(@user.id)
  end
end
{% endhighlight %}

We actually shouldn't have to make these tests pass. The `has_secure_password` method that Rails provides us takes care of this functionality. Most people would agree that these test may then be unnecessary, but for the sake of this post we are writing them. This is just to give you an idea of how authentication works in Rails and also how to get started with testing your models in RSpec.

# Creating and Testing the Users Controller

It's time to create our Users controller which will serve one purpose: to create and store users. We should already have a `users_controller_spec.rb` file in our `spec` folder, and a `users_controller.rb` file in `app/controllers/`. Open these up, and in our spec file we will write our test.

{% highlight ruby %}
# spec/controllers/users_controller_spec.rb

require "spec_helper"

describe UsersController do

  describe "POST /users" do
    it "should redirect and set flash notice upon successful create" do
      user = {
        username:"seth",
        password:"something",
        password_confirmation:"something"
      }
      post :create, user: user
      response.code.should eq("302")
      flash[:notice].should eq("User created.")
    end

    it "should reset session[:user_id] and display error with incorrect information" do
      session[:user_id] = 1
      user = {
        username: "",
        password: "asdf",
        password_confirmation: ""
      }
      post :create, user: user
      session[:user_id].should be_nil
      response.code.should eq("302")
      flash[:error].should eq("Error creating user.")
    end
  end

end
{% endhighlight %}

We check for response code 200 which means success, and we want to make sure we get a `flash[:notice]` that our user was created.

Now let's write the code to make that test pass.

{% highlight ruby %}
# app/controllers/users_controller.rb

class UsersController < ApplicationController
  def create
    @user = User.new(params[:user])
    if @user.save
      session[:user_id] = @user.id
      flash[:notice] = "User created."
      redirect_to root_url
    else
      session[:user_id] = nil
      flash[:error] = "Error creating user."
      redirect_to root_url
    end
  end
end
{% end highlight %}

You'll notice that we are rendering views that we haven't created yet: `session/show` and `session/new`. In order for rails to issue a response code of 200, those view files need to exist, so let's go ahead and generate the session controller which will create the views for us. We'll also want to clean up a few unwanted files as well. 

{% highlight sh %}
$ rails g controller session new show create destroy
$ rm -rf spec/views/session/ \
         spec/helpers/session_helper_spec.rb \
         app/views/session/create.html.erb \
         app/views/session/destroy.html.erb
{% endhighlight %}

Now we should see passing tests. That's all we need for our users controller, now on to the session controller we just generated.

# Creating and Testing Our Session Controller

Now that we can create users we need to be able to log them in. We'll want to create a session controller for this purpose. We have the actions new, show, create, and destroy.

The `new` action will be our root page which will display a registration form and a login form. Posting to `create` will log in the user upon successful authentication. `show` will be the home view of our application once the user is logged in. Finally, a get request to `destroy` will log out the current user.

Inside our `spec/controllers/session_controller_spec.rb` file we'll start by removing everything inside the main `describe` blocka and placing in our first test.

{% highlight ruby %}
# spec/controllers/session_controller_spec.rb

require 'spec_helper'

describe SessionController do

  describe 'GET /new' do
    it "should respond and contain empty user object" do
      get :new
      response.should be_ok
      assigns(:user).should be_a_new(User)
    end
  end

end
{% endhighlight %}

All we need now is to place an empty `@user` variable in our `new` action. We're doing this because this is where we will be placing our registration form as well as our login form.

{% highlight ruby %}
# app/controllers/session_controller.rb
def new
  @user = User.new
end
{% endhighlight %}

Not much else is needed here, next we'll move on to our `create` action.

{% highlight ruby %}
# spec/controllers/session_controller_spec.rb

describe 'POST /create' do
  before(:each) do
    @user = User.create(
      username:'seth',
      password:'something',
      password_confirmation:'something'
    )
  end

  it "should authenticate, assign flash notice and assign session[:user_id]" do
    post :create, username: 'seth', password: 'something'
    session[:user_id].should eq(@user.id)
    flash[:notice].should eq('Logged in.')
  end

  it "should assign flash error and set session[:user_id] to nil upon incorrect login" do
    post :create, username: 'seth', password: 'somethingelse'
    session[:user_id].should be_nil
    flash[:error].should eq('Error logging in.')
  end
end
{% endhighlight %}

Now let's make the tests pass.

{% highlight ruby %}
# app/controllers/session_controller.rb

class SessionController < ApplicationController
  def new
  end

  def show
  end

  def create
    user = User.find_by_username(params[:username])
    if user && user.authenticate(params[:password])
      session[:user_id] = user.id
      flash[:notice] = "Logged in."
      redirect_to home_url
    else
      session[:user_id] = nil
      flash[:error] = "Error logging in."
      redirect_to root_url
    end
  end

  def destroy
  end
end
{% endhighlight %}

Then we have the show action. We need to it only allow access to the page if a user is logged in.

{% highlight ruby %}
# spec/controllers/session_controller_spec.rb

describe 'GET /show' do
  it "should redirect to /session/new if not logged_in" do
    session[:user_id] = nil
    get :show
    response.code.should eq("302")
    flash[:error].should eq("Not logged in.")
  end

  it "should successfully allow access if logged in" do
    session[:user_id] = 1
    get :show
    response.code.should eq("200")
    flash[:error].should be_nil
  end
end
{% endhighlight %}

And to make them pass..

{% highlight ruby %}
# app/controllers/session_controller.rb

def show
  if session[:user_id].nil?
    flash[:error] = "Not logged in."
    redirect_to "/session/new"
  end
end
{% endhighlight %}

Lastly our destroy action. It should reset the `session[:user_id]` variable and redirect to our login/registration page.

{% highlight ruby %}
# spec/controllers/session_controller_spec.rb

describe 'GET /destroy' do
  it "should reset session[:user_id] and assign a flash notice" do
    session[:user_id] = 1
    get :destroy
    response.code.should eq("302")
    flash[:notice].should eq("Logged out.")
    session[:user_id].should be_nil
  end
end
{% endhighlight %}

The passing code..

{% highlight ruby %}
# app/controllers/session_controller.rb

def destroy
  session[:user_id] = nil
  flash[:notice] = "Logged out."
  redirect_to "/"
end
{% endhighlight %}

# Routing and Views

I'm going to set up a few route configurations in our `config/routes.rb` file. Including setting the root to `session#new`, setting `session#show` to `/home`, and setting `session#create` and `session#destroy` to `/login` and `/logout` respectively.

{% highlight ruby %}
# config/routes.rb

Daylog::Application.routes.draw do
  root :to => "session#new"

  match "home" => "session#show", :as => "home"

  match "login" => "session#create", :as => "login"

  match "logout" => "session#destroy", :as => "logout"

  resources :users
end
{% endhighlight %}

We just need to create a view for our root url that shows a login form and a registration form so we can actually open our app and test it out. This will be located at `app/views/session/new.html.erb`.

{% highlight erb %}
# app/views/session/new.html.erb

<div id="login">
<h2>Login</h2>
<%= form_tag controller: "session", action: "create", method: "post" do %>
  <p>
    <%= label_tag :username, "Username:" %>
    <%= text_field_tag :username %>
  </p>
  <p>
    <%= label_tag :password, "Password:" %>
    <%= password_field_tag :password %>
  </p>
  <%= submit_tag "Login" %>
<% end %>
</div><!-- #login -->

<div id="register">
<h2>Register</h2>
<%= form_for @user, url: { controller: "users", action: "create", method: "post" } do |f| %>
  <p>
    <%= f.label :username, "Username:" %>
    <%= f.text_field :username %>
  </p>
  <p>
    <%= f.label :password, "Password:" %>
    <%= f.password_field :password %>
  </p>
  <p>
    <%= f.label :password_confirmation, "Confirm Password:" %>
    <%= f.password_field :password_confirmation %>
  </p>
  <%= f.submit "Register" %>
<% end %>
</div><!-- #register -->
{% endhighlight %}

We also want to be able to display flash notices and errors, so we will alter our main application view wrapper as follows:

{% highlight erb %}
# app/views/layouts/application.html.erb

<!DOCTYPE html>
<html>
<head>
  <title>DayLog</title>
  <%= stylesheet_link_tag    "application", :media => "all" %>
  <%= javascript_include_tag "application" %>
  <%= csrf_meta_tags %>
</head>
<body>

<% if flash[:error] then %>
  <p class="error"><%= flash[:error] %></p>
<% elsif flash[:notice] then %>
  <p class="notice"><%= flash[:notice] %></p>
<% end %>

<%= yield %>

</body>
</html>
{% endhighlight %}

# Conclusion

We should be able to start our application with `rails s` and see our root page displaying a login form and a registration form. We should also see notifications and error messages at the top as they are assigned. All our test should be passing and we should be ready to move on to the next phase of our application.

Next I will be writing a post outlining how to test that our application is serving JSON data correctly for our `home` view. That is the page that will house the rest of our application once we are logged in.

If you would like to download or reference the source of this application, please visit the [github page](http://www.github.com/sethetter/daylog). Note that it will always show the most recent version of the application.

Thanks for taking the time to read, if you have any feedback, questions, or comments, please feel free to contact me or leave a message below. :)
