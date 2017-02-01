# Three Javascript Async Patterns

Node makes heavy use of asynchronous, non-blocking program flow.  As you know, by default, Javascript is single-threaded (although there are ways around this with web workers or other methods of spawning child processes); however, in most cases, the best practice is to embrace this limitation and make use of one of the many patterns for writing asynchronous code.  This post isn't about the *why* to write non-blocking code, but rather *how*.

The three patterns discussed here are callbacks, promises, and async/await.  There are other patterns as well as multiple variations of each so this post might expand in the future.

## Examples

Here is an example of the same action written in each pattern:

Callbacks:
``` Javascript
doThing('value', (err, result) => {
	if (err) {
		// handle error here
		console.error(err);
	} else {
		// handle success here
		console.log(result);
	}
});
```

Promises:
``` Javascript
doThing('value')
	.then(value => {
		// handle success here
		console.log(value);
	})
	.catch(reason => {
		// handle error here
		console.error(reason);
	});
```

Async/await:
``` Javascript
try {
	const result = await doThing('value');
	// handle success here
	console.log(result);
} catch (ex) {
	// handle error here
	console.error(ex);
}
```

Let's break down each of these patterns.

## Callbacks
A callback is nothing more than a function passed as a parameter to another function.  This function is then called whenever an asynchronous operation has completed.  The body of the callback contains code that is intended to execute after the asynchronous operation.  Because they are simply functions passed as parameters, callbacks are heavily dependent on convention, the most common of which is the "error-first" callback passed as the final parameter.  By virtue of its name, an error-first callback is a function that takes an error object as its first parameter (or a falsey value if no error was encountered) and then any return values as subsequent parameters.

### The Good
Callbacks are a language-level feature.  The Javascript language defines functions as objects that, among other things, can be passed around as parameters.  Because of this, callbacks are supported in every platform/environment/engine without the need for additional packages or transpiling.  A callback can be expected to work the same server-side in Node (V8) as it does client-side in the Edge browser (Chakra).

### The Bad
Two words: callback hell.  All code after an asynchronous operation is nested inside a function or lambda.  More operations means more indentation which quickly eats away at recommended max line widths and makes code much less readable.  The problem becomes more pronounced with larger functions; therefore, the best way to avoid these "pyramids of death" is to keep functions small (which is a good thing anyway).

### The Verdict
Aside from the way the code *looks*, callbacks are the simplest, most portable pattern.  Use them when you are working within environments where you want to keep code small by not adding added libraries or don't have access to a transpile such as Babel.

## Promises
Promises are framework-level features.  While there is a default promise implementation that exists all modern Javascript environments, they are still an executed code artifact rather than native Javascript syntax.

### The Good
Their framework-level status is an asset.  They provide for a broader range functionality than just asynchronously returning a value or an error, for example `Promise.all()` allows for multiple operations to be run in parallel rather than just asynchronously.  Promises also provide an alternate code structure.  Multiple promises can be chained to facilitate multiple asynchronous operations without forcing a new level of indentation after each.  This syntax can lead to more readable code.  Aside from the default promise implementation in Node, there are a number of other promise libraries available that even allow for standard callback-based functions to be "promisified."

### The Bad
Their framework-level status is a liability.  Because of this, there is always an overhead cost in both time and memory usage when using promises instead of callbacks.  [Bluebird](https://www.npmjs.com/package/bluebird) is one of the most performant promise libraries, yet still incurs overhead cost, albeit slightly.  They publish [benchmarks](https://github.com/petkaantonov/bluebird/tree/master/benchmark) of how they stack up against the competition, including baseline callbacks.

Another liability is that, as code rather than as syntax, users are free to implement Promise libraries that operates in any particular way.  And as Murphy's Law states, anything that can happen will be implemented independently in 32 different npm packages.  In my opinion, while a diversity in options is often nice, in the case of promises, it is a barrier to adoption.  In addition to the differences between libraries, the general concept of "resolving" and "rejecting" values is not nearly as intuitive as passing a function that will be called later.  In short, while promises offer more functionality than callbacks, they do so at the cost of being initially harder to learn and use.

### The Verdict
My personal preference when presented a choice between language- and framework-level features that are reasonably similar is to choose the language-level feature.  There are cases (eg, parallelism) where promises offer a large reduction in code and are thus the obvious choice.  In any general case, I find myself using callbacks; in more complex cases I use `Promise.all()`, `Promise.one()`, or other promise functionality.

## Async/Await
Async/await borrows from other "real" languages and facilitates executing asynchronous code with synchronous-looking syntax.  Consider the following example:

``` Javascript
async function myFunction(inputValue) {
	try {
		const a = await asyncFunc1('value');
		const b = await asyncFunc2(a);
		const c = syncFunc3(b);
		return await asyncFunc4(c);
	} catch (ex) {
		// handle exception
	}
}
```

It is obvious that functions 1, 2, and 4 are asynchronous, but the developer doesn't have to waste brain cycles mentally parsing this equivalent, callback-based code:

``` Javascript
function myFunction(inputValue) {
	asyncFunc1('value', (err, a) => {
		if (err) {
			// handle error
		} else {
			asyncFunc2('value', (err, b) => {
				if (err) {
					// handle error
				} else {
					try {
						const c = syncFunc3(b);
						asyncFunc4('value', (err, d) => {
							if (err) {
								// handle error
							} else {
								callback(null, d);
							}
						});
					} catch (ex) {
						// handle error
					}					
				}
			});
		}
	});
}
```

### The Good
Similar to callbacks, it is a language- rather than framework-level feature and allows for the use of other common language features such as try/catch.  When awaiting a promise, rejections are thrown as errors which can then be caught in the same way as in synchronous code.  In doing so, they also allow for simpler, more synchronous-looking code than both callbacks and promises (as shown in the example above).  Under the hood, they are just a new way of calling promises: an "async" function is actually a function that returns a promise.  This means that you can call `.then()` on async functions and `await` promises.

### The Bad
Using async/await requires the use of a transpiler such as Babel.  Setting up such an environment can be a pain and the overhead effort to do it right can often be a sufficient barrier to entry.  Currently, this pattern is just syntactic sugar over the default promise code; as with all ES-next code, the "real" version that our future selves get to work with could theoretically be subtly different (or never be implemented at all).  Lastly, it is a much less common pattern when compared to callbacks and promises and will thus be less familiar to most developers.

### The Verdict
If you have access to an environment with the necessary transpiler tooling, use async/await.  It allows for asynchronous code to be written in a very clear, language-level syntax.  Aside from setting up transpilation, there is much less intellectual overhead in grokking async/await compared to promises.

## My Conclusion
From a purely syntactical perspective, I find async/await to exceed the strengths of both callbacks and promises.  It is a language-level feature, asynchronicity is obvious, error handling is natural, the syntax precludes the callback pyramid of death, and it plays nicely with existing promise patterns.  The only real downside is the need to transpile.  For personal projects, I have solved that problem by creating a [reference project](https://github.com/skonves/esnext-reference) wherein I have already ~~wasted~~ invested all of the necessary hours to get Babel setup.  Feel free to check that out and modify it for your needs.

I am considering doing a more in-depth write up on async/await in the future.
