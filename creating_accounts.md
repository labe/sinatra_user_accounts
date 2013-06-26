##User accounts with Sinatra (no BCrypt)

### Create migration

A very basic schema for a user typically looks like this:

```ruby
def change
  create_table :users do |t|
    t.string  :username
    t.string  :email
    t.string  :password
    t.timestamps
  end
end
```
Users can have plenty of other information about them or their account stored in their table, of course-- <code>full_name</code>, <code>hometown</code>, <code>is_private</code>. If ever stumped about what to include in the schema, think of apps you use and the information included in your profile.

### Make validations on the model

```ruby
class User < ActiveRecord::Base
  validates :username, :presence => true, 
                       :uniqueness => true
  validates :email,    :presence => true,
                       :uniqueness => true
                       :format => { :with => /\w+@\w+\.\w+/ }
  validates :password, :presence => true
end
```
Validations mean a User object won't ever be saved unless they meet this criteria. The above code says that a User cannot exist in the database unless it has 

* a username (which is unique)
* an email address (which is unique and is formatted based on a RegEx (this is a very loose RegEx example that says "at least one word character (letter, number, underscore) followed by an "@", followed by at least one word character followed by a ".", followed by at least one word character"))
* a password

###Make the routes

Routes will depend on your UX design. Should login and sign up be on separate pages? If creating an account requires a lot of fields, each should probably have its own page:

```ruby
get '/login' do
  erb :login
end

get '/new_account' do
  erb :new_account
end
```

If creating an account is not much different from logging in, however, it may make more sense to consolidate both forms on the same page.

```
get '/login' do
  erb :login
end
```

Regardless, each form will need to POST to its own route.

```ruby
post '/login' do
  # stuff we'll come back to soon
end

post '/new_account' do
  # stuff we'll come back to soon
end
```
And of course you'll need:

```ruby
get '/logout' do
  session[:user_id] = nil
  redirect_to '/'
end
```
Might your users also have a profile page? Maybe something like:

```ruby
get '/users/:username' do
  erb :user
end
```

### Make the current_user helper method

The Sinatra skeleton contains a "helpers" folder under "app". Methods saved in this folder are accessible from all your controllers and views.

Create a file called <code>user_helper.rb</code> (or something akin to that), and create a method <code>current_user</code>:

```ruby
def current_user
  User.find(session[:user_id]) if session[:user_id]
end
```
If a user has signed in and <code>session[:user_id]</code> has been set, calling this method will return the relevant User object. Else, the method will return <code>nil</code>.

There are a few other ways to write code that produces the same results. This one is the simplest.

### Link to the views

Users should *probably* be able to sign in or logout or access their profile page at any time, regardless of which page they're on, right? (Probably right.) Put those babies in a header above your <code><%= yield %></code> statement in your <code>layout.erb</code> view.

```ruby
<body>
  <div class="header">
    <h5><a href="/">Home</a></h5>
    <% if current_user %>
      <h5><a href='/logout'>Logout</a></h5>
      <h5>Signed in as: <a href="/users/<%= current_user.name %>"><%= current_user.name %></a></h5>
    <% else %>
      <h5><a href="/login">Login</a></h5>
    <% end %>
  </div>
  <div class="container">
      <%= yield %>
  </div>
</body>
```
The above specifies that if the <code>current_user</code> helper method returns true (meaning, a user has logged in):

* display a "Logout" link
* display that they are "Signed in as [username]" (and the username links to their profile page)

Otherwise, just display a link to the login page (and to the "create an account" page, if you've chosen to separate them).

### Make the views

You know how to create forms, so I won't belabor the point here. Things to remember:

* When naming fields, match them to the database table column names (e.g.: <code>username</code>,<code>email</code>,<code>password</code>)
* Get fancy and put them in a hash
  * <code>user[username]</code>,<code>user[email]</code>,<code>user[password]</code> will POST:
	*  <code> params => {user => {username: < username >, email: < email >, password: < password > }</code>
	* **NOTE:** there is no colon (":") used in the input field names. It's just <code>user[username]</code>.
* make sure your form actions match the appropriate routes!
* Get in the habit of using <code>autofocus</code> in forms, usually in whatever is the first input field on the entire page. It makes your users happier, even if they don't realize it at the time.

### Back to the controller

So now that data can be sent through, let's build out those POST routes.

New accounts are fairly straight-forward, since we're not doing anything with password encryption (BCrypt) just yet. Take in the form params and use them to create a new User, then set the session cookie:

```ruby
post '/new_account' do
  user = User.create(params[:user])
  session[:user_id] = user.id
  redirect '/'
end
```
You can choose to redirect them wherever it makes the most sense to you to send your users after they've made a new account. Maybe it's back to the home page? Maybe it's to their profile page (<code>redirect "/users/#{user.username}"</code>? Maybe it's something fancier? Totally up to you.

Logging in is a little tricker, since we'll need to make sure the user has submitted the correct password.

If your login form takes in a username and password, the process should go:

1. Find the user based on <code>params[:user][:username]</code>
2. Check if <code>user.password</code> matches <code>params[:user][:password]</code>
3. If it matches, redirect the user to wherever (see above).
4. If it doesn't match, you'll probably want to just send them back to the login page so they can try again.
5. (You can get fancy and show an error message on the login page so the user knows why they've been sent back there!)

```ruby
post '/login' do
  user = User.find_by_username(params[:user][:username])
  if user.password == params[:user][:password]
  	session[:user_id] = user.id
  	redirect '/'
  else
    redirect '/login'
  end
end
```
Except, oh man, model code in the controller! This can be refactored to:

```ruby
post '/login' do
  if user = User.authenticate(params[:user])
  	session[:user_id] = user.id
  	redirect '/'
  else
    redirect '/login'
  end
end
```
```ruby
class User < ActiveRecord::Base

  def self.authenticate(params)
    user = User.find_by_name(params[:username])
    (user && user.password == params[:password]) ? user : nil
  end
  
end
```
This creates a User class method <code>authenticate</code> which takes in a <code>params</code> hash. The controller sends this method <code>params[:user]</code> as that hash.

Why a class method? Because you need to find a specific user, but you don't want to make the controller do that work. The model should do that work, but without a specific user (yeah, it gets kind of circular), you can't use an instance method… so you have to use a class method.

Speaking of! What is this class method doing?

The first line is trying to find a specific user based on the username that was submitted via the login form, and storing whatever it finds in the local variable <code>user</code>.

The second line is saying:

* **IF** a user was found (because if the submitted username didn't actually exist in the database, <code>User.find_by…</code> would have returned <code>nil</code>, which is the same as <code>false</code>
* **AND IF** that found user's password matches the password that was submitted in the form
* **THEN** this function will return <code>user</code>
* **ELSE** this function will return <code>nil</code>

So if you look back at the refactored route, it's saying:

* If <code>User.authenticate</code> returns a user, store that user in a local variable <code>user</code>, set the session cookie for this user and redirect to the root path
* Else redirect to the login page so the user can try again

### Back to the helper method

Maybe your app has routes you only want logged-in users to access. I.e., only logged-in users can create blog posts or upload photos or submit links or leave comments (etc.).

Let's pretend this is a blogging app, and you have a route for the page that contains the form used to create a new post:

```ruby
get '/create_post' do
end
```
Because you have that helper method, you can do something like:

```ruby
get '/create_post' do
  if current_user
    erb :create_post
  else
    redirect '/login'
  end
end
```
which just says, if <code>current_user</code> returns a user, load this page and show the form for creating a new post. Otherwise, if <code>current_user</code> returns <code>nil</code>, redirect the user to the login page.

### Bonus funtastical features to think about

* If a non-logged in user clicks on a link to a protected route (meaning, only logged-in users can see that page) and is redirected to the login page, and then the user successfully signs in… wouldn't it be nice if the user could be redirected to the page they were trying to access in the first place?
* How can the app know when to display an error message?
* Remember: you can store ~ 4 Kb in a session
* If a user is logged in, are there still pages that user shouldn't be able to access? (Think about editing a profile page. Should users be able to access the profile edit page of other users? (Easy answer: no)).
