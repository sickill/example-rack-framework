!SLIDE

# Easy building of ruby web frameworks with Rack

!SLIDE big

## Presentation plan

 * Few words about Rack
 * Framework vision
 * Implementation
 * Q&A

!SLIDE

# Few words about Rack

!SLIDE

## Rack is aaa....

!SLIDE

# "Foobar" framework vision

## MVCRails/Merb-like framework + REST

## Features we need

 * gem dependency management
 * routing
 * controllers
 * views (layouts, templates and partials)
 * basic helpers (redirects, flash[], session[], headers[])
 * models (ORM)
 * authentication

!SLIDE medium

## Application directory structure

@@@
  +app
    +controllers
    +models
    +views
  +config
  +lib
   config.ru
   Gemfile
@@@

!SLIDE

# Implementation

## Request flow

## Implementing particular features

!SLIDE big

## Request flow

@@@
    HTTP request -> Rack -> router (config.ru) -> controller
    controller (generates response) -> rack -> HTTP response
@@@

!SLIDE

## Implementing particular features

 * gem dependency management
 * routing
 * controllers
 * views (layouts, templates and partials)
 * basic helpers (redirects, flash[], session[], headers[])
 * models (ORM)
 * authentication

!SLIDE

## Prerequisites

@@@ ruby
    # config.ru
    APP_ROOT = ::File.expand_path(::File.dirname(__FILE__))
@@@

!SLIDE

# (1/7) Gem dependency management

!SLIDE

## bundler

"A gem to bundle gems"

[github.com/carlhuda/bundler](http://github.com/carlhuda/bundler)

!SLIDE big

@@@ ruby
    # Gemfile

    source "http://gemcutter.org"
    gem "rack"

    # config.ru

    require "bundler"
    Bundler.setup
    Bundler.require
@@@

!SLIDE

# (2/7) Routing

!SLIDE

## Usher

"Pure ruby general purpose router with interfaces for rails, rack, email or choose your own adventure"

[github.com/joshbuddy/usher](http://github.com/joshbuddy/usher)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "usher"
    
    # config.ru
    
    require APP_ROOT / "config" / "router.rb"

    run Foobar::Router
@@@

!SLIDE medium

@@@ ruby
    # config/router.rb

    module Foobar
      Router = Usher::Interface.for(:rack) do
        get('/').to(HomeController.action(:welcome)).name(:root) # root URL
        get('/:controller(/)').to(lambda { |env| BaseController.dispatch(env, :index) }) # index
        get('/:controller/{:id,\d+}(/)').to(lambda { |env| BaseController.dispatch(env, :show) }) # show
        get('/:controller/new(/)').to(lambda { |env| BaseController.dispatch(env, :new) }) # new
        post('/:controller(/)').to(lambda { |env| BaseController.dispatch(env, :create) }) # create
        put('/:controller/{:id,\d+}(/)').to(lambda { |env| BaseController.dispatch(env, :update) }) # update
        delete('/:controller/{:id,\d+}(/)').to(lambda { |env| BaseController.dispatch(env, :destroy) }) # destroy
        add('/login').to(SessionController.action(:login)).name(:login) # login
        get('/logout').to(SessionController.action(:logout)).name(:logout) # logout
        default ExceptionsController.action(:not_found) # 404
      end
    end
    
    env["usher.params"] == { :controller => "users", :id => 666 }
@@@

!SLIDE

# (3/7) Controllers

!SLIDE

założenia

!SLIDE small

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        def self.dispatch(env, action_name)
          controller_name = env["usher.params"][:controller]
          name = controller_name.camel_case + "Controller"
          controller = Object.const_get(name)
          action = controller.action(action_name)
          action.call(env)
        end
        
        def self.action(name)
          lambda do |env|
            env['x-rack.action-name'] = name
            self.new.call(env)
          end
        end
        
        def call(env)
          @request = Rack::Request.new(env)
          @response = Rack::Response.new
          resp_text = self.send(env['x-rack.action-name'])
          @response.write(resp_text)
          @response.finish
        end
      end
    end
@@@

!SLIDE big

@@@ ruby
    # config.ru
    
    require APP_ROOT / "lib" / "base_controller.rb"
    Dir[APP_ROOT / "app" / "controllers" / "*.rb"].each do |f|
      require f
    end
@@@

!SLIDE

# (4/7) Views

!SLIDE

## Tilt

"Generic interface to multiple Ruby template engines"

[github.com/rtomayko/tilt](http://github.com/rtomayko/tilt)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "tilt"
    
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        include Rendering
        ...
      end
    end
@@@

!SLIDE small

@@@ ruby
    # lib/rendering.rb

    module Foobar
      module Rendering
        def get_templates_dir
          "#{APP_ROOT}/app/views/#{controller_name}"
        end
        
        def render(text=nil)
          layout_path = "#{APP_ROOT}/app/views/layouts/application.html.erb"
          if text.nil? || text.is_a?(Symbol)
            template = text || @request.env['x-rack.action-name']
            template_path = "#{get_templates_dir}/#{template}.html.erb"
            text = Tilt.new(template_path).render(self)
          end
          Tilt.new(layout_path).render(self) do
            text
          end
        end
        
        def partial(template, opts={})
          template_path = "#{get_templates_dir}/_#{template}.html.erb"
          locals = { template.to_sym => opts.delete(:with) }.merge!(opts)
          Tilt.new(template_path).render(self, locals)
        end
      end
    end
@@@

!SLIDE big

@@@ rhtml
    <h2>Users list</h2>
    
    <% @users.each do |user| %>
      <%= partial :user, :with => user, :comments => true %>
    <% end %>
@@@

!SLIDE

# (5/7) Basic helpers

!SLIDE

## Basic stuff like:

 * session[]
 * flash[]
 * redirect_to
 * link_to
 * url(:foo)
 * headers[]
 
!SLIDE

## rack-contrib

"Contributed Rack Middleware and Utilities"

[github.com/rack/rack-contrib](http://github.com/rack/rack-contrib)

## rack-flash

"Simple flash hash implementation for Rack apps"

[nakajima.github.com/rack-flash](http://nakajima.github.com/rack-flash/)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "rack-flash"
    gem "rack-contrib", :require => 'rack/contrib'
    
    # config.ru
    
    use Rack::Flash
    use Rack::Session::Cookie
    use Rack::MethodOverride
    use Rack::NestedParams
@@@

!SLIDE big

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        include Rendering
        include BasicHelpers
        ...
      end
    end
@@@

!SLIDE verysmall

@@@ ruby
    # lib/basic_helpers.rb
    
    module Foobar
      module BasicHelpers
        def controller_name
          self.class.to_s.snake_case.gsub('_controller', '')
        end
      
        def status=(code)
          @response.status = code
        end
        
        def headers
          @response.header
        end
        
        def redirect_to(url)
          self.status = 302
          headers["Location"] = url
          "You're being redirected"
        end
        
        def session
          @request.env['rack.session']
        end
        
        def flash
          @request.env['x-rack.flash']
        end
    
        def link_to(label, url=nil)
          url ||= label
          %(<a href="#{url}">#{label}</a>)
        end
        
        def url(name, opts={})
          Router.generate(name, opts)
        end
      end
    end
@@@

!SLIDE

# (6/7) Models

!SLIDE

## DataMapper

"DataMapper is a Object Relational Mapper written in Ruby. The goal is to create an ORM which is fast, thread-safe and feature rich."

[datamapper.org](http://datamapper.org/)

!SLIDE medium

@@@ ruby
    # Gemfile
    
    gem "dm-core"
    gem "dm-..."
    
    # app/models/user.rb

    class User
      include DataMapper::Resource
      
      property :id, Serial
      property :login, String, :required => true
      property :password, String, :required => true # don't forget to encrypt in real app
    end
    
    # config.ru

    Dir[APP_ROOT / "app" / "models" / "*.rb"].each { |f| require f }
@@@

!SLIDE

# (7/7) Authentication

!SLIDE

## Warden

"General Rack Authentication Framework"

[github.com/hassox/warden](http://github.com/hassox/warden)

!SLIDE medium

@@@ ruby
    # Gemfile
    
    gem "warden"
    
    # config.ru
    
    use Warden::Manager do |manager|
      manager.default_strategies :password
      manager.failure_app = ExceptionsController.action(:unauthenticated)
    end
@@@

!SLIDE medium

@@@ ruby
    # lib/warden.rb
    
    Warden::Manager.serialize_into_session { |user| user.id }
    Warden::Manager.serialize_from_session { |key| User.get(key) }
    
    Warden::Strategies.add(:password) do
      def authenticate!
        u = User.authenticate(params["username"], params["password"])
        u.nil? ? fail!("Could not log in") : success!(u)
      end
    end
@@@

!SLIDE big

@@@ ruby
    # app/models/user.rb
    
    class User
      include DataMapper::Resource
      ...
      
      def self.authenticate(login, password)
        first(:login => login, :password => password)
      end
    end
@@@

!SLIDE big

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        include Rendering
        include BasicHelpers
        include Authentication
        ...
      end
    end
@@@

!SLIDE medium

@@@ ruby
    # lib/authentication.rb
    
    module Foobar
      module Authentication
        def authenticate!
          @request.env['warden'].authenticate!
        end
        
        def logout!(scope=:default)
          @request.env['warden'].logout(scope)
        end
        
        def current_user
          @request.env['warden'].user
        end
      end
    end
@@@

!SLIDE

# That's it!

Code & slides available at: [github.com/sickill/example-rack-framework](http://github.com/sickill/example-rack-framework)

!SLIDE

# Questions?