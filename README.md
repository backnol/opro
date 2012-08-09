## Opro

A production ready Rails Engine that turns your app into an [Oauth2](http://oauth.net/2/) Provider.

  * [Demo OAuth Provider app with Opro](http://opro-demo.herokuapp.com/) on Heroku
  * [Built in Opro docs](http://opro-demo.herokuapp.com/oauth_docs)

## Why would I use this?

Lets say you've built a Rails app, awesome. Now you want to build a mobile app on say, the iPhone... cool. You start throwing around `#to_json` like nobody's business, but then you realize you need to authenticate users somehow. "Basic Auth!!", you exclaim, but then you realize that's not the most secure solution. You also realize that some users already signed up with Facebook & Twitter so they don't have a username/password combo. What ever shall you do?

Wouldn't it be great if we could have a token exchange where the user goes to a mobile web view and grants permission, and then we return back an auth token just like the big boys (Facebook, Twitter, *cough* Foursquare *cough*). With Opro, we can add this functionality pretty easily. We'll use your existing authentication strategy and provide some integration end points for your clients to use out of the box.


## Install

Gemfile

```ruby
    gem 'opro'
```

Then run

```shell
    $ bundle install
```

and don't forget

```shell
    $ rails g opro:install
```

This will put a file in `initializers/opro.rb` and generate some migrations, and add `mount_opro_oauth` to your routes.


Now we're ready to migrate the database

```shell
    $ rake db:migrate
````

This will add `Opro::Oauth::AuthGrant` and `Opro::Oauth::ClientApp` to your database. An iPhone app would need to register for a `client_id` and `client_secret` before using Oauth as a ClientApp. Once created they could get authorization from users by going through the oauth flow, thus creating AuthGrants. In other words a ClientApp has many users through AuthGrants.

## Setup

Go to `initializers/opro.rb` and configure your app for your authentication scheme, if you're not using devise see "Custom Auth" below.

```ruby
      Opro.setup do |config|
        config.auth_strategy = :devise
      end
```


Now in your controllers you can allow OAuth access using the same syntax of the rails `before_filter`

```ruby
      class UsersController < ApplicationController
        allow_oauth!  :only => [:show]
      end
```


You can also disallow OAuth on specific actions. Disallowing will always over-ride allowing.


```ruby
      class ProductsController < ApplicationController
        disallow_oauth!   :only => [:create]
      end
```

By default all OAuth access is blacklisted. To whitelist all access, add `allow_oauth!` to your `ApplicationController` (this is not recommended). The best practice is to add `allow_oauth!` or `disallow_oauth` to each and every controller.

That should be all you need to do to get setup, congrats you're now able to authenticate users using OAuth!!


## Use it

Opro comes with built in documentation, so if you start your server you can view them at [http://localhost:3000/oauth_docs](http://localhost:3000/oauth_docs). Or you can [view the guide](http://opro-demo.herokuapp.com/oauth_docs) on the example app. This guide will walk you through creating your first OAuth client application, giving access to that app as a logged in user, getting an access token for that user, and using that token to access the server as an authenticated user!


# Advanced Setup

Opro is simple by default, but easily configurable for a number of common use cases. Check out the options below.


## Custom Auth

If you're not using devise you can manually configure your own auth strategy. In the future I plan on adding more auth strategies, ping me or submit a pull request for your desired authentication scheme.


      Opro.setup do |config|
        config.login_method             { |controller, current_user| controller.sign_in(current_user, :bypass => true) }
        config.logout_method            { |controller, current_user| controller.sign_out(current_user) }
        config.authenticate_user_method { |controller| controller.authenticate_user! }
      end

## Permissions

When a user auth's with a client they automatically are granting read permission to any action that you `allow_oauth!`. Read only clients are restricted to using GET requests. By default Opro will ask users for write permission on a client by client application. Client apps with `:write` permission can use all HTTP verbs including POST, PATCH, PUT, DESTROY on any url you whitelist using `allow_oauth!`.


### Custom Permissions

To remove write permissions comment out this line in the Opro initializer:

      config.request_permissions = [:write]

You can add custom permissions by adding to the array:

      config.request_permissions = [:write, :email, :picture, :whatever]

You can then restrict access using the custom permissions by calling `require_oauth_permissions` which takes the same arguments as `before_filter`

      require_oauth_permissions :email, :only => :index

You can also skip permissions using `skip_oauth_permissions`. By default permissions will just check to see if a client has the permission, and will allow the action if it is present. If you want to implement custom permission checks you can write custom methods using the pattern `oauth_client_can_#{permission}?` for example if you were restricting the `:email` permission, you would create a method.

      def oauth_client_can_email?
        # ...
      end

The result is expected to be true or false.


## Refresh Tokens

For added security you can require access_tokens be refreshed by client applications. This will help to mitigate risk of a leaked access_token, and enable an all around more secure system. This security comes at a price however, since implementing the `refresh_token` functionality in a client can be more difficult.

By default refresh tokens are enabled, you can disable them in your application and set the timeout period of the tokens by adding this line to your configuration.

    config.require_refresh_within = 1.month




## Password Token Exchange

If a client application has a user's password and username/email they can exchange these for a token. This is much safer than storing username and password on a local device, but does not offer the traditional OAuth "Flow". Because of this all available permissions will be granted to the client application. If you want to disable this feature you can set the configuration below to false:

    config.password_exchange_enabled = true

If you have this feature enabled you can further control what applications can use the feature. Some providers may wish to have "Blessed" client applications that have this ability while restricting all other clients. To accomplish this you can create a method in your ApplicationController called `oauth_valid_password_auth?` that accepts a client_id and client_secret, and returns a true or false based on whether that application can use password auth

    def oauth_valid_password_auth?(client_id, client_secret)
      BLESSED_APP_IDS.include?(client_id)
    end


 If you are using this password functionality without a supported authorization engine (like devise), you will need to add an additional method that supports validating whether or not a user's credentials are valid. The method for this is called `find_user_for_auth` and accepts a controller and the parameters. The output is expected to be a user. Add this to your config like you did to the other required methods in the Custom Auth section.

    config.find_user_for_auth do |controller, params|
      # user = User.find(params[:something])
      # return user.valid_password?(params[:password]) ? user : false
    end

If you're authenticating exchanging something other than a password (such as a facebook auth token) client's can still enable this functionality by setting `params[:grant_type] == 'password'` in their initial request. You can then use `find_user_for_auth` method from above and implement your custom behavior. You can call `find_user_for_auth` multiple times and the application will try calling each auth method in order. It is suggested that you return from this block early if the params are missing a vital key like this:


    config.find_user_for_auth do |controller, params|
      return false if params[:fb_token].blank?
      User.where(:fb_token => params[:fb_token]).first
    end


## Rate Limiting

If your API becomes a runaway success and people starte abusing your api, you might chose to limit the rate that client applications can access your API. It is common for popular read only API's to have an hourly, or daily rate limit to help prevent abuse. If you want this type of functionality you can use Opro's built in hooks, one to record the number of times a client application has accessed your api. And another to let the application know if the Client app has gone over it's alloted rate.

To record the number of times an application has accessed your site add this method to your ApplicationController:

    def oauth_client_record_access!(client_id, params)
      # implement your rate counting mechanism here
    end

Then to let our server know if a given client has reached add this method, the output is expected to be true if the client has gone over their limit, and false if they have not:

    def oauth_client_rate_limited?(client_id, params)
      # implement your own custom rate limiting logic here
    end

Rate limited clients will receive an "unsuccessful" response to any query with a message letting them know they've been rate limited. Using redis with a rotating key generator based on (hour, daty, etc.) is one very common way to count rate, and implement the rate limits. Since there are so many different ways to implement this, we decided to give you a blank slate and implement it however you like. The default is that apps are not rate limited, and in general unlimited API access is the way to go, but if you do find abusive behavior you can always easily add in a rate limit.


## Configurable Authorization Header

By default oPRO allows clients to send their authorization token in a header. For example someone using an auth token of `9693accessTokena7ca570bbaf` could set the `Authorization` header in a request like this:

    $ curl -H "Authorization: Bearer 9693accessTokena7ca570bbaf" "http://localhost:3000/oauth_test/show_me_the_money"

By default oPRO will accept `Bearer` and `token` in the authorization header, but if your client needs to send a custom auth header, you can add a custom extra regular expression to parse parse and return the token. For example if a client was setting the auth header like this:

    $ curl -H "Authorization: cUsTomAuthHeader 9693accessTokena7ca570bbaf" "http://localhost:3000/oauth_test/show_me_the_money"

You could pull out the auth token using this regular expression `/cUsTomAuthHeader\s(.*)/`. If you're not great with regular expressions I highly recommend using [Rubular](http://rubular.com) to test regex matches. It is very important that we are "capturing" data with in between the `()` characters. The data returned inside of the parens are expected to be the auth token with no spaces or special characters such as new lines or quotes. To set parse this auth header in oPRO, you can specify the `header_auth_regex` in an initializer like this:


      Opro.setup do |config|
        config.auth_strategy = :devise

        config.header_auth_regex = /cUsTomAuthHeader\s(.*)/
      end

Now when a client sends your custom auth header it will be parsed correctly. Custom authorization headers should not be used for security through obscurity. They may be exposed in the docs or tests in a later iteration of oPRO, if you have strong feelings against this, then please open a pull request or send me a message stating your case.



## Assumptions

* You have a user model and that is what your authenticating
* You're using Active::Record

## About

If you have a question file an issue or, find me on the Twitters [@schneems](http://twitter.com/schneems). Another good library for turning your app into an OAuth provider is [Doorkeeper](https://github.com/applicake/doorkeeper), if this project doesn't meet your needs let me know why and use them :)

This project rocks and uses MIT-LICENSE.