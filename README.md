# Salvage

![Build Status](https://travis-ci.org/xogeny/salvage.svg?branch=master)

This library to handle the use case where you have React components (or any system where you want to use
the `===` operator to determine equality) but your data is coming from non-in-memory source.  The use case
I'm most concerned with is when data is being deserialized from a web sockets of HTTP responses.  Such data
comes into the Javascript environment as a completely new JSON instance.  But in some cases, it may be
the case that only a fraction of that data is actually new (as compared to some previously deseralized
value).

This library attempts to efficiently salvage as much as possible of the "current" value in memory rather
than completely replacing it with a newly known value.  I could imagine this library could also be useful
as an operator for an `Observable` in order to salvage parts of previously reported values.

## Getting started

```
npm install salvage
```

In TypesScript, you can import the function with:

```
import { salvage} from 'salvage';
```

From `node`, the simplest thing is probably do do:

```
var salvage = require('salvage').salvage;
```

## Examples

Consider that your current application state is represented by something like this:

```
{
    catalog: [{
        product: "Settlers of Catan",
        price: "$35.27",
    }, {
        product: "Blood Rage",
        price: "$50.60",
    }, ..., {
        product: "Star Wars: Rebellion",
        price: "$76.44",
    }]
}
```

Then imagine that you query your server after a price drop for one item and get back this response:

```
{
    catalog: [{
        product: "Settlers of Catan",
        price: "$35.27",
    }, {
        product: "Blood Rage",
        price: "$38.78",  // <-- Only change
    }, ..., {
        product: "Star Wars: Rebellion",
        price: "$76.44",
    }]
}
```

You wouldn't want to trigger unnecessary redraws.  So, an ideal situation would be to preserve the
items in the `catalog` that didn't change.  By preserve, I mean that they are equal (as in `===`)
to the previous values in the catalog.  In the case of the example above, the value of `catalog` would
change (since the array itself is "new"), but the only element that would have a different (`!==`)
value would be the one for "Blood Rage".  All others would be `===` to their predecessor (even if
elements of the array are inserted or removed).

The API consists of just a single function, `salvage`, which is invoked as follows:

```
let newState = salvage(oldState, deserializedValue);
```

The `salvage` function will preserve as many values from within arrays and objects as possible so
as to minimize the number of new values present in the hierarchy of `newState`.

## Options

When calling `salvage`, you can include an options object that provides hints about how `salvage` should
function.  The main options are related to how array comparisions are done (see discussion below about
Performance).  However, at the moment the default behavior seems to be nearly as good as any of the
optimized settings I've tried.  As such, I suggest using the default settings until I see any cases
"in the wild" where tweaking these settings makes a big improvement.

## Why?

Working with React and Angular2 UIs that depend heavily on `===` equality but, at the same time, working
with data sources that generated data from "out of memory", I wanted to be able to preserve the ability
to use `===` for efficiency.

My guess is that something like this has been done before.  However, I couldn't find such a thing.  This
is probably due to the fact that I'm not sure what Google search terms to use.  Perhaps this kind of
thing has a widely accepted name and I simply don't know what it is.  By all means, if there are libraries
out there that do this kind of thing...please let me know.

## Performance

### Introduction

In order to talk about performance, we need to consider what to compare the performance of this library
to.  The way I look at this, I'm trying to allow frameworks to use object identity as a fast equality
check.  This library is paying an "upfront" cost to allow that by processing successive values to
preserve object identity to the extent possible.  So the point is that running `salvage` can be seen
as an investment to make other operations faster later.

That being said, I see the purpose of `salvage` as a way to perform deep equality checks in other parts
of the code.  The worst case scenario for a deep equality check is comparing an object to another object
that is (deeply) equal because it is, therefore, forced to check the entire hierarchy.  So that provides
a kind of "base line" reference time for traversal of the object hierarchy as a reference point.

So then the question becomes, how does `salvage` compare (in CPU time) to this reference point?  I've added
a performance test to the library test suite.  I'm not sure that it can be considered representative
of anything, but it is at least a non-trivial JSON representation.

### Arrays

The primary issue with performance
at this point is in array handling.  This is because handling of object properties is relatively straight
forward (you compare against the value associated with the same property in the previous object).  But
array handling is different.  That is because changes in arrays might cause values to be inserted or deleted.
This means that two values that are equal may shift around in the array (*i.e.,* have different indices).

So how do we check to see if we can preserve object/value identity from one array value to the next?
The safest and most general approach is to compare each element in the new array value to all values
in the previous array value.  For two "large" arrays, this can be computationally intensive and my
benchmarks to date indicate that you definitely don't want to do that.  For this reason, I've
settled on an approach that uses a "key function" to generate a string "key" value (conceptually
you can think of this as a has value) for each element in the arrays.  In this way, we only bother
comparing elements with matching keys.  This follows very closely how a `key` attribute can be used
in React or the `trackBy` function can be used in Angular 2 to accomplish similar things.

### Results

I have only [one large data set](https://github.com/xogeny/salvage/blob/master/samples/sample1.json)
in my test cases.  Again, I cannot say this is representative.  But I think it is probably reasonable.
As I mentioned before, I feel that performance of the `salvage` function should be compared relative
to a deep equality check.  I established this baseline by using `lodash` to perform a deep equality check
on my [one sample data set](https://github.com/xogeny/salvage/blob/master/samples/sample1.json).  The
follow chart demonstrates how invocations of `salvage` (with a few different options) compared to this
deep equality check...

| Operation               | Time (for equal object) | Time (for different objects) |
| ----------------------- | ----------------------- | ---------------------------- |
| Deep Equality Check (`_.isEqual`) | **13ms** |  |
| `salvage` (default) | **22ms** | 18ms |
| `salvage` (`jsonKey`) | 24ms | 15ms |
| `salvage` (use `_id`) | 30ms | 13ms |
| `salvage` (`sameKey`) | 490ms | 492ms |

The main thing to focus on in this chart is the fact that running `salvage` with its default
options took no more than 70% longer than testing for object equality using the `default` key
function.  You can also see that a couple of other key functions were used and the default key
function is quite performant vs. more case specific key functions.

You might think...oh, deep object equality checks are faster.  So what do I need `salvage` for?

Yes, a simple deep equality check is going to be faster.  But this only tells you whether two values
are the same at the root level.  On the other hand, by running `salvage`, you'll find out **all**
(nested) values that are deep equal.

To understand why this is important, consider a typical React hierarchy of components (and the same
can also be applied in the case of Angular 2 as well).  It is quite common to have a rich hierarchy
of DOM components that cascade some top level value down through the DOM hierarchy delegating nested
values to these components along the way.  These components are going to decide whether they need
to redraw based on the *identity* of their values and whether that identity has changed.  What `salvage`
is helping you with here is preserving the identities whenever possible.  As such, **it is potentially
eliminating the need for multiple, nested deep equality checks and the associated work of rerendering
those components**.

## Types

The main goal of this library is to allow in-memory values to be updated by values that originated
outside of memory.  In practice, this means values that have been reconstituted via some deserialization
process and, that, in practice means JSON.  As such, the focus here is on the JSON primitives: numbers,
strings, `null`, booleans, objects and array.  As a practical matter, I included support for both `Date`
and `function` types (even though those are outside the scope of the JSON spec).  Dates in particular could
be pretty important and could easily be transformed as part of a deserialization process.

This still leaves other types of values like `Buffer`, `Uint8Array`, *etc*. unhandled.  If `salvage` encounters
a value that it doesn't know how to handle, it will throw an exception.

## To Do

I've tried to create a representative set of test cases and I have 100% coverage for the test cases
I have.  But I'd really like to have more test cases to make sure the code is robust to as many
weird corner cases as I can find.

Another potential future enhancement would be to be able to use `salvage` in the context of `Observables`
to preserve identity for `next` values in a stream.