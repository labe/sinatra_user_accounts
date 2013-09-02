##BCrypt!

###Requirements

In your <code>Gemfile</code>:

```ruby
gem 'bcrypt-ruby'
```

In your <code>/config/environment.rb</code> file:

```ruby
require 'bcrypt'
```

In your schema:

```ruby
t.string  :password
```
SHOULD but does not *have* to become:

```ruby
t.string  :password_hash
```

("SHOULD" because technically, you'll now be storing the hashed password, not the password itself, and this naming convention will lessen confusion in the encryption/decryption methods.)

In your <code>User</code> model (or whichever model is going to have the password attribute):

```ruby
class User < ActiveRecord::Base
  include BCrypt
  
  def password 
    @password ||= Password.new(password_hash)
  end
  
  def password=(pass)
    @password ||= Password.create(pass)
    self.password_hash = @password
  end

  #remember how we redefined User.create before? That gets a modification as well:
  
  def self.create(params)
    user = User.new(
      :username => params[:username],
      :email => params[:email] )
    user.password = params[:password]
    user.save
    user
  end  
end
```

###What's going on here?

When you create a new user, <code>user.password = params[:password]</code> is calling <code>def password=(pass)</code>, where the <code>pass</code> argument is now <code>params[:password]</code>. BCrypt takes <code>pass</code> and creates a hash out of it, then stores it in an instance variable <code>@password</code>, then assigns it to the user's <code>password_hash</code> attribute.

For logging in: remember that <code>User.authenticate</code> method?

```ruby
  def self.authenticate(params)
    user = User.find_by_name(params[:username])
    (user && user.password == params[:password]) ? user : nil
  end
```

When you evaluate <code>user.password == params[:password]</code>, <code>user.password</code> calls the <code>def password</code> method, which takes whatever's stored as that user's <code>password_hash</code>, unencrypts it via <code>Password.new</code>, and returns that unencrypted value as an instance variable <code>@password</code>. 

BCrypt also has a built in comparison instance method, <code>==(secret)</code>, where <code>secret</code> is just a fancier way of saying "password". If <code>@password</code> (the unencrypted hash) is equivalent to the password the user submitted on login (<code>params[:password]</code>), this method returns true and the user is then authenticated.

###That's it!

Yay!

###See also

[http://bcrypt-ruby.rubyforge.org/classes/BCrypt/Password.html](http://bcrypt-ruby.rubyforge.org/classes/BCrypt/Password.html)
