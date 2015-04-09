---
title: Elixir Pipes in Ruby
---

I've been taking a little time here and there to learn some [Elixir](http://elixir-lang.org/) over the past couple of months. I've really enjoyed the language itself, and it's been a fantastic introduction to functional programming.

I recently came across a problem in a Ruby app I was working on that provided a little "ah ha!" moment. It involved taking some of what I've learned in Elixir–more specifically, it's [pipe operator `|>`](http://elixir-lang.org/docs/stable/elixir/Kernel.html#|>/2)—and applying it to that Ruby code. 

## The Problem

The problem I faced required me to come up with a way to standardize some of the data in one of our database. The gist of the problem was that I needed to take a set of strings that came from a few years of user input and normalize them to a form that we could store elsewhere in the database while minimizing duplication.

```ruby
class KeywordNormalizer
  def call(keywords)
    keywords
      .map { |kw| kw.gsub(/(^[^[:alpha:]]+|[^[:alpha:]]+$)/, '') }
      .map { |kw| LegacySpanishCorrector.new.correct(kw) }
      .map { |kw| kw.gsub(/[^[:alpha:]\d'_\-\/]/, '') }
      .reject { |kw| STOP_WORDS.include?(kw) }
  end
end
```

This is the first pass of code that "worked" (minus a few of the more mundane transformations) and had our tests passing. Functionally, this was done, but it left a little to be desired.

We have a service object called `KeywordNormalizer`. It's single public method is `#call`. It takes an array of strings as its only argument, and it performs a series of transformations on that array.

While this method mostly makes sense, we're left with a couple of steps in the process that aren't exactly clear. Let's focus on the first and third steps. Both of these use regular expressions. While I love the power of regular expressions, they tend to be extremely cryptic, bordering on magical. For the sake of whoever has to touch this code next, these steps could really benefit from a little clarity, so we'll extract these steps into separate, private methods and give them some decent names.

```ruby
class KeywordNormalizer
  # ...

  private

  def strip_outer_punctuation(array)
    array.map { |el| el.gsub(/(^[^[:alpha:]]+|[^[:alpha:]]+$)/, '') }
  end

  def fix_spanish(array)
    array.map { |el| LegacySpanishCorrector.new.correct(el) }
  end

  def clean_inner_punctuation(array)
    array.map { |el| el.gsub(/[^[:alpha:]\d'_\-\/]/, '') }
  end

  def remove_stop_words(array)
    array.reject { |el| STOP_WORDS.include?(el) }
  end
end
```

Now, our regular expression-laden methods have been given names that give us a little clue about their purpose, "strip outer punctuation" and "clean inner punctuation". Since the rules around how we handle internal and external punctuation are a little different, it makes sense that these methods are separate, and their names help to point us in that direction.

This refactoring definitely adds clarity around the individual steps, but even here, you can see that something is starting to smell. All four of these methods take an array, perform a transformation of some sort, and return the result. 

There's also an added awkwardness to our original "call" method. It looks like this now. 

```ruby
class KeywordNormalizer
  def call(keywords)
    keywords = strip_outer_punctuation(keywords)
    keywords = fix_spanish(keywords)
    keywords = clean_inner_punctuation(keywords)
    remove_stop_words(keywords)
  end

  # ...
end
```

While each step is a little more clear, the process and flow is more disjointed. All of the local variable assignments add noise and a certain amount of misdirection from the purpose of this method.

I've seen this kind of "flow", if you can call it that, in a number of codebases (and `git blame` reminds me that I've written more of this kind of code than I'd care to admit). In fact, I've seen this pattern enough that I've started calling it the "tower of assignment" pattern.

We could just call this "good enough" and leave it as it is. But I think we can do better.

If we go back to our new private methods, this smelliness would normally lead us in the direction of extracting a new object. If we go down that road, we can create a new `Collection` class that we'll just embed in the `KeywordNormalizer` for now.

```ruby
class KeywordNormalizer
  class Collection
    def initialize(array)
      @array = array
    end

    def strip_outer_punctuation
      @array.map { |el| el.gsub(/(^[^[:alpha:]]+|[^[:alpha:]]+$)/, '') }
    end

    def fix_spanish
      @array.map { |el| LegacySpanishCorrector.new.correct(el) }
    end

    def clean_inner_punctuation
      @array.map { |el| el.gsub(/[^[:alpha:]\d'_\-\/]/, '') }
    end

    def remove_stop_words
      @array.reject { |el| STOP_WORDS.include?(el) }
    end
  end
end
```

`Collection` takes an array as the only argument to `#initialize`, and each of our previously private methods are now public and do their transforms on the array that the class was initialized with.

That solves the surface-level problem that was staring us in the face, but if we go all the way back up to our `#call` method, things aren't any better. 

```ruby
class KeywordNormalizer
  def call(keywords)
    keywords = Collection.new(keywords).strip_outer_punctuation
    keywords = Collection.new(keywords).fix_spanish
    keywords = Collection.new(keywords).clean_inner_punctuation
    Collection.new(keywords).remove_stop_words
  end

  # ...
end
```

I'd suggest that, from the `#call` method's perspective, this last refactoring step is a step backwards. We still have the disjointed flow and indirection from before, and now we have the added cognitive load of another class that we need to understand to some degree.

## Elixir's Solution

When I took a break for minute, I was reminded of what I'd been learning in Elixir. One of the most recent exercises I'd gone through involved writing some code that would count the number of times each word occurred in a given string. You're given a string of some sort, and you've got to do similar things to what we're doing here–remove punctuation, downcase everything, etc.–to get the words in a form state where you can reliably compare them against each other for counting. 

During that exercise, I was introduced to the pipe operator. To demonstrate its use, here's a simplified version of our `KeywordNormalizer` class written in Elixir.

```elixir
defmodule KeywordNormalizer do
  def call(keywords) do
    strip_outer_punc(keywords) |> fix_spanish |> clean_inner_punc |> remove_stop_words
  end

  defp strip_outer_punc(list) do
    # do work
  end

  defp fix_spanish(list) do
    # do work
  end

  defp clean_inner_punc(list) do
    # do work
  end

  defp remove_stop_words(list) do
    # do work
  end
end
```

Even if you haven't seen Elixir before, this should still feel pretty familiar. It's very Ruby-esque in some ways.  We've got a `KeywordNormalizer` module with a `call/1` function, which we'll come back to. Then we've got our four private functions (`defp` makes a method private, as opposed to the normal `def`, which creates a public function). They're exactly analagous to their Ruby counterparts when we first extracted those methods. Each one takes a `List` (similar to Ruby's `Array`) as it's only argument. We'll assume that each one does the work it needs to and returns a new, transformed `List`.

With that out of the way, let's focus back on the `call/1` function that we're really interested in.  And really, we're interested in these pipe operators. They way they work is that they output of the function to the left of the operator is passed to the function on the right, as its first argument. It's a sort of output-to-input flow, which is especially clean here, where each function just takes a single argument. And that's why we don't have to explicitly pass any arguments to the last three functions; the pipe operator handles that for us.

The result is a `call/1` function that's very clear and concise. It's a fantastic picture of what we're doing.

## Translating |> into Ruby

After I reminded myself about Elixir's pipe operator, I started thinking about how we might be able to implement something like it in Ruby. To begin with, I went back to the basics of what sets these languages apart. As a functional language, Elixir is focused around functions that take input and return output. It, as with any functional language, is steeped in this input/output idea that's at the heart of the pipe operator.

Ruby, on the other hand, as an object oriented language is focused around sending messages to objects and having those objects do work and/or return values. 

_The pipe operator works based on the fact that the return of one function can be passed as an argument to the next function. So, to get this sort of thing working in an object oriented world, we need to have the return value of one method respond to the next message we want to send. Instead of the output becoming the input we need, the return value needs to become the object we need._

If we go back to our `Collection` class, you can see that we're halfway there already. 

```ruby
class KeywordNormalizer
  class Collection
    def initialize(array)
      @array = array
    end

    def strip_outer_punctuation
      @array.map { |el| el.gsub(/(^[^[:alpha:]]+|[^[:alpha:]]+$)/, '') }
    end

    def fix_spanish
      @array.map { |el| LegacySpanishCorrector.new.correct(el) }
    end

    def clean_inner_punctuation
      @array.map { |el| el.gsub(/[^[:alpha:]\d'_\-\/]/, '') }
    end

    def remove_stop_words
      @array.reject { |el| STOP_WORDS.include?(el) }
    end
  end
end
```

We have an object that responds to every message we want to send in this workflow. All we need to do is to make sure that, every time one of these methods is called, we return a `Collection` object rather than a plain `Array`. And since a `Collection` object is essentially a wrapper around an `Array`, this happens to be straightforward.

We can start with this `#remove_stop_words` method. Here's the current implementation.

```ruby
class KeywordNormalizer
  class Collection
    # ...

    def remove_stop_words
      @array.reject { |el| STOP_WORDS.include?(el) }
    end

    # ...
  end
end
```

After our transformation, it looks like this.

```ruby
class KeywordNormalizer
  class Collection
    # ...

    def remove_stop_words
      new_array = @array.reject { |el| STOP_WORDS.include?(el) }
      self.class.new new_array
    end

    # ...
  end
end
```

We just assign it's previous return value to a `new_array` variable and create a new instance of the `Collection` class, passing in the `new_array`.

Since we're going to do this a few times, we'll pull that logic out into a private method.

```ruby
class KeywordNormalizer
  class Collection
    # ...

    def remove_stop_words
      new @array.reject { |el| STOP_WORDS.include?(el) }
    end

    # ...

    private

    def new(new_array)
      self.class.new new_array
    end
  end
end
```

Lastly, we'll add a `#to_a` method so other objects that use instances of this class can still get at the array itself.

```ruby
class KeywordNormalizer
  class Collection
    # ...

    def remove_stop_words
      new @array.reject { |el| STOP_WORDS.include?(el) }
    end

    # ...

    def to_a
      @array
    end

    private

    def new(new_array)
      self.class.new new_array
    end
  end
end
```

Once we've updated the rest of the methods in this class, our original `#call` method looks something like this.

```ruby
class KeywordNormalizer
  def call(keywords)
    Collection.new(keywords)
      .strip_outer_punctuation
      .fix_spanish
      .clean_inner_punctuation
      .remove_stop_words
      .to_a
  end

  # ...
end
```

We create a `Collection` with the keywords that are passed in, we call a series of transformation methods on that set, and then we get an array out the other end.

_While we still have the cognitive overhead of a separate class, I'd argue that that overhead isn't much more than understanding Elixir's pipe operator. The result is much more readable than our initial attempt, while staying true to the procedural nature of the workflow itself._

## Conclusion

As I finished writing this code, it occurred to me that I use objects in a similar way all the time. `ActiveRecord::Relation` has been using a similar pattern of chainable methods for a while now. But there was something about the experience I had recently in Elixir that brought me back to this construct in Ruby. Also, it's one thing to use a code that implements a certain pattern, and it's a very different thing to come across an instance where you need to implement that pattern yourself.

While this is not a strict implementation of pipes in Ruby, I see this as the OO analogue of functional pipes. As I mentioned at the beginning, this pattern seems to fit much better within the Ruby ecosystem; though, that's almost certainly a matter of opinion.

I think the thing that's really stuck with me through this is the power of Ruby (again). We all know that it's a powerful language and one of it's greatest features is its flexibility. While we can (and probably all have) used it's flexibility to a fault, it's still a fantastic feature. I love that I can experiment with other languages–more specifically other programming paradigms–and they can inform and improve even my Ruby code. I didn't expect that. That won't always be the case, but it's great when it is.
