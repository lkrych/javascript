# Promises

## Table of Contents
* [Callbacks](#callbacks)
    * [Nesting Callbacks](#nesting-callbacks)
    * [Handling Errors](#handling-errors-in-callbacks)
* [Promises](#promises)
    * [Promise Constructor](#promise-constructor)
    * [Promise Producers](#promise-producers)
    * [Promise Consumers](#promise-consumers)

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