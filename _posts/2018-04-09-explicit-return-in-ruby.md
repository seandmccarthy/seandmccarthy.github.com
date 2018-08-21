---
layout: post
title: Explicit Return in Ruby
---

#### (or, the Perl community got return right)

In a Ruby function, the result of the last expression becomes the return value for that function.

```ruby
  def demonstrate
    "I'm your return value"
  end
```

That is, unless there are any explicit return statements that execute before the end of the function.

```ruby
  def has_a_guard_clause(value)
    return if value.nil?
    evaluate(value)
  end
```

While you can use an explicit return statement as the last statement in a function, e.g.

```ruby
  def encode(value)
    outcome = transform(value)
    return outcome.success?
  end
```

the Ruby community has come down in favour of not doing this. This is evidenced by the default rules applied by the Ruby linting tool Rubocop.

[https://www.rubydoc.info/gems/rubocop/RuboCop/Cop/Style/RedundantReturn](https://www.rubydoc.info/gems/rubocop/RuboCop/Cop/Style/RedundantReturn)

Perl has the same behaviour as Ruby regarding the optional use of return.

However, the Perl community came down on the side of using an explicit return. This is the default rule applied by the Perl linting tool PerlCritic.

[https://metacpan.org/pod/distribution/Perl-Critic/lib/Perl/Critic/Policy/Subroutines/RequireFinalReturn.pm](https://metacpan.org/pod/distribution/Perl-Critic/lib/Perl/Critic/Policy/Subroutines/RequireFinalReturn.pm)

I'm going to suggest that the Perl community got it right.

A Ruby (or Perl) function will **always** return something, but it may not be intended for that something to be used.

There are three reasons I think it's valuable to have an explicit return.

1. Ruby has no static type checking to catch accidental return value errors
2. Being explicit is good information for subsequent coders
3. It can prevent accidental information leakage

Languages with static types allow us to declare the explicit return type, or the absence of a return value. A static type system would catch at compile time if you forgot to return something, or that it was the wrong type. Quite helpful!

This method returns a value of type `decimal`

```csharp
    public decimal Gradient(decimal rise, decimal run)
    {
        return rise / run;
    }
```

This method returns nothing, as indicated by the `void` return type.

```csharp
    public void Log(string message)
    {
        logger.log(timestamp + message);
    }
```

In Ruby, we have neither a static type system, nor a return type in our method signature, so we can't indicate that a function is meant to return something, or that it doesn't. Without this, we're left with options like:

- Use documentation to indicate the method's return type, or lack thereof, and/or
- Use the method name to help indicate that something (and possibly what) is being returned, and/or
- Rely on users of our code to inspect the source code for themselves and determine what is being returned.

One way we can help our colleagues, successors, and our future selves is by being explicit in the source code at least. We can do this by using the return keyword to indicate that a function does have a return value intended for use.

```ruby
  def current_time
    return Time.now
  end

  # No return keyword, so nothing notable being returned...
  def Log(string message)
    logger.log(timestamp + message);
  end
```

The PerlCritic description for the explicit return rule goes further, it suggests we should always have a return statement, even if that is to return `nil` (`undef` in Perl). So the 2nd example becomes:

```ruby
  def Log(string message)
    logger.log(timestamp + message);
    return nil
  end
```

Again, this is explicit. It indicates that we should not expect a useful value from this function, whereas the previous version might leave you wondering if `logger.log(...)` might have some value.

There is another good reason that the PerlCritic guides gives in favour of always having return. It is the potential for a function with no intended return value to accidentally leak information. Here's another example.

```ruby
  def add_to_group(email)
    group_email_list << email
  end
```

`group_email_list` is going to leak here.

Now admittedly this is quite a contrived example, but you get the idea, and I'm sure with some imagination you can envisage your library being used in unexpected ways where it emits information you didn't intend.

So, I'd advocate that we, the Ruby community, should also embrace the use of an explicit return statement. To ensure our intention is clear, and to ensure we are returning the thing we intended.

