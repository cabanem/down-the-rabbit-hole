# Ruby & Rails
## Ruby's Core Philosphy and Mechanics
### The Object Model - Everything is an Object
- In Ruby, **everything** is an object (even classes)
  ```ruby
  5.class             # => Integer
  Integer.class       # => Class
  Class.class         # => Class
  Class.superclass    # => Module
  Module.superclass   # => Object
  Object.superclass   # => BasicObject
  ```
- The object model enables powerful metaprogramming
- Rails uses this for ActiveRecord attributes, so when you do `User.find(1).eamil`, Rails creates the methods dynamically. 
  ```ruby
  Class User
    # This code runs at class definition time
    puts "Defining User Class"

    # We can programatically define methods
    [:name, :email, :age].each do |attribuate|
      define_method(attribute) do
        instance_variable_get("@#{attribute}")
      end

      define_method("#{attribuate}=") do |value|
        instance_variable_set("@#{attribuate}", value)
      end
    end
  end
  ```
### Method Lookup Chain
- To understand Rails, you need to understand Ruby's method resolution order.
- Rails callbacks use method chain manipulation
```ruby
module Trackable
  def track
    puts "Tracking in #{self.class}"
  end
end

module Auditable
  def track # Note duplication of method name
    puts "Auditing in #{self.class}"
    super # this calls next in chain
  end
end

class User
  include Trackable # ++ ancestry chain
  prepend Auditable # add BEFORE class in chain

  def track
    puts "User tracking"
    super
  end
end

User.new.track
# Output
# Auditing in User (prepend first)
# User tracking (class method)
# Tracking in User (incl module)

# Check the chain
User.ancestors
# => [Auditable, User, Trackable, Object, Kernel, BasicObject]

# This is how Rails' callbacks work
class ApplicationController < ActionController::Base
  before_action :authenticate! # method chain manipulation
end
```

## Rails' Architecture
### How Rails Auto-Loads
- Rails uses Zeitwerk for autoloading (Rails > 6)
  - Zeitwerk maps file paths to constants
  - When Ruby can't find a given constant, Rails uses ActiveRecord to resolve
-

### Request Cycle
1. Rack Middleware Stack (bottom)
   - Middleware processes request before it hits Rails
     `config.middleware.use Rack::Attack` (rate limiting)
     `config.middleware.use Rack::Cors`   (CORS headers)
   - Simplified (default) middleware
     - Rack::Sendfile (serve files)
     - ActionDispatch::Static (static assets)
     - Rack::Lock (thread safety)
     - Rack::Runtime (X-Runtime header)
     - Rack::MethodOverride (REST verbs)
     - ActionDispatch::RequestId
     - Rails::Rack::Logger
     - ActionDispatch::ShowExceptions
     - ActionDispatch::DebugExceptions
     - ActionDispatch::RemoteIp
     - ActionDispatch::Callbacks (before/after)
     - ActionDispatch::Cookies
     - ActionDispatch::Session
     - ActionDispatch::Flash
     - Rack::Head
     - Rack::ConditionalGet
     - Rack::ETag
3. Router matches and dispatches
   - Creates routes with REST conventions
5. Controller is instantiated and action is dispatched
   - Before the action runs:
     - New controller instance is created per request
     - Instance variables are request scoped
     - Callbacks run in order
   - Rails provides:
     - params   `HashWithIndifferentAccess`
     - request  `ActionDispatch::Request`
     - response `ActionDispatch::Response`
     - session  `ActionDispatch::Session`
     - cookies  `ActionDispatch::Cookies`

## Comparisons
### Language Comparison (Ruby vs. ....)
#### Memory Model and Performance
##### Ruby (MRI/cRuby)
- Ruby uses
  - Global Interpreter Lock (GIL)
  - Mark-and-sweep garbage collection
  - Copy-on-write fork() optimization
- Certain coding practices can impact performance.
  - Creating many temporary objects can negatively impact performance
    ```ruby
    def process_large_dataset(data)
      data.map { |item| item.to_s }     # this creates an array
          .select { |s| s.length > 5 }  # so does this
          .map { |s| s.upcase }         # this too
          .join(", ")                   # creates a string
    end
    ```
  - Try to minimize object creation
    ```ruby
    def process_large_dataset(data)
      result = []
      data.each do |item|
        str = item.to_s
        result << str.upcase if str.length > 5
      end
      result.join(", ")
    end
    ```
  - For maximum efficiency, use `Enumerator::Lazy` for large datasets
    ```ruby
    def process_large_dataset(data)
      data.lazy
          .map(&:to_s)
          .select { |s| s.length > 5 }
          .map(&:upcase)
          .first(1000)  # Only processes what's needed
    end
    ```
##### Node.js (V8)
- V8 optimizes with JIT compilation

##### Python
- Python list comprehensions tend to be faster

##### Java
- Java has the Streams API and can parallelize workflows

#### Concurrency
##### Ruby
- Ruby's concurrency model is different from other languages
1. Threads
   - Not ideal for CPU-intensive workloads (limited by GIL)
     ```ruby
     threads = 10.times.map.do |i|
       Thread.new do # GIL prevents parallelization here
         calculate_prime(1_000_000) 
       end
     end
     threads.each(&:join)
     ```
   - Effective for I/O-bound workloads (GIL is released during I/O)
     ```ruby
     require 'net/http'
     urls = ['http://api1.com', 'http://api2.com', 'http://api3.com']
     threads = urls.map do |url|
       Thread.new do # GIL is realeased during I/O!
         response = Net::HTTP.get(URI(url))
         process_response(response)
       end
     end
     results = thread.map(&:value)
     ```
2. Fiber-based
  - Ruby 3+
  - Async/await pattern
  - Non-blocking i/o
  ```ruby
  Async do |task|
    responses = urls.map do |url|
      task.async do
        fetch_url(url)
      end
    end.map(&:wait)
  end
  ```

3. Ractor
  - Ruby 3+ (experimental)
  - Similar to Erlang/Elixir actors
  - Realize true parallelism
  ```ruby
  ractor = Ractor.new do
    loop do
      msg = Ractor.receive
      Ractor.yield(process_message(msg))
    end
  end
  ```
##### Go
##### Node.js
##### Python

### Framework Comparison (Rails vs. ...)
#### ORMs
##### Rails - ActiveRecord
##### Django ORM
##### Express/Sequelize
##### Spring Data JPA

#### Request Handling Philosopy
##### Rails
##### Express.js
##### Django
##### Spring Boot

#### Framework Philosophy Comparison
##### Rails "Omakase"
- Rails makes decisions for you - "There's a Rails way to do things"
- Convention: 1 file, 1 class, named properly
  - `app/models/user.rb` contains class `User`
  - `app/controllers/users_controller.rb` contains class `UsersController`
- Framework provides everything
  - Dozens of methods out of the box

##### Express "Minimalist"
##### Django "Batteries included, but be explicit"
---

## Patterns in Rails
### Service Objects
### Query Optimization
### Proper Use of Concerns

## Advanced Patterns and Techniques
### Metaprogramming
### Background Jobs and Async Processing
### Database and Performance Optimization

## Performance and Scaling Strategies

---
# Key Takeaways
- **Ruby Strengths**
  - Developer happiness - Designed for programmer productivity
  - Metaprogramming - Powerful but dangerous
  - DSL creation - Natural language-like APIs
  - Convention over configuration - Less boilerplate

- **Ruby Weaknesses**
  - Performance - Slower than compiled languages
  - Concurrency - GIL limits true parallelism
  - Memory usage - Higher than Go, Rust, or C++
  - Type safety - Dynamic typing can hide bugs

- **Rails Strengths**
  - Rapid development - Fastest for CRUD apps
  - Ecosystem - Gems for everything
  - Conventions - Team onboarding is easier
  - Full-stack - Everything included

- **Rails Weaknesses**
  - Magic - Hard to debug when conventions break
  - Monolithic tendency - Harder to microservice
  - Performance ceiling - Not for ultra-high-performance needs
  - Learning curve - Conventions must be learned

