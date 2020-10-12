# Promises

## Table of Contents
* [Callbacks](#callbacks)
    * [Nesting Callbacks](#nesting-callbacks)
    * [Handling Errors](#handling-errors-in-callbacks)
* [Promises](#promises)

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