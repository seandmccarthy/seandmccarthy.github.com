---
layout: post
title: Explicit Return in Ruby
---

#### (or Perl got return right)

In a Ruby function, the result of the last expression becomes the return value for that function.

    def demonstrate
      "I'm your return value"
    end

That is, unless there are any explicit return statements that execute before the end of the function.

    def has_a_guard_clause(value)
      return if value.nil?
      evaluate(value)
    end

While you can use an explicit return statement as the last statement in a function, 

    def encode(value)
      outcome = transform(value)
      return outcome.success?
    end

the Ruby community has come down in favour of not doing this. This is evidenced by the default rules applied by the Ruby linting tool Rubocop.

https://www.rubydoc.info/gems/rubocop/RuboCop/Cop/Style/RedundantReturn

Perl has the same behaviour as Ruby regarding the optional use of return.

However, the Perl community came down on the side of using the explicit return. This is the default rule applied by the Perl linting tool PerlCritic.

https://metacpan.org/pod/distribution/Perl-Critic/lib/Perl/Critic/Policy/Subroutines/RequireFinalReturn.pm

I'm here to suggest that the Perl community got it right.

In Ruby, we don't have a return type in our method signature, so we can't indicate that a function is meant to return something, or that it doesn't. Unlike other languages

  Example from Java or C#

A Ruby (or Perl) function will always return something, but it may not be intended for that something to be used.

  Example in IRB showing a procedure

Perhaps the function name can help indicate that nothing is returned 

  Examples

However this may not be enough.

So we can help our colleagues, successors, and ourselves by making things explicit.

We can do this by using the explicit return keyword to indicate that this function does have a return value intended for use

  Example

Or we can omit it as an indicator that nothing of value is emitted by this function.

  Example

By following this guideline we can have a kind of contract about what can be expected from our function. This works if people can look at your source code.

Otherwise you can use documentation to indicate return value, or lack there of.

The PerlCritic description for the explicit return rule suggests goes further, it suggests we should always have a return statement, even if that just returns nil.

The reasoning they give is somethings we saw a little earlier. It is the potential for a function with no intended return value to accidentally leak information. Here's another example.

  Example

Now admittedly this is quite a contrived example, but you get the idea, and I'm sure with some imagination you can envisage your library being used in unexpected ways where it emits information you didn't intend.

So, I'd advocate that we, the Ruby community, should also embrace the use of an explicit return statement. To ensure our intention is clear, and that we are returning the thing we intended.

http://search.cpan.org/~thaljef/Perl-Critic-1.123/lib/Perl/Critic/Policy/Subroutines/RequireFinalReturn.pm
