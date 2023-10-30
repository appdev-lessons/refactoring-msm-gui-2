# Refactoring Must See Movies GUI with Association Accessor Helper Methods

---

<div class="bg-blue-100 py-1 px-5" markdown="1">
[Here is a walkthrough video for this lesson.](https://share.descript.com/view/wy5mgzsL2WX) 
</div>

---

<div class="bg-red-100 py-1 px-5 bleed-full" markdown="1">
**Please note an important difference:** Since the video is from a version of the project using Ruby version 2.7, I defined methods like so:

```ruby{1:(30-47)}
belongs_to(:name_that_we_want, { :class_name => "", :foreign_key => "" })
has_many(:name_that_we_want, { :class_name => "", :foreign_key => "" })
```

However, in Ruby version 3, we _must_ use keyword arguments like so (dropping the curly braces and removing the hash rockets):

```ruby{1:(30-43)}
belongs_to(:name_that_we_want, class_name: "", foreign_key: "")
has_many(:name_that_we_want, class_name: "", foreign_key: "")
```

**Use the latter syntax when you define these methods.** The text below contains the correct syntax, with the keyword arguments.
</div>

## Objective

Our goal is to keep Must See Movies GUI (`msm-gui`) working the same way that it was after we finished building it; we're not going to add much. Therefore, we'll use the same target as before:

[https://msm-gui.matchthetarget.com/](https://msm-gui.matchthetarget.com/)

The project can be loaded here:

LTI{Load Refactoring Must See Movies GUI 2 assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(10)[Refactoring Must See Movies GUI 2 Project]

Our starting point code for this project, `refactoring-msm-gui-2`, is one possible solution for `refactoring-msm-gui-1` and `msm-validations`.

We have instance methods ("association accessors") defined in our models, which allow us to replace queries in the view templates with things like `<%= @the_movie.director.name %>`. We also have validations defined in our models to prevent bogus data from entering our database and breaking things.

Since this is another refactoring projects, most of the specs should already pass when you `rake grade`. However, there's a few failing, which you will implement in this project.

It's time to clean up our association accessor instance methods with the `belongs_to` and `has_many` helper methods!

## `belongs_to`

If we open the model files in our new project, then we will see that the association accessor methods, like `Movie#director` and `Director#filmography`, have already been defined in the models.

Let's take a look at two side-by-side: `Character#movie` and `Movie#director`:

```ruby{5}
# app/models/character.rb

# ...
class Character < ApplicationRecord
  def movie
    key = self.movie_id

    matching_set = Movie.where({ :id => key })

    the_one = matching_set.at(0)

    return the_one
  end
end
```

```ruby{8}
# app/models/movie.rb

# ...
class Movie < ApplicationRecord
  validates(:director_id, presence: true)
  validates(:title, uniqueness: true)

  def director
    key = self.director_id

    matching_set = Director.where({ :id => key })

    the_one = matching_set.at(0)

    return the_one
  end
end
```

Both of these methods are from a 1-N association and represent going from the "many" back over to the "one" that it belongs to. If I am a `Movie`, I contain a foreign key, the `director_id` column, and I want to take that value, go to the ID column of `Director`, find the matching record, and return it. The same goes for `Character#movie`.

All of our association accessor methods follow the same pattern! What if we want to define the `Character#actor` 1-N association?

```ruby{15-23}
# app/models/character.rb

# ...
class Character < ApplicationRecord
  def movie
    key = self.movie_id

    matching_set = Movie.where({ :id => key })

    the_one = matching_set.at(0)

    return the_one
  end

  def actor
    key = self.actor_id

    matching_set = Actor.where({ :id => key })

    the_one = matching_set.at(0)

    return the_one
  end
end
```

Compare the `movie` and `actor` method. They are almost identical with just a difference in a few names.

Defining association accessor methods is incredibly important for a good, maintainable code base. And, because of how formulaic this is, Rails makes it easy to automate!

There are three things that changed between `movie` and `actor`: the name of the method, the name of the foreign key column, and the name of the class that we search in.

There's a meta method called `belongs_to()`, which can be used outside of the method definitions directly in the model. This method will define our association accessor for us if we give it just three arguments:

```ruby
# app/models/character.rb

# ...
class Character < ApplicationRecord
  belongs_to(:name_that_we_want, class_name: "", foreign_key: "")

  def movie
  # ...
```

The first argument is the name of the method we want, then two key/value pairs containing the name of the class and the foreign key. So if we want a method `.movie` that we can call on `Character`, we can just write (and comment out or delete our old code):

```ruby{5}
# app/models/character.rb

# ...
class Character < ApplicationRecord
  belongs_to(:movie, class_name: "Movie", foreign_key: "movie_id")

  # def movie
  #   key = self.movie_id

  #   matching_set = Movie.where({ :id => key })

  #   the_one = matching_set.at(0)

  #   return the_one
  # end
# ...
end
```

Try this out. Comment or delete the `def movie` section of the model once you have the `belongs_to()` filled out. In your live app preview, visit an actor details page (did you run `rake sample_data` yet?). Now if we scroll down to the "Filmography" table, there is a column showing the character name in each movie for the given actor. So everything is still working even though we deleted our previous association accessor method!

### Aside: keyword arguments

Note the structure we are using for our association accessor helper method `belongs_to`:

```ruby
belongs_to(:movie, class_name: "Movie", foreign_key: "movie_id")
```

The structure `some_key: "some_value"` used in e.g. `class_name: "Movie"` is known as a **keyword argument** in Ruby, as opposed to **positional arguments**.

Positional arguments are the most common way to pass arguments to a method in Ruby. They are based on the order in which values are passed:

```ruby
def greet(name, greeting)
  return "#{greeting}, #{name}!"
end

pp greet("Ben", "Hello")
```
{: .repl #positional title="Positional arguments" points="1"}

In this case, `"Ben"` is passed as the first argument and `"Hello"` as the second. The method `greet` then uses these in the order they were passed.

On the other hand, keyword arguments are a way to pass arguments to a method in _a hash-like manner_. With keyword arguments, you can provide a name to the value you're passing to the method. Here's the same example as above, but with keyword arguments:

```ruby
def greet(name:, greeting:)
  return "#{greeting}, #{name}!"
end

pp greet(name: "Ben", greeting: "Hello")]
```
{: .repl #keyword title="Keyword arguments" points="1"}

In this case, we're explicitly saying that `"Ben"` is for the `name` argument, and `"Hello"` is for the `greeting` argument. This gives us more flexibility, as it doesn't matter which order we pass the arguments in because they're explicitly matched to their respective parameters in the method definition.

```ruby
pp greet(greeting: "Hello", name: "Ben")   # Still outputs: "Hello, Ben!"
```

Keyword arguments provide more flexibility and readability, especially when working with methods that can take many parameters like `belongs_to`.

### Why we prefer `belongs_to`

The `def movie` that we wrote out line-by-line and the new `belongs_to` version are producing the same thing. But `belongs_to` is somewhat shorter, and it reads better. It is easier to understand at a glance.

Moreover, we can look at an example in our Rails console for why this technique is so good. Be sure to run `rake sample_data`, then open a `rails console`. 

What if we asked for all of the characters from movies that are newer than 1994? Until today we couldn't do that! 

We can use a new method called `.joins`, which you give the name of an association that you declared with `belongs_to` (it *must* be declared this way for `.joins` to work). Then you can use `.where` and the association accessor with another hash that searches over a specific range specified by another column:

```ruby
pry(main)> Character.joins(:movie).where("movies.year > 1994")
```

Note here that when using `joins`, `where` takes its argument as a string rather than a hash. That will return an `ActiveRecord::Relation` with all of the movies of interest.

It is very common to query one table with the columns of another. And if we look at the SQL issued by the relatively simple Ruby command above, then we see it is getting pretty tricky to write; `ActiveRecord`  does it for us.

Yet another reason that `belongs_to` is great, is that we can write: 

```ruby
belongs_to(:movie, :class_name => "Movie", :foreign_key => "movie_id")
``` 

as simply

```ruby
belongs_to(:movie)
``` 

We can delete most of the code because the method `belongs_to` will follow a well defined pattern of naming! As long as the function, table, and foreign key columns share the same name (with only capitalization difference), then we can write a very short, explicit, and clear association accessor method.

If you prefer to write out the entire thing, you are welcome to, but this is just another shortcut in your toolbox.

## `has_many`

As opposed to `belongs_to(:movie)` in `Character` (`Character#movie`), we also want to have a `Movie#characters` method, which is the other side of the relationship, the many to the one. We could define this ourselves like so:

```ruby{8-12}
# app/models/movie.rb

# ...
class Movie < ApplicationRecord
  validates(:director_id, presence: true)
  validates(:title, uniqueness: true)
  
  def characters
    key = self.id
    the_many = Character.where( {:movie_id => key })
    return the_many
  end

  def director
    key = self.director_id

    matching_set = Director.where({ :id => key })

    the_one = matching_set.at(0)

    return the_one
  end
end
```

This is a different method compared to the one to the many. Of course, the `director` method could already be replaced with just `belongs_to(:director)`! But how can we replace `def characters`? 

`belongs_to(:characters)` won't work, that will write the wrong method in this case, because our new method does not follow the expected pattern for a one to many. 

We need to use the method `has_many`, which has a similar form:

```ruby{9}
# app/models/movie.rb

# ...
class Movie < ApplicationRecord
  validates(:director_id, presence: true)
  validates(:title, uniqueness: true)
  
  belongs_to(:director)
  has_many(:characters, :class_name => "Character", :foreign_key => "movie_id")

  # def characters
  #   key = self.id
  #   the_many = Character.where( {:movie_id => key })
  #   return the_many
  # end
end
```

All we did was provide the method with the three pieces of information that we expect to vary: the name of the method, name of the class, and name of the foreign key. And then `has_many` writes an association accessor method for us of the form that we put together in `def characters`.

Similar to `belongs_to`, if the three arguments match, we can write:

```ruby
has_many(:characters, :class_name => "Character", :foreign_key => "movie_id")
``` 

as simply

```ruby
has_many(:characters)
``` 

This shortcut only works if the names match! For instance, in our `Director` model (`app/models/director.rb`), we have an association accessor method called `filmography`. 

We could have called this `movies`, which would have allowed us to write `has_many(:movies)`, but if we wrote `has_many(:filmography)`, then Rails would look for a class called `Filmography`, and a foreign key column called `filmography_id`, neither of which exist!

So in the case where we selected our own, non-conventional names, then we need to be explicit and write: 

```ruby
has_many(:filmography, :class_name => "Movie")
``` 

We can still omit the foreign key, because now the method will know to call the foreign key `movie_id` based on the `:class_name`.

## Finishing up with the association accessor wizard

For the rest of this project, follow along with the video to learn about the helpful association accessor wizard that we built for you:

[association-accessors.firstdraft.com](https://association-accessors.firstdraft.com/)

Your goal is to define all six 1-Ns using `belongs_to` and `has_many`: 

- `Actor#characters` 
- `Character#actor`
- `Movie#characters`
- `Character#movie`
- `Director#filmography`
- `Movie#director`

When you are done, the last few `rake grade` specs should pass.

## The `:source` and `:through` keywords

It may be helpful to use some additional keyword arguments on the `has_many` method.

There is more detail in the [Rails guide for `has_many :through`](https://guides.rubyonrails.org/association_basics.html#the-has-many-through-association) and for [the `:source` keyword](https://guides.rubyonrails.org/association_basics.html#options-for-has-many-source), but let's discuss them with a practical example relevant to our database tables here.

You can use the `has_many :through` and `belongs_to` associations to set up a many-to-many relationship between models through a join model. The `:source` option can be used to specify the source association for a `has_many :through` association.

For example, let's consider a movie database where a `Movie` has many `Actor`s through `Character`s, and an `Actor` has many `Movie`s through `Character`s. Here's how you can set that up:

In the `Movie` model:

```ruby{3}
class Movie < ApplicationRecord
  has_many(:characters)
  has_many(:actors, :through => :characters, :source => :actor)
end
```

And in the `Actor` model:

```ruby{3}
class Actor < ApplicationRecord
  has_many(:characters)
  has_many(:movies, :through => :characters, :source => :movie)
end
```

In the `Character` model:

```ruby
class Character < ApplicationRecord
  belongs_to(:movie)
  belongs_to(:actor)
end
```

With this setup, you can access all actors in a movie with `the_movie.actors`, and all movies an actor has been in with `the_actor.movies`.

The `:through` option specifies the association name for the join model (`:characters`), and the `:source` option specifies the name of the association in the join model that `has_many :through` is going through to get to the `Actor` or `Movie`. 

The `:source` option is necessary here because the source of the `has_many` could not be automatically inferred from the association name. If the `:source` option is not provided, Rails would look for an association named `actors` in the `Character` model when you invoke `the_movie.actors`, and similarly for `the_actor.movies`. With the `:source` option, you're telling Rails to use the `:actor` and `:movie` associations in the `Character` model instead.

---
