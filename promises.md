# Promises

## Table of Contents
* [Callbacks](#callbacks)
    * [Nesting Callbacks](#nesting-callbacks)
    * [Handling Errors](#handling-errors-in-callbacks)
* [Promises](#promises)
    * [Promise Constructor](#promise-constructor)
    * [Promise Producers](#promise-producers)
    * [Promise Consumers](#promise-consumers)
* [Promise Chaining](#promise-chaining)
    * [Returning Promises](#returning-promises)
* [Error Handling](#error-handling)
* [Promise API](#promise-api)

## Callbacks

JavaScript is typically run in the host environment of a browser, and the browser allows the JavaScript programmer to schedule **asynchronous** actions.

A common one is adding a script tag to the DOM.

```javascript
function loadScript(src) {
    let script = document.createElement('script');
    script.src = src;
    document.head.append(script);
}
```

This function is called, the code is executed, and the script is fetched from some remote area. The code below this call is not blocked though. 

This can cause problems when you want to use JavaScript functions that are included in that script.

```javascript
loadScript('cool_script.js');

// 💀💀💀 the browser didn't have time to load the script
func_from_cool();

```

One way to handle this kind of problem caused by the asynchronous nature of the code is to add a **callback** to the function.

```javascript
function loadScript(src, callback) {
    let script = document.createElement('script');
    script.src = src;

    script.onload = () => callback(script);

    document.head.append(script);
}
```

Now our `func_from_cool` call will work.

```javascript
loadScript('cool_script.js', function() {
    // the callback runs after the script has loaded
    func_from_cool();
})

```
This type of asynchronous programming is called **"callback-based" programming**. A function that does something asynchronously **should provide a callback argument where we put the function to run after it is complete.**

### Nesting Callbacks

What about if we want to load two scripts sequentially, first one, and then the other?

We can just nest the second script inside the callback.

```javascript
loadScript('cool_script.js', function(script) {
    // the callback runs after the script has loaded
    func_from_cool();
    loadScript('even_cooler.js', function(script) {
        alert("yep, the even cooler script has loaded");
    });
})

```

Nice, this will work, but what if we want more scripts? Okay, I guess you can start to see that this will be turtles all the way down.  🐢🌎. This pattern can be unwieldy with many callbacks.

### Handling errors in callbacks

Let's add some code for handling errors in the asynchronous call.

```javascript
function loadScript(src, callback) {
    let script = document.createElement('script');
    script.src = src;

    script.onload = () => callback(null, script);
    script.onerror = () => callback(new Error(`There was an error loading ${src}`));

    document.head.append(script);
}
```
For a successful load, it calls `callback(null, script)`and `callback(error)` otherwise.

One thing to note is that for every subsequent nested function we will need to add another argument. `callback(null, result1, result2....)`.

Anyway, the point is that this kind of coding is possible, and intuitive, but can get problematic. Especially when the dependent code is complicated.

One way to alleviate this problem is to make every action a standalone function:

```javascript
loadScript('1.js', step1);

function step1(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...
    loadScript('2.js', step2);
  }
}

function step2(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...
    loadScript('3.js', step3);
  }
}

function step3(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...continue after all scripts are loaded (*)
  }
};
```

It does the same thing as the previous code, but now there is no deep nesting. Unfortunately though, this code is harder to read. Also, each of these step functions is not general. There is going to be a lot of clutter with this strategy. So what's the better solution? I promise I'll tell you. 😏

## Promises

Promises follow a pattern we often see in programming:

1. A **producer** does something that takes time. 
2. A **consumer** wants the result of the producer.

A **promise** is a a JavaScript object that links the producing and consuming code together. It allows the producer to take as much time as it needs to produce the promised result, and then it makes these results available to all the subscribed consumers.

### Promise Constructor

The constructor syntax for a promise object is:

```javascript
let promise =  new Promise(function(resolve, reject) {
    //executor
})
```

The function passed to `new Promise` is called the **executor**. When a new promise is created, the executor **runs automatically**. It **contains the producing code** which should eventually produce a result.

The executor's arguments: `resolve` and `reject`, are callbacks provided by JavaScript itself. When the executor obtains the result, it should call one of these callbacks.

* **resolve(value)** - if the job finished successfully.
* **reject(error)** - if an error occurred.

The promise object returned by the `new Promise` constructor has some internal properties:

* **state** - initially `pending`, then changed to either `fulfilled` when resolve is called, or `rejected` when reject is called.
* **result** - initially undefined, then changes to `value` when resolve is called, or `error` when reject is called. 

### Promise Producers

```javascript
let promise = new Promise(function(resolve, reject) {

  // after 1 second signal that the job is done with the result "done"
  setTimeout(() => resolve("done"), 1000);
});
```

```javascript
let promise = new Promise(function(resolve, reject) {
  // after 1 second signal that the job is finished with an error
  setTimeout(() => reject(new Error("Whoops!")), 1000);
});
```

The executor should only call one `resolve` or one `reject` call. Any state change is final, all the other calls are ignored.

### Promise Consumers

A promise object is the link between the executor (the consumer) and the consumers, which can receive results or an error. **Consuming functions can be registered** using the methods `.then`, `.catch`, or `.finally`. 

#### Then

```javascript
promise.then(
    function(result) {/* Handle a successful result*/},
    function(error) {/*Handle an error*/}
);
```

The first argument to `.then` is a function that runs when the promise is resolved and receives the result. The second argument of `.then` is a function that runs when the promise is rejected, and receives an error. 

Let's look at a full example of a successfully resolved promise

```javascript
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve("done!"), 1000);
});

// resolve runs the first function in .then
promise.then(
  result => alert(result), // shows "done!" after 1 second
  error => alert(error) // doesn't run
);
```

If we're interested in only successful completions then we can provide only one argument to `.then`.

#### Catch

If we're interested in only errors, then we can pass null as the first argument to `.then`, or we can use the `.catch` function.

```javascript
let promise = new Promise((resolve, reject) => {
  setTimeout(() => reject(new Error("Whoops!")), 1000);
});

promise.catch(alert); 
```

#### Finally

Just like there is a `finally` clause in try/catch blocks. Finally runs no matter what. It is a good clause to use for performing cleanup, stopping or loading indicators, etc.

With finally we don't know if the process is successful or not. It passes through results and errors to the next handler

```javascript
new Promise((resolve, reject) => {
  /* do something that takes time, and then call resolve/reject */
})
  // runs when the promise is settled, doesn't matter successfully or not
  .finally(() => stop loading indicator)
  // so the loading indicator is always stopped before we process the result/error
  .then(result => show result, err => show error)
```
## Promise Chaining

So, let's return to the problem we were motivated by when we were talking about callbacks. How do promises structure a sequence of async tasks?


```javascript
new Promise(function(resolve, reject) {
    setTimeout(() => resolve(1), 1000);
}).then(function(result) {
    alert(result); //1
    return result * 2;
}).then(function(result) {
    alert(result); //2
    return result * 2;
}).then(function(result) {
    alert(result); //4
    return result * 2;
});
```

The result is passed through the chain of `.then` handlers. This whole thing works because a call to `promise.then` return sa promise, so that we can call the next `.then`.

A classic promise error is when a programmer adds many `.then` calls to a single promise instead of chaining the calls.

```javascript
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});
```
What the code above does is add several handlers to one promise. They don't pass results to each other, they process it independently.

### Returning Promises

A handler, used in `.then` may create and return a promise. In this case, the handlers have to wait until it completes resolving. **Returning promises allows us to chain asynchronous actions.** 

```javascript
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result); // 1

  return new Promise((resolve, reject) => { // (*)
    setTimeout(() => resolve(result * 2), 1000);
  });

}).then(function(result) { // (**)

  alert(result); // 2

  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(result * 2), 1000);
  });

}).then(function(result) {

  alert(result); // 4

});
```

### Promise Job Queue

```javascript
let promise = Promise.resolve();

promise.then(() => alert("promise done!"));

alert("code finished"); // this alert shows first
```

Promise handling is always asynchronous because all promise actions are passed through the internal "promises job" queue, also called the microtask queue.

This means that all `.then/.catch/.finally` handlers are run after the current code is finished. That means that if we need to guaranttee that a piece of code is executed after `.then/.catch/.finally`, we need to add it to a chained `.then` call.

The queue is a first-in-first-out queue, and execution of a task is initiated only when nothing else is running.

## Error Handling

When a **promise rejects**, the **control jumps to the closest rejection handler**.

```javascript
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
  .then(githubUser => new Promise((resolve, reject) => {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
      resolve(githubUser);
    }, 3000);
  }))
  .catch(error => alert(error.message));
```

Normally, the catch wouldn't trigger, but if there are any errors, it will throw an alert.

So how does it work? The code of the promise executor has an invisible try/catch around it. If an exception happens, it gets caught and treated as a rejection.

## Promise API


1. **Promise.all(promises)** – waits for all promises to resolve and returns an array of their results. If any of the given promises rejects, it becomes the error of Promise.all, and all other results are ignored.
2. **Promise.allSettled(promises)** (recently added method) – waits for all promises to settle and returns their results as an array of objects with:
    status: "fulfilled" or "rejected"
    value (if fulfilled) or reason (if rejected).
3. **Promise.race(promises)** – waits for the first promise to settle, and its result/error becomes the outcome.
4. **Promise.resolve(value)** – makes a resolved promise with the given value.
5. **Promise.reject(error)** – makes a rejected promise with the given error.