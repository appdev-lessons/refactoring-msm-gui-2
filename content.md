# Refactoring Must See Movies GUI with Association Accessor Helper Methods

- Note: Validations were added, so specs are slightly different

## Objective

Our goal is to keep Must See Movies GUI (`msm-gui`) working the same way that it was after we finished building it; we're not going to add much. Therefore, we'll use the same target as before:

[https://msm-gui.matchthetarget.com/](https://msm-gui.matchthetarget.com/)

The project can be loaded here:

LTI{Load Refactoring Must See Movies GUI 1 assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(10)[Refactoring Must See Movies GUI 1 Project]

Our starting point code for this project, `refactoring-msm-gui-1`, is one possible solution for `msm-gui`. But we're going to make the code much more modular and re-usable, while keeping the functionality exactly the same. How? By **defining methods** to encapsulate our querying logic.

First, you should read through the starting point code and compare it to your own solution to `msm-gui`. Be sure to `rake sample_data` and `bin/dev` so that you can click through the application, verify that it's working, and read the server log.
  
Are there any differences between the starter code and your own solution to `msm-gui`? You will most likely find at least one or two differences. What are they doing? Practice _reading_ the code and reasoning your way through it, line by line; explain it to yourself, or to your rubber ducky. Developers read far more code than we write.

Does any part of the code puzzle you?

Go ahead, I'll wait!

## The problem: queries all over the place

Right now, there are several places in our application where we have a movie and we want to display something about the director of the movie.

For example, in `app/views/movie_templates/show.html.erb`, we have an instance of `Movie` in a variable called `@the_movie`, and we want to display the name of the director. So, first, we use the value in the attribute `@the_movie.director_id` to look up a matching record in the directors table:

```erb
<% the_id = @the_movie.director_id %>

<% matching_directors = Director.where({ :id => the_id }) %>
    
<% the_director = matching_directors.at(0) %>
```

Then, finally, we can get the value in the `name` or `dob` or `bio` or whatever other attribute we care about from that instance of `Director`:

```erb
<%= the_director.name %>
```

This same basic logic — given a movie, use its `director_id` attribute to look up a row in the directors table — is repeated in several places: the movies index page, an actor's filmography, etc. If we were to continue building this application out more, no doubt we would be doing this task many more times; nearly everywhere a movie appears, or in the `rails console`, or in a rake task, along with information about the movie itself (like `title`, `year`, etc), we will want to display information about the associated director (like `name`, etc).

What a pain to have to repeat this query,

```ruby
Director.where({ :id => @the_movie.director_id }).at(0)
```

, every single time I just want some info about `@the_movie`'s director!

## The solution: encapsulate queries in methods

Let's make life easier on our future selves and on all our future teammates: **let's define a nicely named instance method** that will do this work and that we can call on any instance of `Movie` whenever we want to.

The method will encapsulate what we've been doing repetitively over and over until now: it will talk to the `Director` class, retrieve the record from the directors table corresponding to the movie's `director_id`, and return an instance of `Director` to us. Then, we can use `Director` attribute accessors to get whichever column values we are interested in about the director.

Since we named our foreign key column in the movies table `director_id`, we automatically got an attribute accessor method on `Movie` called `.director_id` from ActiveRecord which returns an `Integer`.

What shall we call our new method, which will _use_ the `Integer` returned by `.director_id` to retrieve an instance of `Director` and return it?

How about `.director`?

Here's what I wish we could do. If we have an instance of `Movie` inside a variable called `@the_movie`, instead of:

```erb
<% the_id = @the_movie.director_id >

<% matching_directors = Director.where({ :id => the_id }) %>
    
<% the_director = matching_directors.at(0) %>

<%= the_director.name %>
```

I want to be able to do:

```erb
<% the_director = @the_movie.director %>

<%= the_director.name %>
```

Or, if we're okay with chaining a couple of methods on the same line, we could even do:

```erb
<%= @the_movie.director.name %>
```

Ahhhhh! Wouldn't that be nice? Having a method at our fingertips called `.director` that we can call on any instance of `Movie` whenever we want to, and it would know how to perform the database query to find the associated record and return an instance of `Director` to us? It'd be like an _association_ accessor, the way we have convenient _attribute_ accessors for our column values.

Unfortunately, if you embed `<%= @the_movie.director %>` in `app/views/movie_templates/show.html.erb` right now and visit the details page of a director, you'll get a big ol' `"undefined method 'director' for #<Movie:0x00007faadebb9e88>"` error. If we want handy "association accessors" like that, we'll have to roll up our sleeves and invent them ourselves!

## Instance method review

### Naming the method

If you feel very confident about [defining instance methods](https://learn.firstdraft.com/lessons/78-ruby-intro-our-own-classes#defining-instance-methods), then you can skip forward to the [Defining "association accessors"](#defining-association-accessors){:target="_self"} section. Otherwise, read on.

Recall that, in Ruby, we use the `def` keyword within the `class` definition to add new methods to a class. Here, we're adding the `say_hi` instance method within the definition of the `Person` class:

```ruby
class Person
  def say_hi
    return "Hello!"
  end
end
```

Since the `say_hi` comes immediately after the `def` keyword, Ruby knows we are defining an _instance-level_ method. 

<aside markdown="1">
If, instead of `def say_hi`, we had said `def Person.say_hi`, then we would have defined a _class-level_ method.
</aside>

### Returning a value

Every method's main job is _returning_ a value, which is what it gets substituted by as the program is being evaluated. So, the above class would now work like this:

```ruby
class Person
  def say_hi
    return "Hello!"
  end
end

p = Person.new
hello = p.say_hi
hello_upcase = hello.upcase

pp hello
pp hello_upcase
```
{: .repl #say_hi title="Say hi" points="1"}

Our programs are essentially just a long sequence of methods being called on the return values of previous methods, until we arrive at our desired output.

### Composing methods to build more powerful methods

Usually, we start by defining small, simple methods; but then we combine these methods into ever more powerful ones.

#### Attribute accessors

Some of the simplest but most useful methods are _attribute accessors_. In our pure Ruby days, we had to declare them ourselves (although more recently ActiveRecord has been doing it for us automatically based on column names):

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)
end
```

Each call to `attr_accessor` gives us two instance methods: one for storing a value in an instance of the class, and one for retrieving the value. So now, our `Person` instances can remember their first names and last names:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"
pp rb

bp = Person.new
bp.first_name = "Ben"
bp.last_name = "Purinton"
pp bp

jw = Person.new
jw.first_name = "Jelani"
jw.last_name = "Woods"
pp jw
```
{: .repl #say_my_name_1 title="Say my name, 1" points="1"}

### Using existing instance methods when defining new instance methods

Now, what if we wanted to display each person's full name? We could do this:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"

bp = Person.new
bp.first_name = "Ben"
bp.last_name = "Purinton"

jw = Person.new
jw.first_name = "Jelani"
jw.last_name = "Woods"

pp rb.first_name + " " + rb.last_name
pp bp.first_name + " " + bp.last_name
pp jw.first_name + " " + jw.last_name
```
{: .repl #say_my_name_2 title="Say my name, 2" points="1"}

But wouldn't it be nice if we had a nicely named method we could call instead, like `.full_name`?

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"

pp rb.full_name
```
{: .repl #say_my_name_3 title="Say my name, 3" points="1"}

As expected, an error, because we haven't defined `full_name`. Let's define the method, to get one step closer to our goal:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)

  def full_name
    return "Raghu Betina"
  end
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"

bp = Person.new
bp.first_name = "Ben"
bp.last_name = "Purinton"

jw = Person.new
jw.first_name = "Jelani"
jw.last_name = "Woods"

pp rb.full_name
pp bp.full_name
pp jw.full_name
```
{: .repl #say_my_name_4 title="Say my name, 4" points="1"}

Well, it's progress, I suppose. At least we resolved the `undefined method 'full_name' for #<Person:0x00007fe0c24a6eb0>` issue. But, we want each instance to use it's _own_ first name and last name attributes to put together it's full name. How can we author the `.full_name` method to do that?

If we changed the definition of the method to the following:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)

  def full_name
    assembled_name = bp.first_name + " " + bp.last_name
    
    return assembled_name
  end
end
```

Would it help? No, it wouldn't. Give it a try and see what the error message says.

```
undefined local variable or method `bp' for #<Person:0x00007fe0c24a6eb0> (NameError)
```

We can't use the `rb`, `bp`, or `jw` local variables when we _define_ the instance method. When we _author_ the method, we have no idea what the _invokers_ of the method are going to name their variables next month or next year when they _use_ the method, or whether they are going to create variables at all! Perhaps they are just going to chain this method on to the end of another method.

So, when we're authoring the `.full_name` method, we need some way to refer to whichever instance of the `Person` class the `.full_name` method is going to be called upon _in the future_. Ruby gives us a way: the `self` keyword.

#### The self keyword

You can think of `self` as a special variable that holds whatever object is at the forefront of Ruby's mind when it is evaluating a program. Whatever line of code it is reading, whatever object is it working on at the moment, that is what `self` contains.

For our purposes, here's what matters right now: when we're defining instance methods, when writing code _within_ the instance method definition, `self` represents the instance of the class that the method will be called upon in the future. Formally, this object is known as the **receiver** of the **message** that is the method invocation.

That's great, because since we almost always need to use other, pre-existing instance methods while defining new instance methods, we need a way to _refer_ to the receiver so that we can call those other methods on it. `self` is that way.

So, `self` is what will allow us to use the pre-existing `.first_name` and `.last_name` attribute accessor methods to build up our handy `.full_name` method:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)

  def full_name
    assembled_name = self.first_name + " " + self.last_name

    return assembled_name
  end
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"

bp = Person.new
bp.first_name = "Ben"
bp.last_name = "Purinton"

jw = Person.new
jw.first_name = "Jelani"
jw.last_name = "Woods"

pp rb.full_name
pp bp.full_name
pp jw.full_name
```
{: .repl #say_my_name_5 title="Say my name, 5" points="1"}

Yay! So handy. And, now that `.full_name` exists, we can use it along with `self` to build up other methods:

```ruby
class Person
  attr_accessor(:first_name)
  attr_accessor(:last_name)

  def full_name
    assembled_name = self.first_name + " " + self.last_name

    return assembled_name
  end

  def full_name_caps
    return self.full_name.upcase
  end
end

rb = Person.new
rb.first_name = "Raghu"
rb.last_name = "Betina"

pp rb.full_name_caps
```
{: .repl #say_my_name_caps title="Say my name, CAPS" points="1"}

And so on, and so forth.

## Defining "association accessors"

Let's put our instance method skills into practice by defining "association accessors". Given how important one-to-many and many-to-many associations are to our applications, giving ourselves shortcuts to traverse them easily will be an investment that pays for itself many times over.

### Walkthrough video

<div class="bg-red-100 py-1 px-5" markdown="1">
[Here is a walkthrough video for the rest of this lesson.](https://share.descript.com/view/c9aAdymOLGC) Please also read through the rest of this lesson as you watch the video (pause it frequently). Association accessors are a key concept and it will help to watch, read, and type along!
</div>

### The goal

If we have an instance of `Movie` inside a variable called `@the_movie`, instead of:

```erb
<% the_id = @the_movie.director_id >

<% matching_directors = Director.where({ :id => the_id }) %>
    
<% the_director = matching_directors.at(0) %>

<%= the_director.name %>
```

I want to be able to do:

```erb
<% the_director = @the_movie.director %>

<%= the_director.name %>
```

Or, if we're okay with chaining a couple of methods on the same line, we could even do:

```erb
<%= @the_movie.director.name %>
```

Try to embed `<%= @the_movie.director %>` in `app/views/movie_templates/show.html.erb` right now. 

Now, visit the details page of a director. You'll get a big ol' `"undefined method 'director' for #<Movie:0x00007faadebb9e88>"` error. Let's define the instance method and make it work.

### Define the method

All of our model class definitions are located in the `app/models` folder. Find `movie.rb` and define a method called `director`:

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  def director
    return "Hello!"
  end
end
```

Now, visit the details page of a movie again. Progress — the error message went away!

Please note that we could have called the method something else, if we wanted to. `.the_director`? `.el_capitan`? `.zebra`? Sure, if you want; go for it. Your team/TA/future-self might not be very happy with you, though. Usually, we should try to be as descriptive as possible.

Since it is returning an instance of the `Director` class, `.director` is a good name, I think; just like `.director_id` was a good name for the column/method that returned an `Integer`. Perhaps `.director_instance` or `.director_row` or `.director_record` might be more descriptive than `.director`? I could buy that argument (you know I love long and descriptive names), but `.director` is the convention in the Rails community when returning an ActiveRecord instance `Director`.

In most projects, the choice of what you call your methods, view templates, and anything else internal to your codebase (and not user-facing) is entirely up to you. Since this is a refactoring project, though, and the point of it is to change internal implementation while holding user-facing functionality constant, `rake grade` will be checking to see that you defined a new method called `.director`; so name it that.

### Make it return the right thing

Instead of just returning the string `"Hello!"`, let's make the method do its job: look up the row in the directors table corresponding to the receiving movie's `director_id`, and return an instance of `Director`.

Let's do it step-by-step, and look at the value of `<%= @the_movie.director %>` in the details page of a movie at each step along the way:

#### Step 1: Integer

Step 1 is to return the the receiving movie's `director_id`:

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  def director
    my_director_id = self.director_id

    return my_director_id
  end
end
```

Check out the return value of `<%= @the_movie.director %>` in the details page of a movie. Hopefully you should see an `Integer`, like `42`.

You might see nothing at all, if the `director_id` column was never populated; the value might still be `nil`. You could try `<%= @the_movie.director.inspect %>`, and then you'll get more useful information for debugging; `nil` will show up as "nil" instead of just nothing. (If a `nil` value wasn't the cause of seeing no output, then you forgot the `=` in `<%=`.)

Make sure you've got a value in the `director_id` column before moving on.

#### Step 2: ActiveRecord::Relation

Step 2 is to return an `ActiveRecord::Relation` containing records from the directors table that have the receiving movie's `director_id` in the director table's `id` column:

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  def director
    my_director_id = self.director_id

    matching_directors = Director.where({ :id => my_director_id })
    
    return matching_directors
  end
end
```

Now check out the return value of `<%= @the_movie.director %>` in the details page of a movie. Hopefully you should see an `ActiveRecord::Relation` containing 1 record.

You might see an `ActiveRecord::Relation` containing 0 records. This could be because the director of the movie has been deleted, so the value in the movie's `director_id` column no longer corresponds to any record in the directors table. Or, it could be that when the movie was created, the value that was assigned to its `director_id` attribute was never a valid ID for any director.

For whatever reason, this movie now has an invalid `director_id` and is an orphan. You could either 1) use `rails console` or visit `/rails/db` to fix the problem by updating this movie's director_id attribute to a valid director's ID, or 2) you could visit a different movie details page to test with.

Make sure you've got a non-empty `ActiveRecord::Relation` before moving on.

#### Step 3: Director

Step 3 is to return the `Director` itself!

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  def director
    my_director_id = self.director_id

    matching_directors = Director.where({ :id => my_director_id })
    
    the_director = matching_directors.at(0)

    return the_director
  end
end
```

Now check out the return value of `<%= @the_movie.director %>` in the details page of a movie. Hopefully, you should see something like `#<Director:0x00007faa7c13fb68>`, which means you've successfully returned an instance of `Director` representing a row in the table. You could add a `.inspect` for more details about the object:

```erb
<%= @the_movie.director.inspect %>
```

And then you would see all of the attributes about the record:

```
#<Director id: 1, name: "Frank Darabont", dob: "1959-01-28", bio: "Three-time Oscar nominee Frank Darabont was born i...", image: "http://ia.media-imdb.com/images/M/MV5BNjk0MTkxNzQw...", created_at: "2015-08-12 17:20:05", updated_at: "2015-08-12 17:20:05">
```

Awesome! We have successfully reduced this:

```erb
<% the_id = @the_movie.director_id >

<% matching_directors = Director.where({ :id => the_id }) %>
    
<% the_director = matching_directors.at(0) %>
```

To this:

```erb
<%= @the_movie.director %>
```

#### Step 4: Choose which attribute(s) to use

Now it becomes a joy instead of a chore for us to embed various attributes about the associated record wherever we want within the template. For example, we could do something like:

```erb
<%= @the_movie.director.name %>, born <%= @the_movie.director.dob.year %>, ...
```

To produce something like:

```
Frank Darabont, (born 1959), ...
```

## Re-using the Movie#director method

Now that we've made the investment of defining our handy method, can we use it to refactor any other parts of our code? Can we reduce complexity or increase readability?

Why, yes, I think we can; there are lots of places where we have an instance of the `Movie` class and we need to display something about its director. Go through and see if you can refactor them all.

Imagine if we had defined this "association accessor" method _before_ we started writing our controllers and views — it would have saved a lot of work and typos!

## The other side of the one-to-many

What we did by defining the `.director` method is: we made it easy to travel in one direction of a one-to-many relationship, from a movie to the director it belongs to.

Now, let's make it easy to travel in the other direction: from a director to the many movies that it can (potentially) have.

Right now, in `app/views/director_templates/show.html.erb`, we have this:

```erb
<% the_id = @the_director.id %>

<% matching_movies = Movie.where({ :director_id => the_id }) %>

<% films = matching_movies.order({ :year => :asc }) %>

<% films.each do |a_movie| %>
```

But we shouldn't have to worry about querying associations when writing our controllers and view templates; hopefully, we will have written our "association accessors" already, right after we did our domain modeling and created our tables with the appropriate foreign keys.

Let's define an instance method in the `Director` class called `.filmography` that returns an `ActiveRecord::Relation` of movie records that belong to the receiving director:

```ruby
# app/models/director.rb

class Director < ApplicationRecord
  def filmography
    my_id = self.id

    matching_movies = Movie.where({ :director_id => my_id })

    return matching_movies
  end
end
```

The convention in the Rails community would normally have been to call this method `.movies` (plural), since it is returning an `ActiveRecord::Relation` containing some number of `Movie` instances. However, I like the name `.filmography` better, since I think it's more descriptive of our problem domain.

Now, we can refactor the directors' show template to this:

```erb
<% films = @the_director.filmography.order({ :year => :asc }) %>

<% films.each do |a_movie| %>
```

It's quite concise now! If you're feeling bold, you can even get rid of the variable and take advantage of the short-hand version of `.order` (for ascending ordering):

```erb
<% @the_director.filmography.order(:year).each do |a_movie| %>
```

## Other refactorings

See how much more complexity you can remove by defining similar "association accessors". I think you'll be surprised, especially by how much you can simplify the filmography on the actor details page.

---

Here's a hint: when you're finished, you should be able to move from this:

```erb
<% a_id = @the_actor.id %>

<% matching_characters = Character.where({ :actor_id => a_id }) %>

<% matching_characters.each do |a_character| %>
  <% m_id = a_character.movie_id %>

  <% matching_movies = Movie.where({ :id => m_id }) %>

  <% the_movie = matching_movies.at(0) %>

  <tr>
    <td>
      <%= the_movie.title %>
    </td>

    <td>
      <%= the_movie.year %>
    </td>

    <td>
      <% d_id = the_movie.director_id %>
      
      <% matching_directors = Director.where({ :id => d_id }) %>
      
      <% the_director = matching_directors.at(0) %>

      <%= the_director.name %>
    </td>

    <td>
      <%= a_character.name %>
    </td>

    <td>
      <a href="/movies/<%= the_movie.id %>">
        Show details
      </a>
    </td>
  </tr>
<% end %>
```

To this:

```erb
<% @the_actor.characters.each do |a_character| %>
  <tr>
    <td>
      <%= a_character.movie.title %>
    </td>

    <td>
      <%= a_character.movie.year %>
    </td>

    <td>
      <%= a_character.movie.director.name %>
    </td>

    <td>
      <%= a_character.name %>
    </td>

    <td>
      <a href="/movies/<%= a_character.movie.id %>">
        Show details
      </a>
    </td>
  </tr>
<% end %>
```

A hundred times easier to write, read, and maintain!

## Conclusion

You shouldn't be worrying about writing that database query over and over and over and over again while you are writing your controllers and view templates.

Or, more realistically, on a multi-person team, the people who are crafting the interface probably don't even know how to write the database queries. Or it would be a huge waste of time and resources for them to do so.

We should always define instance methods in our models to encapsulate as much business logic as possible, to make it easy to re-use, easy to change, and easy to test.

Our associations are among the most important domain knowledge there is, and so are among the first thing we should encapsulate in instance methods in our models.

---
