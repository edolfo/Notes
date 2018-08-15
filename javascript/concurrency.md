# Notes on Javascript

Javascript is typically implemented as a single-threaded language, and although there are methods of concurrency 
(fibers, spawning processes on a server, webworkers in a browser), for the most part people work in the single 
thread that is allocated to them.  What this means for you, the programmer, is if you want your application to 
remain responsive (in the 'is it still thinking or did it crash?' sense), you'll need to offload long running 
tasks into the background for the interpreter to pick up again on their completion.

I can hear your objections now:

> But....wait...isn't javascript single-threaded?

> Offloading tasks to the background - doesn't that require at least two threads or two processes?

Yes, that's all true - the single-threaded environment and the requirement for some method of concurrency to 
support the notion of 'offloading tasks'.  The resolution to this seeming contradiction lies in the implementation 
of the interpreter.  For example, with node.js, tasks that are told to process in the background are actually 
sent to the operating system's kernel.  Once the task is complete, the kernel informs node.js, and node.js adds 
the event to its' own internal queue.

So how does one manage to keep track of all this?  A good rule of thumb is to assume the 'underlying system' 
(be it the OS kernel, the JVM, the browser's javascript engine, ...) takes care of the management regarding 
background processing for asynchronous functions, and will inform you once that function finishes.

Oh, and one more thing here - I've mentioned a function *"finishing"* a few times now.  This is pretty vague, 
and really means that one of two things has happened:

1. The function has returned, either by hitting the `return` keyword, or by finishing all it was told to do.
2. The function has prematurely exited because it ran into an error, and has now thrown the error.  
Perhaps you caught the error, perhaps not.  In any case, it definitely got thrown.

> OK,great, enough theory already - show me some examples!

Sounds good, let's start with the two cases above, just so we're clear with one another.

### 1. The function has returned
```javascript
function do_nothing() {
    return;  // Explicitly returns...nothing (technically `undefined`)
}

function hi() {
    return 'hi';  // Explicitly returns a string 'hi'
}

function lost_to_the_void() {
    let a = 0;
    a += 5;
}  // Implicitly returns...nothing (technically `undefined`)

function return_null() {
    return null;  // Explicitly returns null, which is not the same as nothing
}
```

In the examples above wherein the function returns `undefined`, if you try this in your own interpreter, 
inside the REPL, you may or may not see that `undefined` is actually returned.  Seeing the return value
as `undefined` is dependent upon your specific environment - typically the implementation of your interpreter.

However, if you want to definitely see for yourself, you can simply do something like this:

```javascript
function do_nothing() {
    return;
}

let return_value = do_nothing();
console.log(return_value);  // prints out 'undefined'
```

### 2. The function has run into an error

```javascript
function fail() {
    return true = "We've always been at war with Eastasia."
}
```

Running this should yield a stacktrace of some sort telling you exactly how boneheaded of a monkey you are.
My interpreter even ignored the sin of omission with respect to the semicolon!  Here's a copy of mine:

```
> function fail() { return true = "We've always been at war with Eastasia." }
repl:1
function fail() { return true = "We've always been at war with Eastasia." }
                         ^^^^

ReferenceError: Invalid left-hand side in assignment
    at Object.createScript (vm.js:80:10)
    at REPLServer.defaultEval (repl.js:195:21)
    at bound (domain.js:301:14)
    at REPLServer.runBound [as eval] (domain.js:314:12)
    at REPLServer.onLine (repl.js:468:10)
    at emitOne (events.js:121:20)
    at REPLServer.emit (events.js:211:7)
    at REPLServer.Interface._onLine (readline.js:280:10)
    at REPLServer.Interface._line (readline.js:629:8)
    at REPLServer.Interface._ttyWrite (readline.js:910:14)
```

Had I caught the error, I could have done the responsible thing and ignored it.  Oops, I mean, logged the error
and possibly retried the computation, informed the user, or informed the programmer, whatever the system design
called for.

> **WOW**, all this and no examples on concurrency yet, in a document about concurrency.

Yeah, I get it.  I can get sidetracked on the details.  Let's move onto concurrency strategies with some examples.

## Callbacks

OK, look, callbacks aren't all bad.  Yeah, you might have heard of the so-called `callback hell`.  Hell, you may
have even seen it in the wild, or...dare I say it?  You might have even crafted it yourself!  I know I sure did,
but as long as I promise myself to not repeat the mistakes of five minutes ago, I should be golden.

```javascript
// Callback hell
function hello_overly_complicated_world() {
    setTimeout(function() {
        setTimeout(function(){
            setTimeout(function(){
                setTimeout(function(){
                    console.log('world');
                }, 100);
                console.log('complicated');
            }, 100);
            console.log('overly');
        }, 100);
        console.log('Hello');
    }, 100);
}
```

To run this program:

```javscript
hello_overly_complicated_world()
```

With the formatting, it's somewhat readable.  We can kind of figure out which `100` belongs to which level.
With some thinking, we can verify to ourselves that it prints out 'Hello overly complicated world', and not
'world complicated overly Hello'.  But we have to work at it.  And that's the whole point - code should be
first and foremost, readable.  This is still readable, but only technically so.  It's not an easy read, and
it's not something I'd be thrilled to see if it came up in a code review.  It's like Bud Light - technically
beer, but why not just drink plain water instead?

And that's with a simple example, where we just have a single print statement to execute.  Imagine if we added
three more layers of callbacks.  Imagine if each callback had multiple lines of logic to go through, nested `for`
loops, conditional `if-else` statements, the works!  The above example quickly degrades from Bud Light to toilet
water, in the best case scenario.

Now imagine you have an error in one of those `for` loops, and you have many `for` loops sprinkled throughout
this personal seven-level hell.  Won't someone think of the work the stacktrace printer has to do?  Unraveling 
all of that gobbledygook - the poor computer.  OK, screw the computer, what about my poor stupid monkey brain?
Not only do I have an error...somewhere...it's also located somewhere in this barely-readable mass of a mental 
dump.  It's OK, I didn't want to go outside today, really.  I wanted to spend today trying to gingerly pick
through the mass with gloved hands.

But wait!  There is a better way, and it doesn't even involve bargaining with a devil-figure at an unmarked 
crossroads in a dusty backroad in Georgia!  Actually, there are many 'better ways', some of which have widespread 
popularity in other languages (e.g. actors in Erlang and Scala).  However, this is a javascript-focused document,
and so we will focus on two methods.

### Callbacks and named functions

Back when the javscript community was still trying to figure out the 'best' way (for some definition of 'best') 
to untangle the mess of callback hell, when webpack was still in infancy, and the front-end frameworks were
still figuring things out, there was a minority opinion about keeping code as vanilla as possible.  It appealed
to the minimalist in me, as it required no additional third-party code (e.g. if you wanted to use promises, 
you needed to choose one of the many promise libraries as a dependency).

This minority opinion said yes, callback hell sucks!  But don't blame callbacks!  Blame your stupid monkey
brains for blindly following an anti-pattern!  OK, they didn't say that last bit - that's just me.  But the point
still stands - callbacks aren't inherently bad, but they certainly make bad usage fairly easy.

The core idea was also simple - don't declare anonymous functions inline.  Instead, use named functions and
call those.  Let's take the above 'Hello overly complicated world' example from above and flatten it out with this
method.

```javascript
function hello() {
  setTimeout(() => {
    console.log('Hello');
    overly();
  }, 100);
}

function overly() {
  setTimeout(() => {
    console.log('overly');
    complicated();
  }, 100);
}

function complicated() {
  setTimeout(() => {
    console.log('complicated');
    world();
  }, 100);
}

function world() {
  setTimeout(() => {
    console.log('world');
  }, 100);
}
```

To run this program, we need only call the first function, since each function knows what function to call next:

```javascript
hello();
```

OK, right off the bat I can see you scrutinizing this and noticing that I've switched out the anonymous function 
declaration syntax from `function() {}` to `() => {}`.  This is done purely for illustrative purposes - to show
how the above example avoids the indentation/nesting hell.  Put another way, if you relax your eyes a bit and let
your vision blur, it's a bit easier to see that the structure is much more complicated in the first example than
with this second example.

Now, we still get the same execution order (as long as we start by calling `hello()`), but when we go to change,
say, the timeout on `world()` from 100ms to 1s, we don't have to go rooting around for which brace lines up with
which function.  We just go to the `world()` function, and we know **immediately** that the `100` there is exactly
the `100` we want to change to `1000`.  **I cannot stress this enough** - the time saved here is also saved mental
capacity, which is reduced frustration, which increases overall happiness of the human monkey staring at the 
glowing rectangular panel while pushing buttons on the keyboard.

Imagine now that we actually have complicated logic in one or more of these functions - nested `for` loops, 
conditional `if-else` statements, the works!  You can happily add this complexity and continue drinking your beer
of choice without worrying that you might spill some of your beer into your neighbor function's beer.  

What if you have an error in here?  Well, thankfully, because you've separated each of the functions, you can
easily focus on the offending function and ignore the rest of the good little ~~children~~ functions.  For example:

```javascript
// Error handling

function world() {
  setTimeout(() => {
    try {
      console.log('world');  // Hopefully this doesn't actually fail
    } catch (error) {
      console.error(error);  // Maybe do something more?
    }
  }, 100);
}
```

Plus - no need for additional libraries!

OK, admittedly, with modern javascript, you can get promises now without additional libraries...but this approach
is still an elegant way to radically simplify a problem, which is all good in my proverbial book.  Speaking of promises...

### Promises

Now, with modern javascript interpreters, you can make use of promises *without* depending on a third-party library.  
Hooray!

If you want an in-depth discussion on promises, I recommend 
[Mozilla's documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).
However, since these are just meant to be notes, I won't get into as much detail.

Let's start by rewriting the 'Hello overly complicated world' program with promises:

```javascript
// A fairly literal translation of the initial program, without making much use of nice things in promise-land
function hello() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log('hello');
      resolve();
    });
  });
}

function overly() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log('overly');
      resolve();
    });
  });
}

function complicated() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log('complicated');
      resolve();
    }, 100)
  });
}

function world() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log('world');
      resolve();
    }, 100);
  });
}
````
Looks like a lot more boilerplate (more typing...aghhhh!), with not much to show for it.  Well, that's true 
(mostly), but as mentioned above, we aren't really making full use of what promise-land offers.  However, one
important thing to note here is we have now *removed knowledge of the **pipeline ordering** from the 
**implementation** of the pipeline*.  Or, more concretely, with the named callbacks, that example had `hello()`
knowing that it was to call `overly()` next.  The function `overly()` knew it was supposed to call `complicated()`
next, and so on.  Here, we have created a *pipeline* of function calls, and because of the way we've written the
functions (each accepts and returns the same number of arguments, in the same order, with the same types), we can
hook up each function in whatever order we please.

Hence, to call the original program, since each function has no idea what to call next, we need to tell the interpreter:

```javascript
hello().then(overly).then(complicated).then(world);

// However, this is typically written as such:
hello()
.then(overly)
.then(complicated)
.then(world);
```

But just to illustrate the point, if we wanted to modify the program to print out `world complicated overly Hello`, 
then we instead call:

```javascript
world()
.then(complicated)
.then(overly)
.then(hello);
```

What about errors?  OK, this is something that nearly all, if not all available documentation sort of glosses over.
Here's the typical pattern that is shown:

```javascript
hello()
.then(overly)
.then(complicated)
.then(world)
.catch(error => console.error(error));
```

If an error comes up, where's your error?  Is it in the `hello()` function, the `overly()` function, the 
`complicated()` function, or the `world()` function?  Sure, you can pore over the stack trace if you omit the
final `catch()` function in an attempt to trace down where you attempted to, say, access an `undefined` property 
of `[object Object]`.  Or, you can have your `catch()` function tell you where your offending bits are:

```javascript
hello()
.catch(err => console.error(`Failed in hello!  Error was ${err}`))
.then(overly)
.catch(err => console.error(`Failed in overly!  Error was ${err}`))
.then(complicated)
.catch(err => console.error(`Failed in complicated!  Error was ${err}`))
.then(world)
.catch(err => console.error(`Failed in world!  Error was ${err}`))
```

Your pipeline isn't as clean as it once was, but neither is the real-world.  And just like the real-world. it's
pragmatic, and lets you know exactly where things failed without having to rely on monkey brain memories to put
back the deleted `catch()` function at the end so you'd get a stack trace printed out.  Of course, this is only
possible because all promise methods are chainable.

OK, so now what about all that extra boilerplate that went unused?  Let's try a last rewrite to make everything
come together:

```javascript
function hello() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('hello');
    });
  });
}

function overly(input_str) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log(input_str);
      resolve('overly');
    });
  });
}

function complicated(input_str) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log(input_str);
      resolve('complicated');
    }, 100)
  });
}

function world(input_str) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (input_str == '') {
        reject('Received nothing as input');
      }
      console.log(input_str);
      console.log('world')
      resolve();
    }, 100);
  });
}
```

Calling this program is much the same as before:

```javascript
hello()
.then(overly)
.then(complicated)
.then(world);
```

Hmm...We've lost some of the beauty of the first promise example - our functions are no longer symmetric like
they once were.  The `hello()` function takes no inputs and prints nothing itself.  The `world()` function
prints two things and returns nothing.  However, the `overly()` and `complicated()` functions remain symmetrical.
Chalk it up to first-pass woes!

However, it does demonstrate what can be done with the `resolve()` argument of the promise, and the `world()` 
function illustrates how to throw errors with the `reject()` argument of the promise.

Let's look at the `resolve()` portion first.  This function allows the programmer to have individual functions return
one or more variables through the machinery that is promise-land.  As we can see with the function signatures of
`overly()`, `complicated()`, and `world()`, they each accept a single argument as input, which is generally hoped to 
be a string, but not necessarily required.

Additionally, even though we've added inputs and outputs to (some) of the functions, the construction of the 
pipeline **remains invariant!**.  Yet another way in which the promise-land machinery provides an abstraction 
from the implementation of a function, and the calling of that function in the context of some greater...context.  
Come back to me on that, I can do better.

The `reject()` portion is useful for a few reasons:

1. You want to stop the execution of the pipeline
2. You want to have more detailed/granular/modular error reporting
3. You want to do additional checking/validation outside of what the interpreter *technically* allows you to do.

The first reason is probably the most important to note - if the `reject()` argument is invoked, 
**the remainder of the promise chain/pipeline is cancelled**.  
