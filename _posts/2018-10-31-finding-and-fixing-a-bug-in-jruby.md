---
layout: post
title: Finding and Fixing a Bug in JRuby
---

In a Rails application I work on we have a class in that provides a kind of null/nil value. It can be used in both the context of arithmetic operations, and in a string context. It is pretty similar to the example below:

<script src="https://gist.github.com/seandmccarthy/a0e771e6fa4f7e3c3a6efe2a67c4099f.js"></script>

With this we can do things like the following without needing to check for nil values:

```ruby
irb(main):010:0> values = [BigDecimal('12.3'), BigDecimal('45.6'), NullValue.new]
irb(main):011:0> values.reduce(:+).to_f
=> 58.9
```

The way the `coerce` method is implemented, it should work with any operation a Numeric supports.

However, it turned out that we'd never used this `NullValue` in subtraction previously. When a developer in the team introduced code using this operation, it worked just fine using MRI/CRuby, however when it was deployed to our continuous deployment server that used JRuby, it suddenly failed.

I decided to track down what was going on.

The issue was easy to reproduce:

```
irb(main):001:0> RUBY_VERSION
=> "2.5.0"
irb(main):002:0> JRUBY_VERSION
=> "9.2.0.0"
irb(main):003:0> load './null_value.rb'
=> true
irb(main):004:0> BigDecimal(1) - NullValue.new
Traceback (most recent call last):
        7: from /Users/seanm/.rbenv/versions/jruby-9.2.0.0/bin/jirb:13:in `<main>'
        6: from org/jruby/RubyKernel.java:1180:in `catch'
        5: from org/jruby/RubyKernel.java:1180:in `catch'
        4: from org/jruby/RubyKernel.java:1418:in `loop'
        3: from org/jruby/RubyKernel.java:1037:in `eval'
        2: from (irb):4:in `<eval>'
        1: from org/jruby/ext/bigdecimal/RubyBigDecimal.java:1134:in `-'
TypeError (NullValue can't be coerced into BigDecimal)
```

I tried a few different versions of JRuby to check if it was a recent regression, however those versions (9.1.17.0 and 1.7.27) had the same issue.

The stack trace gives a place to start. Examining line [1134](https://github.com/jruby/jruby/blob/4e8bb2666a7257f0f5986800f96bb88efdd6acbd/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java#L1134) of `org/jruby/ext/bigdecimal/RubyBigDecimal.java`

```java
@JRubyMethod(name = "-", required = 1)
public IRubyObject op_minus(ThreadContext context, IRubyObject b) {
    return subInternal(context, getVpValueWithPrec(context, b, true), b, 0);
}
```

left me scratching my head a bit. Without a bit of context nothing stands out. I looked at the implementation of [subInternal()](https://github.com/jruby/jruby/blob/4e8bb2666a7257f0f5986800f96bb88efdd6acbd/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java#L1152-L1155)

```java
private IRubyObject subInternal(ThreadContext context, RubyBigDecimal val, IRubyObject b, int prec) {
    if (val == null) return callCoerced(context, sites(context).op_minus, b);
    RubyBigDecimal res = subSpecialCases(context, val);
    return res != null ? res : new RubyBigDecimal(context.runtime, value.subtract(val.value)).setResult(prec);
}
```

That line containing `callCoerced()` looked like the path to investigate. It is only called if `val` is null. From the calling location in `op_minus`, that `val` is the result of `getVpValueWithPrec(context, b, true)`.

To try to get some context of how this is used, I looked for similar calls to `getVpValueWithPrec`. There's a method [op_plus](https://github.com/jruby/jruby/blob/f7ef0fb0ef6a4fcc178facb9f25595ed8be4cad3/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java#L1038) which appears to be similar to `op_minus`.

```java
@JRubyMethod(name = "+")
public IRubyObject op_plus(ThreadContext context, IRubyObject b) {
    return addInternal(context, getVpValueWithPrec(context, b, false), b, vpPrecLimit(context.runtime));
}
```

The difference in this case is that the third parameter to `getVpValueWithPrec` is **`false`** instead of **`true`**. Time to see what that does.

```java
private RubyBigDecimal getVpValueWithPrec(ThreadContext context, IRubyObject value, boolean must) {
    if (value instanceof RubyFloat) {
        // Unimportant details omitted
    }
    if (value instanceof RubyRational) {
        // Unimportant details omitted
    }

    return getVpValue(context, value, must);
}
```

At this point we see the third parameter is named `must`.  I know that our `value` isn't a RubyFloat or RubyRational (it's a NullValue), so this method falls through to call `getVpValue`, passing through the `must` value.

```java
private static RubyBigDecimal getVpValue(ThreadContext context, IRubyObject value, boolean must) {
    switch (((RubyBasicObject) value).getNativeClassIndex()) {
        case BIGDECIMAL:
            ...
        case FIXNUM:
            ...
        // Unimportant details omitted
    }
    return cannotBeCoerced(context, value, must);
}
```

Nearly there. Again it falls through the other type checks and ends up at the call `cannotBeCoerced` with the `must` value passed in.

```java
private static RubyBigDecimal cannotBeCoerced(ThreadContext context, IRubyObject value, boolean must) {
    if (must) {
        throw context.runtime.newTypeError(
            errMessageType(context, value) + " can't be coerced into BigDecimal"
        );
    }
    return null;
}
```

This method contains the text of the error message received from the failed subtraction that started this investigation. So if `must` is `true`, then the error is raised. Recall that the `op_plus` has the `must` value as `false`. 

Falling all the way back up the call stack, we see that if `must` is instead `false`, then `getVpValue` and subsequently `getVpValueWithPrec` will return `null`. Thus the `val` passed to `subInternal` will be `null`. 

```java
private IRubyObject subInternal(ThreadContext context, RubyBigDecimal val, IRubyObject b, int prec) {
    if (val == null) return callCoerced(context, sites(context).op_minus, b);
    // Unimportant details omitted
}
```

With `val` being null, the `callCoerced` method would actually be called in the first line of `subInternal`, instead of an exception being raised.

The fix is to change that `must` value:

```bash
diff --git a/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java b/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java
index 12b7788b2d..5c3373b7ee 100644
--- a/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java
+++ b/core/src/main/java/org/jruby/ext/bigdecimal/RubyBigDecimal.java
@@ -1131,7 +1131,7 @@ public class RubyBigDecimal extends RubyNumeric {

     @JRubyMethod(name = "-", required = 1)
     public IRubyObject op_minus(ThreadContext context, IRubyObject b) {
-        return subInternal(context, getVpValueWithPrec(context, b, true), b, 0);
+        return subInternal(context, getVpValueWithPrec(context, b, false), b, 0);
     }
```

That addresses the exception that was being raised.

However, while I was looking around trying to understand how [callCoerced](https://github.com/jruby/jruby/blob/4e8bb2666a7257f0f5986800f96bb88efdd6acbd/core/src/main/java/org/jruby/RubyNumeric.java#L460) worked, I noticed that there was a difference in the way it is called by methods similar to `subInternal`.

`subInternal` calls it as follows:

```java
return callCoerced(context, sites(context).op_minus, b);
```

whereas `addInternal` calls it with four parameters:

```java
return callCoerced(context, sites(context).op_plus, b, true);
```

Most of the other invokations of `callCoerced` used the 4 parameter version, with that last parameter set to `true`.

The implementation of the 3 parameter form of `callCoerced` method is:

```java
public final IRubyObject callCoerced(ThreadContext context, CallSite site, IRubyObject other) {
    return callCoerced(context, site, other, false);
}
```

Of interest is how it calls the 4 parameter version of the method with a value of **`false`**. So I decided to check what that does.

The 4 parameter version of [callCoerced](https://github.com/jruby/jruby/blob/4e8bb2666a7257f0f5986800f96bb88efdd6acbd/core/src/main/java/org/jruby/RubyNumeric.java#L464-L469) looks like:

```java
protected final IRubyObject callCoerced(ThreadContext context, CallSite site, IRubyObject other, boolean err) {
    IRubyObject[] args = getCoerced(context, other, err);
    if (args == null) return context.nil;
    IRubyObject car = args[0];
    return site.call(context, car, car, args[1]);
}
```

That 4th parameter is `err` and that gets passed to [getCoerced](https://github.com/jruby/jruby/blob/4e8bb2666a7257f0f5986800f96bb88efdd6acbd/core/src/main/java/org/jruby/RubyNumeric.java#L435-L452). The interesting part is below:

```java
    try {
        result = sites(context).coerce.call(context, other, other, this);
    }
    catch (RaiseException e) { // e.g. NoMethodError: undefined method `coerce'

        if (error) {
            throw runtime.newTypeError(str(runtime, types(runtime, other.getMetaClass()), " can't be coerced into ", types(runtime, getMetaClass())));
        }
        return null;
    }
```

If there is no `coerce` method provided, an exception is raised. This exception is swallowed unless the `error` parameter (which is the `err` parameter from `callCoerced`) is `true`.

In the case of the `NullValue` example from the beginning of this post, if you remove the `coerce` method (and the `method_missing` one) then the outcome would look like this:

```
irb(main):001:0> load './null_value.rb'
=> true
irb(main):002:0> BigDecimal(1) + NullValue.new
Traceback (most recent call last):
        7: from bin/jirb:12:in `<main>'
        6: from org/jruby/RubyKernel.java:1181:in `catch'
        5: from org/jruby/RubyKernel.java:1181:in `catch'
        4: from org/jruby/RubyKernel.java:1415:in `loop'
        3: from org/jruby/RubyKernel.java:1043:in `eval'
        2: from (irb):2:in `evaluate'
        1: from org/jruby/ext/bigdecimal/RubyBigDecimal.java:1039:in `+'
TypeError (NullValue can't be coerced into BigDecimal)
irb(main):003:0> BigDecimal(1) - NullValue.new
=> nil
```

Addition fails with an exception as expected, subtraction fails quietly.

To make it consistent, the invokation of `callCoerced` in `subInternal` needs to change to the 4 parameter version with an `err` value of `true`, just like `addInternal`.

The full PR for these fixes is https://github.com/jruby/jruby/pull/5391

As a final thought, it's often interesting how much effort can go into what ultimately is a tiny code change.
