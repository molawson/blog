---
title: The Power of Ruby's Case Statement
---

Case (or switch) statements don't seem very exciting. When you're using a language like Ruby that lets you do things like metaprogramming, why bother talking about case statements? I felt the same way until recently. It turns out that the way Ruby implements case statements makes them quite a bit more powerful than they first appear.

## The Basics

Let's take a look at a simple method that contains a single `case` statement.

```ruby
def type_for(name)
  case name
  when 'Clive'
    :author
  when 'Peter'
    :character
  when 'Susan'
    :character
  when 'Edmund'
    :character
  when 'Lucy'
    :character
  else
    :unknown
  end
end

type_for 'Susan' # => :character
```

Now, this code is troubling for a number of reasons, but for now, it gets the job done and illustrates a point. 

### Case Equality

You probably know that `case` statements in Ruby just compare the argument passed to the `case` keyword with the argument in each `when` branch.  If it finds one that matches, the code follows that branch. 

The key to grasping the *power* of Ruby's `case` statements is understanding that it compares each branch using the `===` ( "case equality" or "threequals") operator. So, in some senses, the previous code could be translated to something like this:

```ruby
def type_for(name)
  if 'Clive' === name
    :author
  elsif 'Peter' === name
    :character
  elsif 'Susan' === name
    :character
  elsif 'Edmund' === name
    :character
  elsif 'Lucy' === name
    :character
  else
    :unknown
  end
end
```

You can see from the redundancy of that code why we'd want to use `case...when` instead of `if...elsif`, but that's beside the point. 

It is important to note the *order* of the objects being compared in each branch above because we'll be able to use it to our advantage in just a bit, but we're not there yet.

### Multiple Arguments

A great feature of Ruby's `case` statement is that you can pass multiple arguments to each `when` keyword. That lets us clean up our ugly code quite a bit.

```ruby
def type_for(name)
  case name
  when 'Clive'
    :author
  when 'Peter', 'Susan', 'Edmund', 'Lucy'
    :character
  else
    :unknown
  end
end

type_for 'Susan' # => :character
```

That's much better!  If this were our real code, we might stop there, but for the sake of our experiments, we'll keep going. First, let's take a second to understand how this multiple arguments thing works. 

When you pass multiple arguments to `when`, Ruby evaluates each one of them individually, in order.  So, you could translate the previous code to something like this:

```ruby
def type_for(name)
  if 'Clive'
    :author
  elsif 'Peter' === name || 'Susan' === name || 'Edmund' === name || 'Lucy' === name
    :character
  else
    :unknown
  end
end
```

With the basic mechanics out of the way, we can dig into the fun stuff.

## Defining Case Equality

As we said before, the *power* of Ruby's `case` statement is hiding in the `===` operator. I say "operator", but really, it's just a method. 

The `===` method is defined on the `Object` class, which, according to the Ruby docs is "the default root of all Ruby objects." That is, most any object in Ruby will inherit this behavior from `Object`.

`Object` essentially defines `===` as an alias for `==`, which is what we use for equality comparison in Ruby. So, for most types of objects (e.g. strings, arrays, etc.), using them in a case statement means that we're evaluating them based on some sort of "normal" equality. But that doesn't have to be the case (see what I did there?).

### Regular Expressions

Since most classes ultimately inherit from `Object`, they get a default for the `===` method, but, thanks to Ruby's method call chain, they're free to override that behavior.  The `Regexp` class takes advantage of this.  Using the `==` equality method with a regular expression will do what you probably expect: test whether or not the argument is equal to itself.

```ruby
/foo/ == /foo/ # => true
/foo/ == /bar/ # => false
/foo/ == 'foo' # => false
```

That makes sense, but `Regexp` then defines the `===` method a little differently. Since the general purpose of regular expressions is pattern matching, `Regexp#===` tests whether or not the argument *matches* the regular expression.

```ruby
/foo/ === 'foo' # => true
/foo/ === 'bar' # => false
```

This behavior gives us another way to use `case` statements.

```ruby
def half_for(name)
  case name
  when /^[a-m]/i
    :first
  when /^[n-z]/i
    :second
  else
    :unknown
  end
end

half_for 'Peter'  # => :second
half_for 'Edmund' # => :first
```

### Procs and Lambdas

Regular expressions are great and all, but we're really just starting to scratch the surface. My favorite objects to use in `case` statements are `Proc`s and lambdas (If you're not familiar with Procs and lambdas in Ruby, Robert Sosinski has a great [blog post](http://www.robertsosinski.com/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/) on the topic).

We can refactor our original code using a lambda, like so.

```ruby
def type_for(name)
  characters = ['Peter', 'Susan', 'Edmund', 'Lucy']

  case name
  when 'Clive'
    :author
  when ->(n) { characters.include? n }
    :character
  else
    :unknown
  end
end

type_for 'Susan' # => :character
```

We can translate that to its `if...elsif...else` form.

```ruby
def type_for(name)
  characters = ['Peter', 'Susan', 'Edmund', 'Lucy']
  is_character = ->(n) { characters.include? n }

  if 'Clive' === name
    :author
  elsif is_character === name
    :character
  else
    :unknown
  end
end
```

To get us the behavior we've just seen, the `Proc` class (of which lambdas are a special type) essentially aliases the `===` method to its `call` method, passing along the argument. So, we end up with something like this:

```ruby
def type_for(name)
  characters = ['Peter', 'Susan', 'Edmund', 'Lucy']
  is_character = ->(n) { characters.include? n }

  if 'Clive' === name
    :author
  elsif is_character.call(name)
    :character
  else
    :unknown
  end
end
```

Now we can use Procs and lambdas to construct more complex methods for evaluating objects in `case` statements.

### Going Further

While the previous examples are certainly simplistic and contrived, the behavior of regular expressions, Procs, and lambdas opens up all kinds of possibilities. Things get even more interesting when you define the `===` method in your own classes for custom `case` statement behavior.

This tool can be really handy in the right situation. There are a number of situations where the more "advanced" use of case statements can make for very expressive and readable code.

The way Ruby takes advantage of inheritance and method calls in `case` statements is a great example of how flexible and extensible a system can be that relies on a common *message* interface rather than a certain *type* of object.

#### (Re)sources
* [Ruby Tapas - Episode #37: Proc and Threequal](http://www.rubytapas.com/episodes/37-Proc-and-Threequal) - Screencast from Avdi Grimm that inspired this blog post (subscription required).
* [Confident Ruby](http://www.confidentruby.com/) - Book by Avid Grimm that explores a number of related topics, especially common interfaces, in more detail.
