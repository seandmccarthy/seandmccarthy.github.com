---
layout: post
title: Parsing and Comparing Version Numbers
---

A recent bug hunt led me to discover the issue was caused by some well-meaning attempts at backward compatibility, but a not-so-great choice of how to evaluate the version of Ruby in use

```ruby
unless RUBY_VERSION =~ /^1\.9/
  # supply a method not available before Ruby 1.9.x
end
```

So when Ruby 2.x became prevalent, the above code failed the check and supplied a method that was actually incompatible with Ruby versions >= 1.9.x.

I've also seen versions treated as decimal numbers and then compared, e.g.

```ruby
this_version = Float('1.9')
other_version = Float(other_version_string)

if this_version > other_version
  # ...
end
```

That seems like it should work but it has two failings.

If we think of the first number as the "major version" number, and the second as the "minor version" number, what's the next minor version after `1.9`?

Normally it would be `1.10`. So how would that go in our numerical comparision?

```ruby
if Float('1.9') > Float('1.10')
  # ...
end
```

Decimal `1.10` is actually `1.1` (dropping the insignificant 0), and is therefore _less_ than `1.9`.

The other problem is if you wanted to have [semantic versioning](https://semver.org/) with three version indicator values- "major version", "minor version", and "patch version". E.g. `1.9.3`. We can't use a decimal here.

The solution is not treat a version number as anything like a decimal. It is three distinct whole number values grouped together, needing a comparison at each level.

If you are using Ruby, you can use the [Gem::Version](https://ruby-doc.org/stdlib-2.5.0/libdoc/rubygems/rdoc/Gem/Version.html) library, it does the right thing.
