---
layout: post
title:      "Media Critic: Review Your Favorite Films"
date:       2021-07-12 13:42:00 +0000
permalink:  media_critic_review_your_favorite_films
---

Media Critic is a website for users to review and critique their favorite movies. Users can share their own experiences or read what others have thought after experiencing a new film. Media Critic can also be helpful for those who simply trying to find something new to watch.

Media Critic was created using Ruby on Rails. The code for this project can be found at my GitHub profile at [https://github.com/JWDonovan/media_critic](https://github.com/JWDonovan/media_critic).

## Project Setup
Below are the steps needed to run Media Critic on your own device. These instructions assume you already have Ruby, Ruby on Rails, and PostgreSQL installed and configured on your target machine.

* Download and extract the source code from GitHub or clone the repository if you have git installed. You can find the source code by clicking this [link](https://github.com/JWDonovan/media_critic).
* Navigate to the source directory and run the following command to install local dependencies:

```ruby
bundle install
```

* Run these commands to create and setup the database:

```ruby
rails db:create
rails db:migrate
rails db:seed
```

* Finally, start the rails server with this command:

```ruby
rails s
```

## Project Requirements

This project was created as part of my Flatiron School curriculum. The requirements were to create a Ruby on Rails app that supports user management, logging in via OAuth 2.0, at lease three database models that form a many-to-many relationship and also accept nested attributes. We'll take a look at how I tackled each of these requirements in the sections below.

## User Management with Devise and Omniauth

The foundation of most dynamic websites and webapps is a comprehensive user management system. In order for users to be able to interact with your website and provide safety, security, and customizability, you need to be able to authenticate users and, very often, authorize them to perform specific, restricted actions as well.

Since these functions are so common and have standarized throughout most websites, there are many libraries and frameworks created to help support these features. By far the most common such library for the Ruby language is the gem [Devise](https://github.com/heartcombo/devise). Devise is an industry standary for Ruby on Rails projects that need to support authentication and many other gems and libraries have been created to support it and extend it's features.

After installing and configuring Devise according to its instructions, I was able to setup a basic Devise user model and support basic user sign up, sign in, and sign out functions. In order to allow a user to sign in using an OAuth provider, in my case Google OAuth, I needed to add another gem that works well with Devise. I decided to use [Omniauth](https://github.com/zquestz/omniauth-google-oauth2), specifically the version that was created to handle Google authentication. While installing and testing this feature, I ran into my first major bug in this project. Ruby on Rails had recently been updated to version 6 and Omniauth was not able to fhandle this upgrade properly. After many hours trying to diagnose this problem, I discovered the gem was simply not compatible with the recent Rails upgrade and I would have to downgrade my project to Rails version 5. After this downgrade was complete, I was able to login using Google's authentication service and Devise was able to create user accounts based on the information provided from Google.

Many applications also need to support user authorization. Often times, users need to be restricted from or granted access to perform specific actions. There are many ways to support these features and many common gems exist to make this task easier as well. The basic features I needed to support for Media Critic was to determine if a user was logged in, this allows them the review movies and edit their profile. I also needed to check if they had permission to create movies and delete movies, since I only wanted certain users to be able to perform this function. Because Devise comes pre-built with a few methods to check if a user is logged in and because I only needed to perform one simple authorization check, I decided that using a gem to support these features would not be necessary. Instead, I simply added a single boolean value on the Devise user model that checks if a user has the ability to create and destroy movies and used Devise for all other authorization needs.

## Model Associations and Nested Attributes

Media Critic has three ActiveRecord models: a user, movie, and review. User's can write reviews that belong to movies. Movies have many reviews and many users through those reviews. And reviews belong to both movies and users. The basic code for these models is listed below:

```ruby
# user.rb

class User < ApplicationRecord
  has_many :reviews, dependent: :destroy
	has_many :movies, through: :reviews
	
	devise :database_authenticatable,
         :registerable,
         :validatable,
         :omniauthable,
         omniauth_providers: [:google_oauth2]

  def self.from_google(email:, provider:, uid:)
    create_with(uid: uid, provider: provider, password: Devise.friendly_token[0, 20]).find_or_create_by!(email: email)
  end
end
```

```ruby
# review.rb

class Review < ApplicationRecord
  belongs_to :movie
	belongs_to :user
end
```

```ruby
# movie.rb

class Movie < ApplicationRecord
  has_many :reviews, dependent: :destroy
	has_many :users, through: :reviews
	accepts_nested_attributes_for :reviews, reject_if: proc { |attributes| attributes['title'].blank? }
end
```

Notice the last line of the movie model. This allows us to create a movie and review it at the same and setup all the necessary associations between the two in one single action. This can make the user experience much better by speeding up repetitive tasks. We'll explore more about how this is accomplished from the perspective of the form view and the corresponding controller in the next section.

## Nested Forms

Ruby on Rails has many form builders that allow us to create complex form in a simple, intuitive manner. In order to support creating a parent and child relationship in a single form submission, aka a nested form, like a movie and its corresponding review, we can use code like the following:

```erb
<%= form_with model: @movie do |f| %>
  <%= f.text_field :title %>
  <%= f.number_field :release_year %>
  <%= f.rich_text_area :synopsis %>
	
  <%= f.fields_for :reviews do |r| %>
    <%= r.text_field :title %>
    <%= r.number_field :rating %>
    <%= r.rich_text_area :content %>
  <% end %>
<% end %>
```

Using the `fields_for` helper function, we can include another nested form within our movie form to create a review. Once the movie form is submitted the information will be sent back to the controller at the same time. Speacking of the controller, let's take a look at how that handles the creation of two elements at once.

```ruby
# movies_controller.rb

class MoviesController < ApplicationController
  # GET /movies/new
	def new
    @movie = Movie.new
    @movie.review.build()
	end

  # POST /movies
  def create
    @movie = Movie.new(movie_params)
    current_user.reviews << @movie.reviews
    @movie.save
	end
	
  private
	
  def movie_params
    params.require(:movie).permit(:title, :synopsis, :release_year, reviews_attributes: [:id, :title, :rating, :content])
	end
end
```

Notice how in the `new` function, we must first create a new, blank movie and then we can use the `build` function to create an empty reviews that is associated with it. This allows us to create the forms we need to fill in the information that then is sent back to the controller to do the rest.

Once the form is submitted, the `create` function is called. This function creates the movie and review at the same time. In order for this to work, we must let Rails know exactly what parameters we are expecting. This is handled by the `movie_params` function. Notice that in this fuction we must explicitly allow the review attributes to be sent along with the movie attributes. Once this is done, we can safely create the movie and review. But since reviews belong not only to a movie, but also the user that wrote the review, we also need to make this association before finally saving the movie.

## Conclusion

This project provided some interesting challenges that I had never faced before. The nested attributes and Omniauth support were among the most interesting and difficult problems to solve. After completing the basic requirements the only left for me to do was flush out the design, add some more interesting features, and clean up and secure the code for production. Now that I've completed this project, I look forward to bringing these features to future websites that I make. Allowing users to sign in using their Google account and streamlining their workflow by allowing them to create multiple associated models in a single action will certainly be helpful in many projects to come.
