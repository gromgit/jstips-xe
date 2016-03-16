### Tip XX (Extended Exposition)

One of the suggestions for improvement to the [**DOM event listening made easy**](tip_to_be_published) tip suggests [monkeypatching `HTMLElement` to add jQuery-style `.on()` event registration](https://github.com/loverajoel/jstips/pull/306#issuecomment-196607135). Here are my personal guidelines on safe and unsafe monkeypatching.

### Q: What is "monkeypatching"?

_Monkeypatching_ is basically the practice of _overriding existing functions that you don't control_. You may want to do so for many reasons, including:

- the original function has a bug, but the authors are taking too long to issue a fix, and you feel that editing the original minified code is too risky,
- the function needs some additional functionality to be useful, but the authors disagree with you,
- it's a standard runtime function, and therefore implemented in native code (e.g. adding user-specified format strings to `Date.prototype.toString()`).

I'll extend the definition of _monkeypatching_ to include _adding new functions to objects you don't control_ (e.g. `HTMLElement.prototype.on`, or leaving `toString()` alone and defining `Date.prototype.format(fmt)`).

There are [enough](https://davidwalsh.name/monkey-patching) [monkeypatching](http://me.dt.in.th/page/JavaScript-override/) [howtos](http://raganwald.com/2015/08/08/monkey-patching-extension-methods-bind-operator.html) online; please read those if you're new to this concept.

### DO: Change as little as possible.

The best monkeypatches:

- don't remove any published functionality of the original function,
- call the original function to do most of the work, and
- only add a little code to fix or add some missing functionality.

This way, any other code that calls the monkeypatched function still gets the behavior it expects, thus minimizing logic bugs.

The _Meet the `override` function_ section of [this article](http://me.dt.in.th/page/JavaScript-override/) shows a neat way to do this:

```js
// callback_factory(original) takes the original callback and
// returns a new method that uses the original somehow
function override(object, methodName, callback_factory) {
  object[methodName] = callback_factory(object[methodName])
}

// Tester() defines prototype.saveResults(filepath)...
var test = new Tester()

// ...which we now enhance with additional functionality
override(test, 'saveResults', function(original) {
  return function(filepath) {
    var returnValue = original.apply(this, arguments)
    var planpath = filepath.replace('.xml', '_plan.xml')
    console.log('Save test plan to ' + planpath)
    return returnValue
  }
})

test.run()
```

### DON'T: Change function interfaces (but extending is usually OK).

If the original function looks like:

```js
(string, int, array) => object
```

don't monkeypatch it to:

```js
(int, array, string) => boolean
```

even if you think your definition makes more sense. All calls to this function from third parties (and perhaps even some of your own code) would expect the original interface, and your application will break.

However, _adding_ new arguments is generally OK. For instance, `Date.prototype.toString() => string` could become `Date.prototype.toString(fmt) => string`. If `fmt == undefined`, you simply call the original `toString()`; otherwise, build a date string according to `fmt` and return that instead.

### DO: Document your monkeypatch extensively.

Your monkeypatch changes something in the original API, so you **MUST** document it thoroughly, both in the source code and API documentation. You should include, at minimum:

- what problem is being solved by your monkeypatch
- how does your monkeypatch change the behavior of the base objects/methods
- how to _remove_ your monkeypatch when it's no longer necessary

If the changes are significant, label them as **potentially breaking changes** to remind everyone to pay special attention. This applies especially if your monkeypatch actually _fixes_ the function to comply with the API documentation; if you don't:

- your colleagues may write their own fixes, and the combined fix "stack" will probably do strange things,
- the library author finally fixes the function, and the library upgrade + your forgotten monkeypatch breaks the function again, but in a different way

### DON'T: Monkeypatch new functions to "core objects".

Monkeypatching new methods to "core objects" (e.g. `HTMLElement` and `Object`) is a fairly common way to quickly insert functionality into a large part of your JavaScript environment, but you also run the risk of having your method names clash with other libraries and third-party code. Since these "core objects" are fundamental to your runtime environment, you can't use techniques like [IIFEs](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression) and [ES2015 modules](https://babeljs.io/docs/learn-es2015/#modules) to avoid namespace collisions.

For example, the `HTMLElement` example at the beginning does this:

```js
HTMLElement.prototype.on = function(event,cb) {
  // register 'cb' as 'event's handler
}
```

Then you import a UI library that quietly does this internally:

```js
HTMLElement.prototype.on = function(color) {
  // apply CSS "backlight" to the target (with optional color)
}
```

One will overwrite the other, so you'll see strange behavior like exceptions being thrown, no handlers being registered, etc. Blindly using the `override` function in this scenario will cause even more havoc, since the two functions have different interfaces and "meaning".

Worse, if the library focuses on buttons, perhaps the developer monkeypatched `HTMLButtonElement` instead, which is a child of `HTMLElement`. Now, all your buttons don't respond to clicks, but everything else works perfectly.

### SUMMARY

Most of the downsides in monkeypatching result from one simple fact: you're publicly changing the behavior of objects and functions that _you don't own_. Keeping this in mind should help you monkeypatch with minimal side effects.

- Don't monkeypatch if there are other solutions that are less "clever", but also have less widespread impact on the correctness of your code.
- KISS your monkeypatches - Keep It Short & Simple.
- Reduce the scope of your changes as much as possible.
- Write down what your patch changes, everywhere the original function was also documented.
- Don't monkey with Very Important Objects.
- Only apply the `override` technique to existing functions with standardized behavior.
