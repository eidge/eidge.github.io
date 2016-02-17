---
layout: post
title: Getting validations out your models
date: 2015-12-13
tag: markdown
blog: true
#star: true
---

[Validations](http://guides.rubyonrails.org/active_record_validations.html) are the Rails way of validating user input and data integrity. They're usually written inside your model and constrain which states are saveable to the database.

```ruby
class Human < ActiveRecord::Base
  attr_accessor :name
  validates :name, presence: true
end

# > human = Human.new
# > human.valid?
# => false

# > human.name = 'jonh'
# > human.valid?
# => true
```

They're a simple but effective tool to manage your data validation but shouldn't be abused as a data integrity mechanism. In this post I'm making a case for getting some of your validations out of your model and closer to the user input you're validating.

### Data integrity checks belong in the database

If you want to guarantee a given state (say name presence), than that validation belongs in the database and should be replicated in the application layer for user feedback.

**But that's not DRY, is it?**

It isn't. But your model validations can be bypassed (think ```Human.save(validate: false)```) or there might be a few services using your database directly and therefore bypassing your application data checks. Therefore, the only way to **guarantee data integrity is to rely on your database to enforce it**.

### Data validation belongs in the application

Data integrity and data validation are complementary but two different things.

Say you only want to allow users called "Jonh" to sign up, that's the kind of validation that belongs in your application code. It is business logic and therefore your database has got nothing to do with it.

#### A simple misleading example

With rails, that's quite an easy requirement:

```ruby
class User < ActiveRecord::Base
  validate :user_is_named_jonh
  
  def jonh?
    first_name.downcase == 'jonh'
  end
  
  private
  
  def user_is_named_jonh
    errors.add(:first_name, :not_jonh) unless jonh?
  end
end
```

The code above seems like the right way but the truth is that it isn't.

What happens if you decide you have too many "Jonh"s on your platform and from now on you only want to accept "Paul"s signing up?

That's simple enough, you say:

```ruby
class User < ActiveRecord::Base
  validate :user_is_named_paul
  
  def paul?
    first_name.downcase == 'paul'
  end
  
  private
  
  def user_is_named_paul
    errors.add(:first_name, :not_paul) unless paul?
  end
end
```

Done! Except... you just broke the app for every existing user named "Jonh" as they won't be able to update their user information unless they change their name to Paul.

The problem here, is that you don't really want to validate your model data. You want to validate user input upon registration and your model is not a good place to do it. **Model validations should be used to ensure your models are in a state your application can handle** and in this case, the application code doesn't really care what the first name of the user is.

#### The bad solution

```ruby
class User < ActiveRecord::Base
  validate :user_is_named_paul, if: :new?
  # ...
end
```

This definitely works! But once you start collecting a few validations like this it starts to be difficult to argue which validations run and when.

If your validations are coupled to a specific action (say user registration) then why not couple it directly to that action instead of the model?

#### A better solution: Form objects

[Form objects](https://robots.thoughtbot.com/activemodel-form-objects) have been an essential tool in my day to day with Rails. Not only it makes dealing with complicated forms cleaner (think *accepts__nested__attributes*, or *json* fields) it also helps declutter your models and controllers.

Form objects are great for cases like the one we've been going over:

```ruby
class User < ActiveRecord::Base
  validates :first_name, :email, :password_hash, presence: true
end

class NewUserForm < ActiveRecord::Model
  attr_accessor :first_name, :email, :password
  
  validates :first_name, :email, :password, presence: true
  validate :user_is_named_paul
  
  def paul?
    first_name.downcase == 'paul'
  end
  
  def user
    User.new(first_name: first_name, email: email, password: password)
  end
  
  def valid?
    super && user.valid?
  end
  
  def save
    valid? && user.save!
  end
  
  private
  
  def user_is_named_paul
    errors.add(:first_name, :not_paul) unless paul?
  end
end
```

Now we keep validations that ensure our data is in a state our application can handle in the model and validations that are tied to user registration in a object that sole purpose is validating user input for that specific action  (users have name, email and password_hash validations belong in the database as well).

If the requirements change which users are allowed to sign up, only one class needs to be changed. Because the changes are self contained you don't have to worry about making sure your new validations don't break in other cases. They will only run on user registration.

### Summing up

- When writing a validation, try to figure out which layer you're validating. Are you guaranteing data integrity and therefore you should also write a database constraint? Are you guaranteing you application data is in valid, computable state? Or are you enforcing business logic as data?

- Form objects are a good tool to declutter your models and controllers from parameter validation, complex associations forms and to move your validations closer to the user input.

- When in doubt, prefer to write your validations in the model and refactor them out to form objects once you're sure they do not belong there. If you end up instantiating form objects outside of controllers and without user input, you're doing something wrong.
