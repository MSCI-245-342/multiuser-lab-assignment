# User Authentication and Authorization

Note: The material in this self-guided lab and assignment is based on Chapters 6, 7, 8, and 10 of "Ruby on Rails Tutorial, 6th Edition" by Hartl.  Portions of code are used from Hartl's tutorial.  Students can view the book by [viewing the book's page in the university library catalog](https://ocul-wtl.primo.exlibrisgroup.com/permalink/01OCUL_WTL/156lh75/cdi_safari_books_9780136702726) and clicking on the "O'Reilly for Higher Education" link under "View Online".

Objective: Refresh Ruby on Rails knowledge by showing how to add simple user authentication and authorization to a web app.

## Introduction

The vast majority of our web applications have a means for individual to create an account (sign_up), be authenticated (sign_in), and be authorized to perform certain actions based on who they are.  Authentication is determining that you are a certain person, for example by providing an email address and password to prove your identity.  Authorization is the act of allowing an authentication person to perform actions with the application.  For example, some people may be considered administrators of the system and be authorized to perform actions that regular users are not.

### Getting Rails up and running

You are reading these directions having cloned the repository to a directory named `multiuser`.  First:

```
cd multiuser
```

and then do:

```
rails new . --skip-action-mailbox --skip-action-text --skip-active-storage --skip-action-cable --skip-spring --skip-javascript --skip-turbolinks --database=postgresql --skip-bundle 
```

to create your rails install.  In the Codio setup for MSCI 245 and MSCI 342, we do not have a good way to test web apps built using javascript.  Thus, both `--skip-javascript` and `--skip-turbolinks` are used to disable parts of Rails that depend on javascript.  The other "skip" options are to exclude parts of Rails you are unlikely to need to use on the course project and which simply clutter up your app and make working on Rails harder. 

In contrast to MSCI 245, the above Rails app will be setup to help you generate automated tests using [Minitest](https://github.com/seattlerb/minitest), which is the default testing framework for Rails.  The Engineering Software as a Service (ESaaS) textbook instead encourages the use of Cucumber and RSpec.  Minitest is much simpler to understand, and by using it instead of Cucumber and RSpec, we'll have lots less to learn to be able to do the same amount of work.  

I've tried learning Cucumber and RSpec, and the effort drove me mad.  I am solidly behind the creator of Rails, [DHH](https://twitter.com/dhh), and his decision to [choose Minitest over RSpec](https://twitter.com/dhh/status/52807321499340800?lang=en) and to consider Cucumber as a non-programmer tool: ["Don’t use Cucumber unless you live in the magic kingdom of non-programmers-writing-tests (and send me a bottle of fairy dust if you’re there!)"](https://signalvnoise.com/posts/3159-testing-like-the-tsa) 

Then, open up the `Gemfile` and comment out the `jbuilder` and `tzinfo-data` gems:

```
# Build JSON APIs with ease. Read more: https://github.com/rails/jbuilder
#gem 'jbuilder', '~> 2.7'

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
#gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```

and uncomment the `bcrypt` gem, which is used for encrypting passwords:

```
gem 'bcrypt', '~> 3.1.7'
```

Then:

```
 bundle install
```

Add the following to `config/environments/development.rb` after "`Rails.application.configure do`":

```
config.hosts << /[a-z0-9]+\-[a-z0-9]+\-3000\.codio\.io/
```

For user authentication on the web to be secure, we have to make sure users only connect to our server with SSL.  We do this by editing `config/environments/production.rb` and uncommenting the setting of `config.force_ssl` to `true`:

```ruby
config.force_ssl = true
```

If a user does not use SSL, it is simple for an attacker to listen to the unencrypted communications and steal the session token stored in a user's browser cookie and then be authenticated as that user.

Now, need to adjust some things to make testing work better for us on
Codio in `test/test_helper.rb`.  In this file, comment out:

```
# parallelize(workers: :number_of_processors)
```

By commenting out `parallelize(workers: :number_of_processors)`, this will force our automated tests to run 1 at a time.  This is very handy if we want to use the byebug debugger to debug tests or see how testing works.  If you aren't going to debug while testing, you are welcome to leave parallelization of tests turned on.

In `test/test_helper.rb` also comment out:

```
#fixtures :all
```

For now, the use of fixtures, for testing, complicates our work, and we'll avoid using them.

In `test/application_system_test_case.rb`, comment out: 

```ruby
driven_by :selenium, using: :chrome, screen_size: [1400, 1400]
```

and add:

```ruby
driven_by :rack_test
```

So, you'll end up with:
```
#driven_by :selenium, using: :chrome, screen_size: [1400, 1400]
driven_by :rack_test
```

The `:rack_test` driver cannot process javascript, but it runs very quickly.  The issue with using `:selenium` and Chrome as our system test driver is that I haven't been able to get these components to install correctly on Codio.  Projects in MSCI 342 should not use javascript, and so this will not be an issue.

Then do:

```
rails db:create db:migrate
```

This will create your databases.  

Then run `rails server -b 0.0.0.0` and go to your "Box URL".  Make you get "Yay! You're on Rails!".  This is a good time to commit and push to GitHub.

## Users

Sometimes it can be super nice to use Rails scaffolding and create a bunch of code quickly that we then go and customize to our needs.  Users are basically the same as a resource, and the basic scaffolding makes sense except we'll want to modify the routes a bit for naming convienence.

```
rails generate scaffold User name:string email:string
```

In addition to the files you are used to seeing made for a scaffold, with the test framework turned on, we get some additional files made for us:

```
create      test/models/user_test.rb
create      test/fixtures/users.yml
create      test/controllers/users_controller_test.rb
create      test/system/users_test.rb      
```

The `test/fixtures/users.yml` file is the automatically generated fixtures for the User model.  You'll almost always need to edit this file if you use fixtures.  We're going to ignore it.

Rails produces tests for the model, the controller, and the "system" that are appropriate for testing the scaffolding code it produced for us.  

In testing terms, the model tests (`test/models/user_test.rb`) are unit tests, i.e. tests for the User model object only.  This is the lowest level of testing.  

The controller test works by sending HTTP requests directly to the app and allows us to test the controller without relying on a user interface to exist for input of data.  The controller will still render any views, but we don't use the app's user interface for input. The controller tests are essentially the same thing as what might be called "integration" tests, i.e. they test how several objects work together. These tests are good for testing the actions of a single controller, e.g. creating, reading, updating, and deleting resources.

The system tests, operate by simulating user clicks and input into the user interface, and these are what are typically called "end to end" tests (E2E tests).  These tests are good for testing a complete workflow in an application.

The tests as they currently exist would fail in many cases because they rely on the fixtures, which we've disabled.

Let's fix up the User model in first.  We know that we want some help from the database to maintain data integrity, and so we first go to `db/migrate` and open up the migration for the users table (will be named with a timestamp and create_user.rb).

We want to limit the size of names to something reasonable such as 70 characters and prevent nulls.  We also want a length limit for the email, no nulls, and we want each user to have a unique email address.  No two users should have the same email, and so we'll add an index on email to enforce uniqueness at the database level.  Make your migration look like:

```ruby
class CreateUsers < ActiveRecord::Migration[6.0]
  def change
    create_table :users do |t|
      t.string :name,  null: false, limit: 70 
      t.string :email, null: false, limit: 255 

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

Update the database:
```
rails db:migrate
```

and if you'd like to see the inside of the database:

```
psql -d multiuser_development
```

and then inside of psql, `\d` to see the tables and `\d users` to see the table created.  To exit psql, `\q`.  

With the database in good shape, we need to add validations to the User model to match.  Edit `app/models/user.rb` to be:

```ruby
class User < ApplicationRecord
    validates_presence_of :name
    validates_length_of :name, maximum: 70
    
    before_save { self.email = email.downcase }    
    validates_presence_of :email
    validates_length_of :email, maximum: 255    
    validates_uniqueness_of :email, case_sensitive: false    
    validates_format_of :email, with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i      
end
```

Recall that `validates_presence_of` prevents empty strings.  The other validations are self-evident.  The `before_save` is an [Active Record Callback](https://guides.rubyonrails.org/active_record_callbacks.html) that executes the code to downcase (lowercase) the email before storing a User object to the database.  This allows our database to enforce email uniqueness.

Now, you might trust me to have written these validations correctly, but if you were to write them youself, you should consider testing the tricky ones, especially the email format validation.  

### Super Brief Intro to MiniTest

We can do that with unit tests on the User model.  At the moment, `test\models\user_test.rb` is a nearly empty skeleton:

```ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```

Go ahead and delete the commented material from the file.

We write tests in MiniTest by simply create a ruby method that starts with the name "test".  For example, let's say that I wanted to test that I cannot create and save two User objects with the same email, I could add the following method to the UserTest object (go ahead and do this):

```ruby
def test_email_must_be_unique
    user1 = User.new(name: "Example User", 
       email: "user@example.com" )
    user1.save! # if fails, an error is raised
    user2 = User.new(name: "Example User", 
       email: "user@Example.com" ) # also check downcase
    assert_not user2.valid?     
end
```

We can run all the tests in a given file as follows:

```
rails test --verbose test/models/user_test.rb
```

You should see output similar to:

```
Run options: --verbose --seed 32634

# Running:

UserTest#test_email_must_be_unique = 0.02 s = .

Finished in 0.018030s, 55.4646 runs/s, 55.4646 assertions/s.
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

where `UserTest#test_email_must_be_unique = 0.02 s = .` is in the color green, which means the test passed.  

What Rails and MiniTest are doing is loading the whole web app in Rails, and then running each method that starts with the word "test" in the UserTest class.  This is how we can access the User model and create a new user and save it to the database.  The `assert_not` is one of [many different assert statements available in MiniTest](http://docs.seattlerb.org/minitest/Minitest/Assertions.html).  The `assert_not` means that `user2.valid?` should not evaluate to true.  If `user2.valid?` did evaluate to true, the assertion would fail.  When any assertion in a test fails, the test fails.  If an error is raised, for example by the call to user1.save!, then a test also fails.

To see this behavior, change either user1's or user2's email to be different and run the test again.  You should see now that the test fails.  Before continuing, put the test back as it was and check that it now passes.

Did you notice that we can run this test over and over again and not have problems with user1 already being saved to the database? The testing framework in Rails runs each test in a transaction and then rolls it back at the end of the test so that the database goes back to the same state as it was at the start of the test.  In addition, it is the multiuser_test database that is used by testing and not the multiuser_development database.  It is common for us to use the development database for debugging, and we may leave it filled with all sorts of strange data, which could affect the running of tests.  When we run tests, we want to be absolutely certain of the state of the system, and a separate testing database helps with this goal.

We will talk much more about testing in MSCI 342.  For now, it is good to know that it exists and isn't that hard to use.

## Routes

We can run `rails routes` to see the current routes:

```
   Prefix Verb   URI Pattern               Controller#Action
    users GET    /users(.:format)          users#index
          POST   /users(.:format)          users#create
 new_user GET    /users/new(.:format)      users#new
edit_user GET    /users/:id/edit(.:format) users#edit
     user GET    /users/:id(.:format)      users#show
          PATCH  /users/:id(.:format)      users#update
          PUT    /users/:id(.:format)      users#update
          DELETE /users/:id(.:format)      users#destroy
```
Instead of `/users/new`, it would be nice to have the route be `/signup`.  If we look in `config/routes.rb`, we'll see:

```
resources :users
```

which produced all of those routes for us.  Replace `resources :users` with:

```
  get '/signup', to: 'users#new'    
  resources :users, except: [:new]  
```

run `rails routes` to see that we now use the users#new controller action for GET /signup instead of GET /users/new.

### A Better Homepage

Let's quickly put into place a better homepage.

```
rails generate controller static_pages home
```

add to routes.rb:

```
root 'static_pages#home'
```

Edit `app/views/static_pages/home.html.erb` to have YOUR NAME (not mine) and a link to the signup page:

```html
<h1>Welcome!</h1>

<p>My name is Mark Smucker.</p>

<p><%= link_to 'Signup for an account', signup_path %></p>
```

run `rails server -b 0.0.0.0` and verify that it works.

## Passwords

Our User model lacks a password.  When we create a new user, we want the user to supply a password that we can use when they want to login to verify that they are who they say they are.

Never ever store passwords in a database.  

When a user types in a password that we can read, that is called a "plaintext" password.  If we store the plaintext password in the database, any employee or any hacker with access to the database can copy the password and use it to gain access to the user's account.  Instead of storing the plaintext password, we store a cryptographic hash of the password in the database.  It is computationally infeasible to determine a plaintext password from the hashed password.  Only the original plaintext password can be used by a user to login, and thus the passwords are safe when stored in this hashed form.

When a user shows up to login, we take their plaintext password and hash it and compare the hash version to the version stored in the database.  If they match, we know the user used the same plaintext password as before, and we have authenticated them.

This is such an important thing for web apps, that Rails provides a special method for ActiveRecord models to handle the password.

In `app/models/user.rb` add:

```ruby
has_secure_password
```

this adds two virtual fields to the User model: `password` and `password_confirmation`.

In the database, we need to add a single "password_digest" column to the users table to hold the hashed password:

```
rails generate migration add_password_digest_to_users password_digest:string
rails db:migrate
```

If you'd like, you can verify using psql that a column has been added to the users table.

It is nice to validate that people don't use a blank password and that passwords have a minimum length.  Add the following validations to your User model in `app/models/user.rb`:

```
validates_presence_of :password
validates_length_of :password, minimum: 5
```

Now that the User model has virtual fields `password` and `password_confirmation`, we need to modify the view to allow users to specify these fields when creating an account.  Modify `app/views/users/_form.html.erb` to have this additional code:

```html
  <div class="field">
    <%= form.label :password %>
    <%= form.password_field :password %>
  </div>

  <div class="field">
    <%= form.label :password_confirmation %>
    <%= form.password_field :password_confirmation %>
  </div>
```

Notice the use of `password_field` and not `text_field`.

Now, we need to modify the users controller to accept the new input fields.  

In `app/controllers/users_controller.rb` modify the `user_params` method to also permit the fields `:password` and `:password_confirmation`:

```ruby
def user_params
   params.require(:user).permit(:name, :email, :password,
                                :password_confirmation)
end    
```    

Run your server, and see that you can create a user.  Take a look in the database to see the password digest:

```
psql -d multiuser_development
```
and then
```
select * from users ;
```

Now would be a good time to commit your changes and push to Github.

## Sessions

After a user has created an account, we want to store an identifier securely in their web browser so that we can identify them on every request they make to our web app.

If you recall from MSCI 245, Rails has a [`session` variable that works as a hash](https://guides.rubyonrails.org/action_controller_overview.html#session).  For example, when a user logs in, we can save their user id into the session, which Rails then put into their browser via an encrypted cookie.  Assuming `user` holds our User object:

```
session[:user_id] = user.id
```

Whenever we interact with a user, we can always check the session to see if the user is logged in or not.  If there is not a value for `session[:user_id]`, then we know the user is not logged in.  If there is a value, then we have the logged in user's id and can fetch the proper User object from the database or check if a user is authorized for the currrent page or action, etc.

When a user closes their web browser, the session cookie is destroyed and they will need to login again.  For MSCI 342, this seems fine to me.  If you want to handle permanent logins, you should read Chapter 9 of the Ruby on Rails Tutorial as well as likely switch to using a prepackaged solution such as the [devise](https://github.com/heartcombo/devise) gem.

To manage sessions, we're going to closely follow the methods of Hartl in Chapter 8 of the Ruby on Rails Tutorial, 6th Edition, and portions of this code are copied from the book.

We want to support login and logout, and to do so, we will create a sessions controller, routes, and views to support these capabilities.

First, in `config\routes.rb` add:

```ruby
  get    '/login',  to: 'sessions#new'    
  post   '/login',  to: 'sessions#create'    
  get    '/logout', to: 'sessions#destroy' 
```

From the above routes, you can see that we want our sessions_controller to support the actions: new, create, and destroy.  Only the `new` action needs a view to display a login form with email and password.  Thus, we'll generate a Sessions controller with an action of `new` and then hand edit the controller to add the other actions:

```
rails generate controller Sessions new
```

To determine what will be passed into the controller when a user submits their login information, lets edit the view in `app/views/sessions/new.html.erb` first.  Delete what is in there and paste in this:

```erb
<h1>Login</h1>
<%# Code copied and modified from Hartl's Ruby on Rails Tutorial. %>

<%= form_with(url: login_path, local: true) do |f| %>
    <%= f.label :email %>
    <%= f.email_field :email %>

    <%= f.label :password %>
    <%= f.password_field :password %>

    <%= f.submit "Login" %>
  <% end %>

 <p><%= link_to "Create Account", signup_path %></p>
 ```

 This use of [form_with](https://api.rubyonrails.org/v6.1.0/classes/ActionView/Helpers/FormHelper.html#method-i-form_with) introduces a new feature: the `url:` option.  The `url:` option says that the form should be POSTed to `login_path`.  

 This is a basic version of the view that we need, but it is missing any code to display login attempts that fail.  Let's use the [flash](https://api.rubyonrails.org/classes/ActionDispatch/Flash.html) mechanism and add code to our application layout to always display flash messages on any page.

 Edit `app/views/layouts/application.html.erb` to have inside the html `<body></body>` this code:

 ```erb
  <body>
    <%# Modified from: https://stackoverflow.com/questions/9390778/best-practice-method-of-displaying-flash-messages %>
    
    <% flash.each do |key, value| %>
      <%= content_tag :div, value, id: "#{key}" %>
    <% end %>      
      
    <%= yield %>
  </body>
 ```

Now, we can use the `flash` as part of our solution.

### Intellectual Property and ethics

In the above code snippet, you will see that I've added an erb comment to give proper credit to the source of the code.  The author of source code owns it.  If someone else writes something, you do not automatically have a right to copy it or reuse it for your own purposes.  

It is great that you live and work in Canada where smart people like you have their intellectual property protected.  If someone copies your work outside of what is allowed by law, you can persue legal action against them.  Likewise, if you illegally copy other people's work, they can come after you.

Thankfully, all of the code in Hartl's tutorial is licensed for use by its readers under the [MIT License](https://opensource.org/licenses/MIT), and since we legally have a right to the code via our library, we can use the code and modify it.  If we use a "substantial portion" of the software, we need to include the full License.

In MSCI 342, I expect all code copied from the web to include a reference, usually a URL, that cites where the code came from.  In most cases, it is clear that the authors have provided the code to help you, and they expect you to use it in some fashion.  For example, the small code snippets that people provide on stackexchange are obviously there to help you.  You are unlikely to use those snippets without modifying them substantially, and thus creating your own new code.  **No matter what, you must cite the original source in your code.**  By citing your sources, you show that you are not attempting to steal without credit the work of others.  Also, this provides useful documentation of the code.  It can be super helpful to other developers to be able to read the original web page where you found a solution.

If you copy code without substantial modification, you need to make sure you have the right to do so.  Thankfully, most code you will have access to in this form are open-source projects, and most of them license their code for other open-source or personal, non-profit uses.

When working for a company, be aware that not all code on the internet comes with a license that permits its use for commerical purposes!  In most cases, people who create intellectual property, such as software, want to be paid for their work if someone else is going to make money using it!  
If code has no license notice on it, it is still copyrighted automatically by the original author, and you do not have the right to copy it.

In MSCI 342, I expect that either you will be using a small portion of a larger body of code, and thus likely falling under "fair use" rules, or the code has a license that allows your use of it.  Again, **no matter what, you must cite the source of code and ideas for code if they came from someone else**.

#### Sessions Controller

Let's work on the controller in `app/controllers/sessions_controller.rb` first.  Replace the code in the file with this code:

```ruby
# This code is copied and modified from Hartl's 
# Ruby on Rails Tutorial, 6th edition.
# 
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:email].downcase)
    if user && user.authenticate(params[:password])
      log_in user
      redirect_to root_url, notice: "Logged in."
    else
      flash.now[:notice] = 'Cannot log you in. Invalid email or password.'
      render 'new'
    end
  end

  def destroy
    log_out
    redirect_to root_url, notice: 'Logged out.'
  end
end
```

Nothing needs to go in the `new` method because the only thing it does is render the `new` view (the login page).  In create, we are handling the POST to the login_path.  The first line:

```ruby
user = User.find_by(email: params[:email].downcase)
```

simply uses the User model to search the database for a user with the submitted email address.  If found, `user` will hold the User object for that user.  If not found, `user` will be nil.

Next, if we have a user object, we then check to see if they gave us the correct password.  The `authenticate` method was automatically created for us when we added `has_secure_password` to the User model.  

If the password is correct, `authenticate` returns the user object, otherwise it returns false.  

So, if we get a correct email and password combination, we then want to "login" the user.  Recall that to login the user, we only need to store the user.id into the user's session object:

```ruby
 session[:user_id] = user.id
 ```

If it is so simple, why create a method named `log_in` to handle it?  We do this to hide the implementation of logging a user in.  You may well want to login a user in some other way in the future, and if you hard code `session[:user_id] = user.id`, you have to find it everywhere in your code to make the change.  Using methods to abstract away and hide the details of implementation is perhaps the most important software design technique.  

Where should we put the `log_in` method?  Hartl makes the design choice to create a set of methods in a `module` for logging in, logging out, checking if a web request comes from a logged in user or not, and returning the current logged in user.  

#### About Modules

A ruby module is a set of methods and instance variables that can be included into any class.  Hartl puts these session related methods and variables into the `app/helpers/sessions_helper.rb` module.  (The variables are not shared across classes.  Each class gets it own instance variables.)

He then includes the module into the ApplicationController, from which all other controllers inherit.  Thus, every controller that exists in the system will have the various session authentication methods available.  This makes sense because potentially every request to our web app needs to check for who the user is, and if the user is not logged in, to redirect them to the login page, or at least refuse to serve content to them.  If this functionality wasn't needed by almost all controllers, a better design would have placed the methods and variables into a class that could be instantiated when needed.  Such a class could then be used or passed around where needed.  As implemented by Hartl, the methods he places in the module are not available to models or other classes unless they also include the module.

#### SessionsHelper module

The SessionsHelper module is found in `app/helpers/sessions_helper.rb`.  Hartl's design uses an instance variable, `@current_user` to store the User object we get from the database.  Rather than access @current_user directly, we'll have a method named `current_user` and call it when we want the current user.  On the first call to `current_use`, if the user is logged in, but `@current_user` is nil, we fetch the user from the databae and store the value in `@current_user`.  If `@current_user` has a value, we just return it.  When we want to fetch the current user from the database, we should only do it once.  We shouldn't repeatedly query the database for the same information.

Here is my SessionsHelper, which is based heavily on Hartl's, (copy and paste into `app/helpers/sessions_helper.rb`):

```ruby
module SessionsHelper
  # Code based on Hartl's Ruby on Rails tutorial, 6th ed.
    
  def log_in user
    session[:user_id] = user.id
    @current_user = nil
  end

  def logged_in?
     # You may have a session[:user_id], but if you don't
     # also have an entry in the database, we cannot say
     # you are logged in, for you don't exist!  This could
     # happen if an administrator deleted your account 
     # while logged_in.  
     !current_user.nil?
  end    
    
  def current_user
     if !session[:user_id].nil?
        if @current_user.nil?
            # if id not in DB, find_by returns nil
            @current_user = User.find_by(id: session[:user_id])
        end
     else
        @current_user = nil
     end
     return @current_user 
  end
    
  def log_out
      session.delete(:user_id)
      @current_user = nil
  end
    
end
```

The method `log_in` is simple.  We store the `user.id` in the `session` with the key `:user_id`.  What this does is set a session cookie in the user's browser that we can use to know the user has been authenticated on future requests to the web app.  When the user closes their browser, the session cookie will be deleted.

Checking if the user is logged in, is tricky.  When I first read Hartl's code, it seemed strange to me that he called `current_user` instead of just checking the `session` object for a user_id.  As you can see by my comment above, I eventually figured it out, and documented this design decision to prevent future developers from thinking that, as I did, that `logged_in?` shouldn't hit the database and "improving" the code.

For logging out, we delete the `:user_id` key and its value from the `session` hash.  It would be semantically incorrect to simply set the value to nil, for that is not a valid user id.  

We include the SessionsHelper in all controllers by including it in `app/controllers/application_controller.rb`:

```ruby
class ApplicationController < ActionController::Base
    include SessionsHelper
end
```

### Cleanup

Now that we have a means for users to sign up and login, we should do some cleanup of the app.  

First, when users sign up and create a new account, we should also log them in.  It is annoying to create an account and then have to log in separately.

It is in the `UsersController` that a new user is created.  If the new user is successfully saved to the database, before we redirect the user, we should log them in.  I also think it makes more sense to send them back to the home page rather than to their user page.  So, we change `UsersController#create` to be:

```ruby
  def create
    @user = User.new(user_params)

    if @user.save
      log_in @user    
      redirect_to root_url, notice: 'Account created and logged in.'
    else
      render :new
    end
  end
```

Let's edit the links on the users edit view (`app/views/users/edit.html.erb`) to only have a link back to the home page.  So we change it to be:

```erb
<h1>Editing User Account</h1>
<%= render 'form', user: @user %>
<%= link_to 'Home', root_path %>
```

Same for `app/views/users/new.html.erb`:

```erb
<h1>Create New User Account</h1>
<%= render 'form', user: @user %>
<%= link_to 'Home', root_path %>
```

When viewing a user (`app/views/users/show.html.erb`), lets have links to all users and root.  We'll also get rid of the flash notice because we added them to our application layout (make your copy match):

```erb
<h1>User Account Details</h1>
<p>
  <strong>Name:</strong>
  <%= @user.name %>
</p>

<p>
  <strong>Email:</strong>
  <%= @user.email %>
</p>

<%= link_to 'Home', root_path %> |
<%= link_to 'List All Users', users_path %>
```
For the list of users (`app/views/users/index.html.erb`), we'll also remove the flash notice so that it isn't duplicated.  Also, the link to destroy (delete) a user won't work for us because it relies on javascript, and so we'll change it to a form submit.  By the way, there are ways to use CSS to style a submit button as a link if needed.  Do not turn the destroy action into a GET request!  Destructive actions should not be done via GET requests. Make your `index.html.erb` match the below:

```erb
<h1>Users</h1>

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Email</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @users.each do |user| %>
      <tr>
        <td><%= user.name %></td>
        <td><%= user.email %></td>
        <td><%= link_to 'Show', user %></td>
        <td><%= link_to 'Edit', edit_user_path(user) %></td>
        <td><%= form_with model: user, method: :delete, local: true do |f| %>
               <%= f.submit 'Delete' %>          
            <% end %> 
        </td>
      </tr>
    <% end %>
  </tbody>
</table>

<br>
<%= link_to 'Home', root_path %>
```

Finally, let's update the home page with links to appropriate functions.  Let's say our app is all about maintaining a list of users, and only authenticated users should see other users.  You have to join to be part of the club. In addition, if a user is logged in, we should provide a logout link and not show the login or signup links, and we can do that with a simple if statement and the logged_in? method that is now available in all controllers (`app/views/static_pages/home.html.erb`):

```erb
<h1>Welcome!</h1>
<p>My name is Your First and Last Name Here.</p>

<% if logged_in? %>
  <p><%= link_to 'List All Users', users_path %></p>
  <p><%= link_to 'Logout', logout_path %></p>
<% else %>
  <p><%= link_to 'Login', login_path %></p>
  <p><%= link_to 'Signup for an account', signup_path %></p>
<% end %>
```

Go ahead and run your app with `rails server -b 0.0.0.0` and give it a spin and see that it works as expected.  Now is another good time to commit and push to GitHub.

## Authorization

Our app has authentication, but it allows all users to do everything on the site!  

Go ahead and try to directly go to `/users` (by typing in the url in your browser) when you are logged out.  You can do it.  Just because we hide a link on the homepage from people not logged in, doesn't mean the link doesn't exist!  

Following Chapter 10 in Hartl's tutorial, we'll implement authorization at the controller level.  Remember that we build our web apps with the idea that HTTP requests come in (GET, POST, PATCH, DELETE) to a given path and then an appropriate controller action handles that request.  It makes sense to control access at this request level.  **Never rely on your user interface in a web app to be your security.**  Web apps respond to HTTP requests, and anyone in the world can send any HTTP request to your app without ever using your user interface.  (If you know the appropriate HTTP requests for a website, you could craft your own user interface for it.Indeed, this is a model that some web apps have gone to.  They provide an API that sends back JSON in response to HTTP requests, and then people write the UI layer with a javascript framework.)

For this app, we want only users to be able to edit and delete their own accounts, but all users can see all accounts.  People who are not logged in, should only be able to view the home page, sign up and login.

### Making sure user is logged in

First, we will add a new method to the SessionsHelper module that can be used to require a user to be logged in.  Add to `app/helpers/sessions_helper.rb`:

```ruby
  # Code based on https://guides.rubyonrails.org/action_controller_overview.html#filters
  def require_login
    unless logged_in?
      flash[:notice] = "Please log in."
      redirect_to login_url 
    end
  end
```

We'll use a `before_action` filter to check that a user is logged in before allowing actions to run.  Because the default behavior should require logins, we'll use the filter to only mark the actions that will not require being logged in.  In this way, we are less likely to make a mistake and forget to require a login on an action.

In `app/controllers/users_controller.rb`, add inside the definition of UsersController this filter as the first before_action filter:

```ruby
before_action :require_login, except: [:new, :create]
```

Thus, the top of your UsersController should look like:

```ruby
class UsersController < ApplicationController
    before_action :require_login, except: [:new, :create]
    before_action :set_user, only: [:show, :edit, :update, :destroy]
```

Filters are run in the order that you define them.  There is no reason to run `set_user`, which hits the database, if we're going to redirect a request because the user is not logged in.

Go run your server and test out that things work as expected.  If you aren't logged in, you should only be allowed to create a new user.  

Now, we want to make sure users are allowed to delete or edit their own accounts and not the accounts of other users.  These are the `edit`, `update`, and `destroy` actions.  So, we'll create another before_action filter that checks if a user is authorized to make these changes.  Add to UsersController **after** the other two before_action filters:

```ruby
before_action :authorized_to_modify_and_destroy, only: [:edit, :update, :delete]
```

so now UsersController looks like this at the top:

```ruby
class UsersController < ApplicationController
  before_action :require_login, except: [:new, :create]
  before_action :set_user, only: [:show, :edit, :update, :destroy]
  before_action :authorized_to_modify_and_destroy, only: [:edit, :update, :destroy]
```
and then this new private method:

```ruby
def authorized_to_modify_and_destroy
   if current_user != @user
      redirect_to users_url, notice: "You can only edit or delete your own account."
   end
end
```

We want to be sure to place the `authorized_to_modify_and_destroy` filter after `set_user` so that `@user` has been set!

The comparison `current_user != @user` is on ActiveRecord objects, and this comparison checks that the two objects have the same `id`.  Note that `current_user` should never be `nil`, for the user has to be logged in at this point, but a comparison to `nil` will also return `true`.  Because `set_user` retrieves the User object using `find`, if for some reason someone tries to modify a user that doesn't exist in the database, an error will be thrown by `find` and execution will stop.

#### Deleting you own account

If you delete your own account, you should not stay logged in.

Change the `UsersController#destroy` method to log you out if you've deleted your account:

```ruby
  def destroy
    if @user == current_user
        log_out
    end
    @user.destroy
    redirect_to users_url, notice: 'User was successfully destroyed.'
  end
```

### Deploy to Heroku

Let's deploy your app to Heroku.  Recall how we do this:

1. Log in to your Heroku account by typing the command: `heroku login -i` in the terminal. This will connect you to your Heroku account.

2. While in the root directory of your app (the multiuser directory), type `heroku apps:create multiuser-watiamUsername` to create a new project in Heroku. Replace "watiamUsername" with your WatIAM username. This will tell the Heroku service to prepare for some incoming code, and locally it will add a remote git repository for you called heroku.  If you have too many Heroku apps, use Heroku's [heroku apps:destroy](https://devcenter.heroku.com/articles/heroku-cli-commands#heroku-apps-destroy) command to get rid of one.

3. Before doing anything else, record the URL that Heroku provides telling you where to access your app on the web.

4. In the multiuser directory, create a file named `Procfile` and put this command in the file:
```
bundle exec puma -C config/puma.rb
```
5. Make sure you have committed all changes to git.  Run `git status` to make sure there are no outstanding changes to commit.

6. git push heroku main

7. Go verify that your Heroku deployment works.

### Update the README.md

Following this template, update your app's README.md file:

```md
# Multiuser

Author: Your Name

Heroku URL of deployed web app: http://replace-with-your-heroku-app-hostname/

Notes: Any notes to TA or instructor or notes for yourself.
```

Commit and push to GitHub.

## Finishing up

1. Verify that everything seems to work.  
2. Make sure with `git status` that there isn't anything left to commit and push to GitHub.  If there is, commit and push.
3. Go check GitHub to verify you see you whole project committed.  Because the README.md was the last thing to get added, if you see it, you should probably be good.

That's it.  The purpose of this assignment was to show you how we can do authentication and authorization in our apps, and to get you back in the Ruby on Rails groove.  

More details can be found in books such as Hartl's Ruby on Rails Tutorial.  If you need to craft a fancier auth system, you should use a gem such as [devise](https://github.com/heartcombo/devise).  I don't think your MSCI 342 project will require a fancier auth system.  


























