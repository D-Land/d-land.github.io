---
layout: post
title:  "Part Two: Rails 4.2 + Twilio + Heroku"
date:   2015-3-9
---

This is part two of three on building an app that utilizes Rails 4.2 and ActiveJob along with Twilio and Heroku.

<p class="intro"><span class="dropcap">H</span>eroku has requirements for applications that run on its service. Due to these requirements we are going to have to modify and add a few things to our project and in the process we are going to try to keep our local development environment as close to Heroku as possible.

The first requirement is that applications use PostgreSQL. Rails by default uses sqlite3 for its databases. To install Postgres locally on a Mac I suggest the [PostgreSQL app](http://postgresapp.com/). If you are on a different platform please see the [PostgreSQL website](http://www.postgresql.org/download/). Now to get our application to use Postgres we need to make a few changes. First, let's switch out the sqlite3 gem by replacing it with `gem 'pg'` in our Gemfile. Next we need to change the database.yml file to specify that we're using PostgreSQL.

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: 5

development:
  <<: *default
  database: anslatortry_development
  host: localhost

test:
  <<: *default
  database: anslatortry_test
  host: localhost

production:
  <<: *default
```

You'll want to run `bundle exec rake db:setup` after installing the PostgreSQL app and changing your database.yml file if you are trying to keep the app running locally.

Heroku currently recommends using the puma web server (there are many other options that will work too, like unicorn). We are going to add `gem 'puma'` to the top of our Gemfile. Heroku relies on a Procfile that you provide to know what processes to start and with what parameters. To let it know we want to run a Puma server we need to add a file named Procfile to the project root directory and add the line `web: bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}` to that file. This lets Heroku know that you want to startup puma as the web server process and not the default which is not a good option in a production environment.

We can also use this in method of starting up processes locally by using a tool called Foreman. Add `gem 'foreman'` to the :development and :test groups of the Gemfile. Now we can run `foreman start` to start Puma locally.

Now we are ready to deploy to Heroku. If you do not have a [Heroku](https://heroku.com/) account go make one now and if you do not have the [Heroku tool belt](https://toolbelt.heroku.com/) installed do that now as well. Once you have both of those done, if you haven't logged into Heroku from the command-line before you will want to run `heroku login`. Now that you're logged in let's create a new Heroku project by going to our Rails project root directory and running `heroku create`. To deploy make sure you have everything committed to your master branch in git and run `git push heroku master` and our code will deploy! After it finishes we need to run `heroku run rake db:migrate` to setup our databases. You can run `heroku logs --tail` in another terminal to see the logs of what your project is doing live.

Our application needs to have access to the environment variables we mentioned earlier. We can add these through the Heroku web interface. Go to the Heroku site and navigate to your project project. Go to the settings tab at the top and then reveal config vars. Here you can add TWILIO\_ACCOUNT\_SID, TWILIO\_AUTH\_TOKEN, and TWILIO\_PHONE\_NUMBER (make sure to include the +1 like this "+15558675309") to the list of variables. The values for these variables are located on the account page of the Twilio website when you are logged in.

The final step is to configure your phone number to POST to your-app-name.herokuapp.com/text. You do this by going to the numbers page on the Twilio website. Click on the phone number you are using for this project and under messaging change the request URL to be your-app-name.herokuapp.com/text. (make sure to change you-app-name to the actual name Heroku gave your app or that name you gave it if you specified one)

Now send a text from your phone that says "catch me if you can" and if you've been following along you should get back a text that says "atchcay emay ifay ouyay ancay".

In part 3 we are going to add in ActiveJob. The calls to Twilio can take up to one second each which makes it a perfect candidate for a simple task that can be done by a background worker to free up the web server.

Additional Resources:
* [Twilio On Rails - Integrating Twilio With Your Rails 4 App](https://www.twilio.com/blog/2014/02/twilio-on-rails-integrating-twilio-with-your-rails-4-app.html)
* [Twilio On Rails - Sending SMS MMS](https://www.twilio.com/blog/2014/10/twilio-on-rails-part-2-rails-4-app-sending-sms-mms.html)
* [Getting Started with Rails 4.x on Heroku](https://devcenter.heroku.com/articles/getting-started-with-rails4)
