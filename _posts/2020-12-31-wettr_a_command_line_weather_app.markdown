---
layout: post
title:      "wettr: A Command Line Weather App"
date:       2020-12-31 00:05:53 -0500
permalink:  wettr_a_command_line_weather_app
---

My first gem! This post is all about wettr -- a command line weather app written in Ruby and published to [rubygems.org](https://rubygems.org/gems/wettr). You can find the source code for this gem at (https://github.com/JWDonovan/wettr). It uses OpenWeatherMap's Current Weather API, as well as, ipapi.co's IP Address API to give you current weather at your location or a location of your choosing.
Give it a try. But, if you want to read a little more about how this gem was made, then stick around.

As part of the Flatiron School's Software Engineering course, in which I am currently enrolled, students are required to make 5 personal projects; the first of which being a command line app written in Ruby. For this project I decided to make a weather app simply because it sounded like a fun project and might be something I'd use day-to-day. After reading some helpful articles on creating Ruby gems, like [this one from bundler](https://bundler.io/guides/creating_gem.html) or [this one from rubygems.org](https://guides.rubygems.org/make-your-own-gem/), I set off to make my own.

**Project Setup**

I used [bundler](https://rubygems.org/gems/bundler) to create the initial project scaffolding.
```bash
$ bundle gem wettr
```
After updating my gemspec file, I was able to build a simple "Hello, world" version of the app to make sure things were working correctly
```bash
$ gem build wettr.gemspec
```
Because my app was going to use OpenWeatherMap's API get retrieve it's weather data, I needed a good way to call API's and parse their JSON responses into Ruby objects. Here comes [HTTParty](https://github.com/jnunemaker/httparty) to save the day! HTTParty is a gem that makes sending API requests and handling their responses simple. I simply added the gem as a runtime dependency in my gemspec and installed it locally.

```ruby
# ./wettr.gemspec
spec.add_dependency "httparty", "~> 0.18.1"
```

```bash
$ bundle install
```
**Making an API Request**

In order to make a request to OpenWeatherMap's API, I first needed an API key. After getting the key, I needed a way to include it in the request to OpenWeatherMap and get the data back in a nice, readable format. To do this, I created a class called `Wettr::WeatherAPI` that inherited from the `HTTParty` base class. This gave my class access to HTTParty methods like `get` and class variables like `base_uri` and `default_params`

By setting the `base_uri` variable of my class to `"https://api.openweathermap.org"` and including my API key in the `default_params` variable, I was finally able to send my first request.

```ruby
# lib/wettr/weather_api.rb

require "httparty"

class Wettr::WeatherAPI
  include HTTParty
	
  base_uri "https://api.openweathermap.org"
  default_params appid: <API_KEY_HERE>

  def self.call
    response = self.get("/data/2.5/weather")
    response
  end
end

puts Wettr::WeatherAPI.call
```
**Storing the Response**

Once I make the API request I need a way to store the resulting response into a Ruby object. In order to keep a good separation of concerns, I decided to create a separate `Wettr::Weather` class to represent the data retrieved from the API call. This class will handle turning the hash response from HTTParty into a new `Weather` object and printing that weather data to the screen. The class will look something like this:

```ruby
class Wettr::Weather
  attr_reader :list_of_weather_data_fields_here
	
  def initialize(list_of_weather_data_fields_here:)
    # this is simply an example, we would want to set our variables
    #based on the fields returned in the api, just use your imagination
    
		@list_of_weather_data_fields_here = list_of_weather_data_fields_here
	end
	
  def print
    # print out our weather data in a nice, readable format
  end
	
  private
  
  def self.new_from_api_response(response)
    weather = self.new(list_of_weather_data_fields_here: response["data_fields"])
    weather
  end
end
```

**Adding a CLI**

Now that we can make API calls and store their responses, we need a way for users to interact with our program. In order to do this we need a command line interface (CLI). Our CLI doesn't need to be that fancy. Simply offer a few basic commands to user, parse any arguments they might give us, and display the output.

My first thought to solve this problem was to reach for a very popular gem called `optparse`. Optparse is a library used to make creating CLIs and their parsers easier. After playing around with this gem for a while, I decided this wasn't a good fit for my project at the moment. Optparse does not support mutually-exclusive arguments out of the box, which is something that I would like to have in my next version of wettr.

Next I tried `docopt` -- originally a Python library ported to Ruby. Docopt provides a DSL for creating CLI argument parsers based on standard UNIX man page coding conventions. Docopt was a step in the right direction, in that it supported mutually-exclusive arguments, but I did not feel like this gem gave me the ability to finely tune the parsers that were being generated or when and how they are invoked.

In the end, I decided to parse the command line arguments with plain, ol' Ruby. This works for now, but will soon become buggy and hard to maintain as I add more command options in future versions of wettr.

**Publishing and Pitfalls**

Now that I have a working minimum viable product, I can finally push this project out the door and hopefully get it into the hands of real people. Bundler makes building and pushing to rubygems.org a simple one-step process:
```bash
$ bundle exec rake release
```

My project is published and out in the world, ready for people to enjoy! Well, almost. Unfortunately, things did not go as smoothly as that. My original plan was to ship my OpenWeatherMap API key in the compiled gem so that users of wettr could get up and running immediately without having to provide their own key. Because OpenWeatherMap charges a fee for API usage above a certain amount, I thought it would be wise to hide my key so that only wettr users can use it indirectly.

To do this, I included the popular gem `dotenv`. Dotenv is a gem to share and load environment variables, like API keys, without them being tracked in a public viewable open-source git respository. I added `dotenv` to my project's gemspec and the relevent code to my project's source files. All things were running smoothly until I went to publish the project. Dotenv could not find the API key and my app would not work.

To solve this problem, normally you would manually include the environment variables on the host machine, but I don't believe this is supported by rubygems.org. Since I wanted to keep my project open-source and I didn't want to include my API key in the source code directly, I decided the best course of action would simply be to have users include their own API keys.

Users are now prompted to create a .wettr.yml file in their home directory and include an OpenWeatherMap API key in order to use wettr. While this is not a perfect solution, this seems to be the best one for now. User's can now provide their own keys and are not dependent on my key or it's current rate limit.

With that production hotfix now in the place, the app is published and working in production! To install it, simply type the following:
```bash
$ gem install wettr
```

**Conclusion**

Well, it's done and out there! This sure has been a big learning experience and things didn't always go as planned, but I did very much enjoy the adventure. From learning Ruby just a few weeks prior, to now having my own published gem, the Ruby language, and the helpful community around it, makes getting up and running quickly a pretty painless and enjoyable experience. I look forward to future projects in this language and, of course, adding more fun features to wettr.

