---
layout: post
title:  "Building an SMS App Using Rails 4.2 and Twilio"
date:   2015-3-8
---

This is part one of three on building an app that utilizes Rails 4.2 and ActiveJob along with Twilio and Heroku.

<p class="intro"><span class="dropcap">I</span>n this part we are going to create a SMS based English to Pig Latin translator. In the end you will be able to text a phone number with an English sentence and have the Pig Latin translation sent back to you. There will be no front end. The only way to communicate with the app as an end user is via text message.

My code is on [Github Here](https://github.com/D-Land/anslatortray) and if you look at the commit history it follows along with this tutorial  closely.

First, we're going to setup a new rails project. Let's call it anslatortray (that's pig latin for translator).
Run `rails new anslatortray` to create the project. This is also a good time to initialize a git repo for the project and make your initial commit. I'm not going to mention making commits throughout the piece but I suggest you make them often.

This project is going to consist of a single model and controller. The model is going to hold the to and from phone numbers as well as the body of the SMS message. Since Twilio formats its phone numbers as 12 digit strings starting with a "+1"(e.x. "+11111111111"), we will have two such strings on our model. _If you are using a Twilio phone number that is not US it may not be formatted exactly the same, check their [article on international number formatting](https://www.twilio.com/help/faq/phone-numbers/how-do-i-format-phone-numbers-to-work-internationally)._ In addition, we will add a non-null string to store the body. We can create this by running `bundle exec rails g model TextMessage to:string from:string body:string`. This will generate both the model file and the migration (as well as some files used for testing that we are going to ignore, at least for this part of the tutorial).

We need to open up the migration and add the 12 digit limit and non-null column modifiers.

```ruby
# File: /db/migrate/YYYYMMDDHHMMSS_create_text_messages.rb

class CreateTextMessages < ActiveRecord::Migration
  def change
    create_table :text_messages do |t|
      t.string :to,   limit: 12
      t.string :from, limit: 12
      t.string :body, null:  false

      t.timestamps null: false
    end
  end
end
```

We can now run `bundle exec rake db:migrate` to create our databases and the schema. This is a good time to commit to git.

We can use an after\_create hook in the model to run our process method on all newly created TextMessage records. For now let's put in the after\_create and a process method that just prints the TextMessage body to the console. We will come back shortly and flesh out this method to actually process text messages.

```ruby
# File: /app/models/text_message.rb

class TextMessage < ActiveRecord::Base
  after_create :process

  def process
    puts body
  end
end
```

Having a place to store our messages is great but we need to setup a controller and a route to get our messages into the model. Let's run `bundle exec rails g controller TextMessages` to auto generate the files. Then we'll want to add a few things to /app/controllers/text\_messages\_controller.rb. At the top we're going to need to disable CSRF (Cross Site Forgery) protection to allow Twilio to post to this controllers URL. We are also going to need a create method that takes the params from the post request and creates a new TextMessage record. Finally, need to send an ok response to let Twilio know we got the request. _You'll get errors later if you forget the `head: ok` bit._

```ruby
# File: /app/controllers/text_messages_controller.rb

class TextMessagesController < ApplicationController

  skip_before_action :verify_authenticity_token

  def create
    TextMessage.create(message_attributes)
    head :ok
  end

  private

  def message_attributes
    { to: params[:To], 
    from: params[:From], 
    body: params[:Body].strip.downcase }
  end
end
```

To get to that controller action, we're now going make a route to our TextMessagesController by adding `post 'text', to: 'text_messages#create'` to the top of the config/routes.rb.

We should now test to make sure our route, controller, and model code thus far are working together. We can do this by starting a rails server by running `bundle exec rails s` and in another terminal using CURL to emulate a POST request. The CURL command we are going to use is `curl -X POST localhost:3000/text -d "To=%2B12345678910" -d "From=%2B12345678911" -d "Body=Catch me if you can"`. The %2B part of the phone numbers is actually a '+'.

You should see something similar to this in the server log if everything is working:

```bash
Processing by TextMessagesController#create as */*
  Parameters: {"To"=>"+12345678910", "From"=>"+12345678911", "Body"=>"Catch me if you can"}
   (0.0ms)  begin transaction
  SQL (0.3ms)  INSERT INTO "text_messages" ("to", "from", "body", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?)  [["to", "+12345678910"], ["from", "+12345678911"], ["body", "catch me if you can"], ["created_at", "2015-03-08 19:44:38.993672"], ["updated_at", "2015-03-08 19:44:38.993672"]]
catch me if you can
   (0.6ms)  commit transaction
Completed 200 OK in 20ms (ActiveRecord: 1.2ms)
```
Make sure that you can see on the 3rd line from the bottom where the SMS body is printed to the console.

Now we have a structure around which we can build our Pig Latin translator. Let's go back to the TextMessage model and change the process method to actually handle converting some text to Pig Latin. 

Let's assume for now that our Twilio number is +12345678910. Later we'll use our actual Twilio number and load it into our application using an environment variable. So when we get a TextMessage whose __to__ property is equal to our Twilio phone number we know that it is an incoming SMS and should be translated. Otherwise, it is a record that we created that contains the Pig Latin translation and should be sent out. With that in mind we can edit our process method. 

If the message is incoming we want to break up the sentence into individual words and translate them. Then we want to rejoin all of our translated words back into a single string. Then we'll create a new TextMessage record with the correct to and from fields to send the translation back.

```ruby
# File: /app/models/text_message.rb

def process
  if to == '+12345678910'
    new_sentence = body.split.map{ |x| translate(x) }
    new_sentence = new_sentence.join(" ")
    TextMessage.create(to: from, from: '+12345678910', body: message_body)
  else
    send_outgoing
  end
end
```

At this point we should also add the actual translate method which we got from [stackoverflow!](http://stackoverflow.com/a/13499011/4541669)

```ruby
# File: /app/models/text_message.rb

# From http://stackoverflow.com/a/13499011/4541669
def translate(str)
  alpha = ('a'..'z').to_a
  vowels = %w[a e i o u]
  consonants = alpha - vowels

  if vowels.include?(str[0])
    str + 'ay'
  elsif consonants.include?(str[0]) && consonants.include?(str[1])
    str[2..-1] + str[0..1] + 'ay'
  elsif consonants.include?(str[0])
    str[1..-1] + str[0] + 'ay'
  else
    str
  end
end
```

This takes care of processing incoming SMS and generating a new TextMessage record that contains the outgoing text information but what about actually sending the message back. For now lets make a send outgoing method that just prints the body of the text message to the console to make sure our translator is working!

```ruby
# File: /app/models/text_message.rb

def send_outgoing
  puts body
end
```

Start the rails server again and use curl to verify that everything is working. If it is, you should see this:

```bash
Processing by TextMessagesController#create as */*
  Parameters: {"To"=>"+12345678910", "From"=>"+12345678911", "Body"=>"Catch me if you can"}
   (0.0ms)  begin transaction
  SQL (0.4ms)  INSERT INTO "text_messages" ("to", "from", "body", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?)  [["to", "+12345678910"], ["from", "+12345678911"], ["body", "catch me if you can"], ["created_at", "2015-03-08 19:59:52.195009"], ["updated_at", "2015-03-08 19:59:52.195009"]]
  SQL (0.1ms)  INSERT INTO "text_messages" ("to", "from", "body", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?)  [["to", "+12345678911"], ["from", "+12345678910"], ["body", "atchcay emay ifay ouyay ancay"], ["created_at", "2015-03-08 19:59:52.198832"], ["updated_at", "2015-03-08 19:59:52.198832"]]
atchcay emay ifay ouyay ancay
   (0.6ms)  commit transaction
Completed 200 OK in 22ms (ActiveRecord: 1.4ms)
```

Now we need to add Twilio to the mix. First by adding `gem 'twilio-ruby'` to the top of our Gemfile. Now we can replace our the body of our send_outgoing method in our model with a call to Twilio. We can also add a private helper method that makes the client object that we need.

```ruby
# File: /app/models/text_message.rb

def send_outgoing
  client.messages.create(
    to: to,
    from: from,
    body: body
  )
end

private

def client
  @client ||= Twilio::REST::Client.new
end

```

Now you need to setup a Twilio account if you don't already have one. After you register go to the numbers section of their site from the top menu bar. Once there click on buy a number to purchase a Twilio phone number (make sure that you buy one that supports SMS).

Twilio provides you with an Account SID and Auth Token that your application is going to need to communicate with their API. We don't want to hard code these values in so we are going to pull them into our application from environment variables. We can do this by editing our config/secrets.yml file to pick up our Twilio credentials from environment variables using some basic ERB. We can also add a Twilio phone number here so that we can change it in our TextMessages model using environment variables. We can do this by adding a default (cause we want it in multiple environments) and updating the following code.

```erb
# File: /config/secrets.yml

default: &default
  twilio_account_sid:  <%= ENV["TWILIO_ACCOUNT_SID"] %>
  twilio_auth_token:   <%= ENV["TWILIO_AUTH_TOKEN"] %>
  twilio_phone_number: <%= ENV["TWILIO_PHONE_NUMBER"] %>

development:
  <<: *default
  secret_key_base: your-dev-secret-key

test:
  secret_key_base: your-test-secret-key

# Do not keep production secrets in the repository,
# instead read values from the environment.
production:
  <<: *default
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

We also need to tell the Twilio gem about the keys. We can do this by creating a config/initializers/twilio.rb that looks like this:

```ruby
# File: config/initializers/twilio.rb

require 'twilio-ruby'

Twilio.configure do |config|
  config.account_sid = Rails.application.secrets.twilio_account_sid
  config.auth_token  = Rails.application.secrets.twilio_auth_token
end
```

Let's also go back and change our hard coded phone number in the text messages model to use the number we are passing in as an environment variable. One way to do this is by making a private phone number method and calling that method in place of the hard coded number.

```ruby
# File: /app/models/text_message.rb

def process
  if to == phone_number
    new_sentence = body.split.map{ |x| translate(x) }
    new_sentence = new_sentence.join(" ")
    TextMessage.create(to: from, from: phone_number, body: new_sentence)
  else
    send_outgoing
  end
end

private

def phone_number
  @phone_number ||= Rails.application.secrets.twilio_phone_number
end
```

Finally we need to secure the /text url in production so that only Twilio can POST to it. We do this by adding `  config.middleware.use Rack::TwilioWebhookAuthentication, Rails.application.secrets.twilio_auth_token, '/text'
` to the end of config/environments/production.rb. _This means that when its running in production mode you will not be able to curl your URL to simulate a Twilio request_

Now that we need to connect to Twilio's servers it becomes much harder to run things locally. If you want to experiment with getting it working locally [this](https://www.twilio.com/blog/2013/10/test-your-webhooks-locally-with-ngrok.html) is an article on Twilio's site about how to do it. Instead in Part 2 of this tutorial we are going to get our application running on Heroku.

Addition resources:

* [Twilio On Rails - Integrating Twilio With Your Rails 4 App](https://www.twilio.com/blog/2014/02/twilio-on-rails-integrating-twilio-with-your-rails-4-app.html)
* [Twilio On Rails - Sending SMS MMS](https://www.twilio.com/blog/2014/10/twilio-on-rails-part-2-rails-4-app-sending-sms-mms.html)
* [Handy - An SMS based Attendance Tracking Application](https://github.com/Pitt-CSC/handy)


This last one was built by University of Pittsburgh students and is one of the best open source examples of a more complex Rails 4.2 + Active Job applications that also use Twilio (at least that I could find). Shout out to [Joel](https://github.com/jroggeman) and [George](https://github.com/georgeclaghorn) for the awesome work!
