---
layout: post
title:      "Chaperone: Inspiration for Your Next Vacation"
date:       2021-01-10 22:52:59 +0000
permalink:  chaperone_inspiration_for_your_next_vacation
---

Chaperone is a website where users can create and share their favorite travel destinations. You can create a post about places you've been or would like to visit and other users can share in your experience. Similarly, if you're ever looking for travel inspiration of your own, browsing Chaperone is a good way to discover your next vacation.

Chaperone was created using Ruby and [Sinatra](http://sinatrarb.com/). The code for this project can be found at my GitHub profile at [https://github.com/JWDonovan/chaperone](https://github.com/JWDonovan/chaperone). I also plan to publish a demo of Chaperone onto [Heroku](https://heroku.com) soon. With all of that out of the way, let's get to how I built it!

## Sinatra Project Structure

Unlike Ruby on Rails, Sinatra does not have a strictly enforced folder structure or project layout. You are free to structure your project as you see fit. Because of this, many Sinatra apps take shape in a lot of different ways. However, there are two basic ways that most Sinatra apps are laid out: flat and nested.

The flat folder structure is similar to other basic Rack-based apps, with a `config.ru` file  at the project root and all other files and folders laid out at the project root as well. The other layout, and the one I chose for Chaperone, was the nested layout. This is where most of the project code is located in a folder call `app` at the project root. All controllers, models, and views have their own folders within this main app folder. This folder structure is most similar to the one used by Ruby on Rails. I chose this layout simply because of it's similarity to Ruby on Rails and because I thought this layout would be able to grow more cleanly as I added new features.

Many new Sinatra apps that use this nested folder structure use a gem called [Corneal](https://thebrianemory.github.io/corneal/) to create the basic app structure for you. Because I wanted to understand all the ins and outs of my Sinatra app, I decided not to use this gem and instead start from scratch.

The bare minimum of any Sinatra app usually includes a rackup file called `config.ru` at the project root. We also need to create a basic root controller class that inherits from `Sinatra::Base`. The following is what a basic Sintra hello-world app might look like using this folder layout.

```ruby
# config.ru

require './environment'
run ApplicationController
```

```ruby
# environment.rb

ENV['SINATRA_ENV'] ||= 'development'
require 'bundler/setup'
Bundler.require(:default, ENV['SINATRA_ENV'])

require_all 'app'
```

```ruby
# app/controllers/application_controller.rb

class ApplicationController < Sinatra::Base
  get '/' do
    'Hello, world!'
  end
end
```

```ruby
# Gemfile

source 'https://rubygems.org'

gem 'require_all'
gem 'sinatra'
gem 'shotgun'
```

The project can be started by running the following commands:

```bash
$ bundle install
$ shotgun
```

Viewing the project in a browser should output `Hello, world!`

The full project structure for Chaperone looks something like this:

```
Chaperone
└─ app
   └─ controllers
      └─ application_controller.rb
      └─ destinations_controller.rb
      └─ users_controller.rb
   └─ models
      └─ destination.rb
      └─ user.rb
   └─ views
      └─ destinations
         └─ edit.erb
         └─ index.erb
         └─ new.erb
         └─ show.erb
      └─ users
         └─ edit.erb
         └─ login.erb
         └─ new.erb
         └─ show.erb
└─ config
   └─ database.yml
└─ db
   └─ migrate
   └─ schema.rb
   └─ seeds.rb
└─ public
   └─ css
      └─ style.css
   └─ images
└─ spec
   └─ controllers
      └─ application_controller_spec.rb
      └─ destinations_controller_spec.rb
      └─ users_controller_spec.rb
   └─ models
      └─ destination_spec.rb
      └─ user_spec.rb
   └─ spec_helper.rb
└─ Gemfile
└─ LICENSE.txt
└─ README.md
└─ Rakefile
└─ config.ru
└─ environment.rb
```

## Controllers

In the previous hello-world example, I showed the basic `application_controller.rb` file, which includes a controller class that inherits from `Sinatra::Base`. The base controller for Chaperone is not too different from this example. The only things that needed to be added were some base configurations for Sinatra, some global helper functions to be used by other controllers, and displaying something useful when a user navigates to our website instead of just "Hello, world!".

Our updated `ApplicationController` class looks something like this:

```ruby
# app/controllers/application_controller.rb

class ApplicationController < Sinatra::Base
  configure do
    set :views, 'app/views'
    enable :sessions
    set :session_secret, 'super_secret'
    register Sinatra::Flash
  end
	
  get '/' do
    redirect '/destinations'
  end
	
  helpers do
    def logged_in?
      !!current_user
    end
		
    def current_user
      User.find_by(id: session[:user_id])
    end
  end
end
```

In our `ApplicationController` inside a configuration block, we enable sessions and flash messages as well as setting the default path for Sinatra to look for our views. We also create some handy helper methods to keep track of the current logged in user that we can reuse in our other controllers.

The other two controllers that we need for Chaperone, `UsersController` and `DestinationsController`, will inherit from this base `ApplicationController` class like this:

```ruby
# app/controllers/destinations_controller.rb

class DestinationsController < ApplicationController
  get '/destinations' do
    redirect '/login' unless logged_in?
    erb :'destinations/new'
  end
end
```

You can see that in Sinatra, we can define routes with the `get` method. Other HTTP verbs are also supported, such as `post`, `patch`, and `delete`. In our method, we can define what actions to take when the user makes a request to that route. For a simple `get` request, we can render out an HTML file by sending a command to the built-in erb rendering engine that points to our erb view. Sinatra will build this path based on the value of `:views` that we set previously in our `ApplicationController`.

## Models and Associations

For this project, I am using ActiveRecord and Postresql to handle our model and database needs. This means that we can leverage ActiveRecord to create associations between our models simply and use validations to verify our data matches what we expect to have in our database.

To match our `DestinationsController` and `UsersController`, we also have destination and user models. These are stored in our `app` directory in their own `models` folder.

```ruby
# app/models/destination.rb

class Destination < ActiveRecord::Base
  belongs_to :user
  validates_associated :user
end
```

```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  has_secure_password
  has_many :destinations

  validates :email, :password_digest, :first_name, :last_name, presence: true
  validates :email, uniqueness: true
end
```

With these validations in place, we can make sure our data matches our expectations before being saved to our database. We also can verify that there is an association between our user and their destinations. Making this association will make it easy to verify the owner of a destination when writing our controller method that might need to be restricted based on a user's permissions. These associations will also make building our views based on what destinations the user has access to easier as well.

## Views and Layouts

Back in our `ApplicationController` class we set the `:views` directory to 'app/views'. This is so Sinatra knows where to look for our ERB view templates. Our controllers can then use a relative path to render these templates with `erb :'path/to/template'`, like we saw in our `DestinationController` example. Sinatra also has the ability to use other templating engines and languages other than ERB, like HAML. I decided to use ERB simply because it is included in Ruby and suits all of my needs at the moment.

In order to make our templates more [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), we can leverage partials to reuse our HTML. In Sinatra, the way this is done is simply by including the other ERB file from with our current template like this:

```erb
# main.erb

<!DOCTYPE html>
<html lang="en">
  <body>
    <%= erb :'partial' %>
  </body>
</html>
```

```erb
# partial.erb

<p>Hello from a partial!</p>
```

Similar to partials, Sinatra also supports layouts. This allows us to reuse the structure of our HTML pages and simply focus on writing the content that is specific to that page. The way that layouts and partials differ, is that layouts include a `yield` command which is the point in the rendering process in which the layout ERB stops and our view ERB begins being parsed.

By default, Sinatra will look for a `layout.erb` file in our views directory. All views, unless explicitly told not to, will use this base layout file when rendering. To use another layout, you need to pass the path to the layout in the call to the ERB rendering engine. This is done in Sinatra like so:

```ruby
erb :'path/to/view', layout: 'path/to/layout'
```

Adding layouts and views is the final step in making our Sinatra app complete. Users now have a way to interact with Chaperone and manage their own destinations, as well as, viewing reading other users' posts.

## Conclusion

I enjoyed making my first app with Sinatra. Although my project structure was very similar to a Ruby on Rails app in the end, it was a great learning experience being able to set everything up from scratch. Sinatra is a great way to get a better idea of what a larger framework like Ruby on Rails is doing under the hood. Although I plan to use Ruby on Rails for many of my future projects, I might use Sinatra again if I ever want a little more find-tuned control of exactly how my project is laid out and functions. Sinatra is a simple, but extremely flexible framework that strikes a good balance between Ruby magic, ease of use, and customizability that just can't be found anywhere else.
