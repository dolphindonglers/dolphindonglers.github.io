---
layout: post
title: "A first look at JAX"
author: Adi
date: 2025-06-13 17:06:00 -0400
categories: project update
comments: true
unlisted: true
excerpt: "The first real blog post."
---

So my mentor asked me to learn JAX before showing up to RSI.

What is JAX? Well, it's a goofy python library that allows you to do some
pretty interesting things. Seems pretty nifty on the surface, just looking at
the descriptions -- but the one time I saw JAX code, I didn't realize it was
python. Seemed like its own programming language or something.

Anyways, here's some stuff I found by reading through the tutorials. This also
functions as my notes so I don't forget everything I read.

## Some basic stuff

A few intro things that seemed useful to know.

### Arrays

You can make arrays in JAX just like numpy arrays, torch tensors, etc. One
interesting thing though is that they can be split across multiple devices (so
part of the array could be on GPU #1, another part can be on GPU #2, etc). This
is called sharding. Hopefully I won't have to worry about this too much (but it
relates to parallelizing computations, so maybe).

### Tracing

JAX has this construct called a tracer that allows it to track the operations
that occur within a python function. It seems crazy that python code can
actually do this, but I suppose there are ways. Interesting to keep in mind.

### Jaxprs

So apparently JAX is kind of its own programming language, in a way. It can
convert python functions into a goofy looking language called jaxpr. I guess
this probably also relates to how tracing works (although still kind of beats me
how they manage to pull this off).

### RNG

JAX uses something called RNG keys. They seem similar to seeds -- if you use a
certain random key for a random function, the output is always the same. However
you can get a different output by splitting the key into two new keys. The docs
recommend deleting the original key once you split it in order to keep
generating random values.

## JIT compiling

Ok so this kind of explains where the jaxpr language comes into play. You can
use JIT compiling to take a python function and pre-compute a series of
transformations (which makes subsequent calls to the function execute much
faster). However there are some things to note:

- JIT compiled functions are re-compiled on first use and afterwards any time
the input shape changes. When the function is re-compiled it essentially runs
the raw python code.
- The JIT compiler requires pure functions, which essentially means they have to
produce the same output every time for a given input and cannot use or modify
anything outside the function body. In a sense it's like a function from math.
Things that would count as impure functions produce side effects, such as:
  - Uses or modifies a global variable. Whenever the function is compiled the
  current value of the global is cached. Attempting to save a global will
  replace the global with a JAX object.
  - Prints out a value. The print statement will be executed when the function
  is compiled, but not any other time. Useful to know for debugging.
  - Uses an iterator (these create global states apparently. I also didn't know
  that these really existed in python in the first place). This can cause an
  error, or it might just do something unexpected.
- If the function contains a conditional, only the branch that is taken when
the function is compiled is saved. All the other branches of the if/else are not
compiled. There is a different way to use conditionals in JIT functions. Also,
conditionals will fail to compile in JIT if they read the value.
- You can JIT compile part of a function: just isolate it into its own function,
JIT it, and call it, for example if the whole function isn't pure:

{% highlight python linenos %}
import jax
import jax.numpy as jnp

@jax.jit
def inner(x: jax.Array, y: float):
    return x.at[0].add(y)

def conditional_fn(x):
    if jnp.all(jnp.greater(x, 0)):
        inner(x, 7.0)
    else:
        inner(x, -7.0)

print(outer(jnp.array([1.0, 2.0])))  # prints [8.0, 2.0]
print(outer(jnp.array([1.0, -2.0]))) # prints [-6.0, 2.0]
{% endhighlight %}

There was some other stuff but I kind of skimmed over it for now.

## Vectorization

JAX allows you to automatically vectorize a function that takes array inputs by
allowing parallel computation for batched inputs. This is another transform that
takes advantage of jaxprs and tracing. They provide the example of computing the
convolution of two vectors, but since I was confused by it, here's a simpler
example (multiplies the [0][0] element of each matrix):

{% highlight python linenos %}
import jax
import jax.numpy as jnp

def thing(x, y):
    a.at[0, 0].multiply(b.at[0, 0].get())

# If we had two 2x2 matrices, this is easy:
a = jnp.array([[1, 2], [3, 4]])
b = jnp.array([[5, 6], [7, 8]])
print(thing(a, b))

# Output:
# Array([[5, 2],
#        [3, 4]], dtype=int32)

# But if we wanted to compute this for a batch:
c = jnp.tile(a, (3, 1, 1))
d = jnp.tile(b, (3, 1, 1))

# We can vectorize our function:
thing_vec = jax.vmap(thing)
print(thing_vec(c, d))

# Output:
# Array([[[5, 2],
#         [3, 4]],
# 
#        [[5, 2],
#         [3, 4]],
# 
#        [[5, 2],
#         [3, 4]]], dtype=int32)
{% endhighlight %}

It uses 0 as the batch dimension by default, but you can specify a different one
for eithr the input or the output (or both).

You can also chain `jax.jit()` with `jax.vmap()`.

## Autodiff

Ok, this is the cool stuff. You can use JAX to compute the gradient of a python
function.

For example, let's say we have a function $f(x) = x^2 + 7^x$. We then know that:
<div>$$
\begin{align}
f'(x) &= 2x + 7^x (\ln 7) \\
f''(x) &= 2 + 7^x (\ln 7)^2
\end{align}
$$</div>
and so on. We can do this in python with JAX:

{% highlight python linenos %}
import jax

def f(x):
    return x ** 2 + 7 ** x

fp = jax.grad(f)
fpp = jax.grad(fp)

# and so on.
{% endhighlight %}

Of course the real utility is for computing gradients of more complex functions.
Pretty cool stuff!
