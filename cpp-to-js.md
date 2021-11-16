# My experience coming from C++ to JavaScript

Having been with C++ for a long time, I'd become comfortable with it. I'd use it both as a scripting language and for building complex desktop applications. Adding [Qt](https://www.qt.io/) gave me powerful tools for creating user interfaces; and I had access to [OpenCL](https://www.khronos.org/opencl/) for accelerated computing, [OpenGL](https://www.opengl.org/) for real-time graphics, [OpenCV](https://opencv.org/) for image processing, etc. For my needs, this ecosystem worked very well for a long time.

But over time, pressure mounted for me to expand into the web development sphere. So I started learning JavaScript.

In this post, I'll go over some of the experiences I had coming from C++ to JavaScript, including some of the things that went well and some that didn't. I'll try to give a few pointers to people who're planning to do the same, and hopefully some perspectives to JavaScript developers who might be interested in C++.

## First impressions

I had some prior familiarity with HTML and CSS from using them to style Qt components and from dabbling in simple web pages, but I'd never done anything with JavaScript. By this time the language was a couple decades old; the specification was up to about ES9.

JavaScript's syntax is very similar to C++, to the point that the differences feel dialectal. You get your familiar statements like *if*, *for*, and *switch*; you define functions much the same way (just with dynamic typing); and especially if you've written C++ in a procedural style, your program flow may be very similar.

But although the syntactic similarity can ease the transitioning from one language to the other, it's in some ways a hinderance: things can appear familiar but end up working in an unexpected way, quickly landing you in one of the pitfalls of JavaScript.

## The initial pitfalls

There were three notable pitfalls I got to experience when getting started with JavaScript:

1. Behavior of array indices
2. Deep vs. shallow copies
3. The 'this' keyword

They all had to do with what on the syntactic level looked like familiar concepts from C++ but in practice turned out to work quite differently.

### Behavior of array indices

On the surface, indexing arrays works the same way in C++ and JavaScript: given an array `array`, you can access its elements with `array[0...n]`. And this indeed holds true if you only use (positive) integers for the array index. But as soon as you do something like `array[x / 2]`, you'll run into divergent behavior.

In C++, the result of `x / 2` - where `x` is an integer - is an integer. When you write `array[3 / 2]`, you'll be getting `array[1]`, and you're probably very used to it being this way.

In JavaScript, different rules apply. First, the language has no integers, only floating-point numbers. Second, array indices are converted into strings. That means `3 / 2` produces the floating-point value 1.5, which means `array[3 / 2]` gives you `array["1.5"]`, which in turn will equal `undefined` in the likely case that the array has no element with the key "1.5".

For example, in C++:

```c++
const std::array array = {10, 11, 12};
// array[1] == 11
// array[3 / 2] == array[1] == 11
```

In JavaScript:

```javascript
const array = [10, 11, 12];
// array[1] === array["1"] === 11
// array[3 / 2] === array["1.5"] === undefined (oops)
```

This isn't a big problem at the end of the day because you can coax the index value into an integer (e.g. `array[Math.floor(3 / 2)] === array["1"]`). However, unlike a C++ compiler, a JavaScript engine probably won't warn you about using a floating-point value as an array index, so until you notice it manually, you'll get a silent NaN propagating in your program from what at first glance looks to be well-behaved code.

Read more about arrays in JavaScript [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array).

### Deep vs. shallow copies

In C++, I had gotten used to being able to deeply copy plain structs with direct assignment:

```c++
struct pod_structure
{
    int array[3];
};

pod_structure object = {10, 11, 12};

// This copies by value, i.e. makes a deep copy.
pod_structure objectCopy = object;

// Since the copy is deep, modifying it doesn't modify the original.
objectCopy.array[0] = 99;
// object.array[0] == 10
// objectCopy.array[0] == 99
```

This didn't translate well to JavaScript, where objects are copied by reference:

```javascript
function data_structure(array) {
    return {
        array: [...array]
    }
};

const object = data_structure([10, 11, 12]);

// This copies by reference, i.e. makes a shallow copy.
const objectCopy = object;

// The copy is shallow, so modifying it modifies the original.
objectCopy.array[0] = 99;
// object.array[0] === 99
// objectCopy.array[0] === 99
```

I remember being somewhat stunned when I first ported a piece of C++ code to JavaScript and saw that making a deep copy of a plain data structure was actually a bit of a rabbit's hole. Direct assignment typically but not always copies by reference (JavaScript makes no syntactic distinction between data and reference, so you'll need to know the spec), certain methods like `Object.assign()` create copies that are partially deep and partially shallow, and semi-official deep-copying kludges like `objectCopy = JSON.parse(JSON.stringify(object))` also have their pitfalls (e.g. `JSON.stringify()` will silently omit JSON-incompatible data).

One way to get around this issue - a way I ended up favoring - is to use immutable objects, so that you don't need to care whether a copy is a reference. If you want to modify an object, you create a new immutable object with those modifications, and references to the original object remain as they were. The downside is that this isn't fully straightforward to accomplish either - you could use the standard `Object.freeze()` method to make an object immutable, but since it doesn't freeze nested objects, you'd have to go an extra mile to ensure true immutability down the whole property tree.

(As of late 2021, although the landscape of deep copying in JavaScript remains somewhat messy, support for a standard [structuredClone()](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone) method has begun to land in browsers.)

### The 'this' keyword

C++ and JavaScript both provide a `this` keyword, and in both languages it represents a reference to a particular context. But the mechanics of it can differ in painful ways.

In C++, `this` evaluates to a pointer that refers to an instance of the class (or struct) in which it's used:

```c++
struct object
{
    object* self()
    {
        return this;
    }
};

object instance;
// instance.self() == &instance == 0x...

// Can't use 'this' outside of a class context.
std::cout << this;
// Compiler: ^~~~ error: invalid use of 'this' in non-member function
```

In JavaScript, `this` refers to an execution context and has a much broader domain both functionally and in what values it can represent:

```javascript
function say_this() {
    console.log(this);
}

const object = {
    initial_this: this,
    say_this: function(){console.log(this)},
    say_this_arrow: ()=>console.log(this)
};

console.log(this);
// "Window {...}"

console.log(object.initial_this);
// "Window {...}"

object.say_this();
// "{initial_this: Window, say_this: f, ...}"

object.say_this_arrow();
// "Window {...}"

say_this();
// "undefined" (in strict mode) or "Window {...}"

new say_this();
// "say_this {}"

say_this.call(document.head);
// "<head>...</head>"
```

The `this` of C++ is semantically telling you, "this instance of the class". The `this` of JavaScript likewise appears to say, "this context that I'm in", but means to say, "this branch of execution", which in turn depends on factors that aren't visible in the local semantic context. So if you use `this` inside an object in JavaScript, you might expect it to refer to the object itself, but in reality it may refer to (a) the object, (b) nothing, or (c) anything, depending on how the object is being accessed. For this reason, especially if you're used to how it works in C++, it's very easy to mess up `this` in JavaScript.

I'd recommend ignoring what you know about C++ in this regard and even to avoid using JavaScript's `this` altogether until you're sure you understand what it would do - and why so - in the particular situation you'd use it in. Otherwise it's too easy to be creating fragile, hard to understand code.

Read more about JavaScript's `this` keyword [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this).

## Time to refactor?

After a while of building things with JavaScript, I'd become settled into certain patterns, some of which I later felt best to move away from and some that I chose to keep. Among those patterns have been:

1. C++-isms
2. Compensating for the lack of static typing
3. Easing into the messy package ecosystem

### C++-isms

One of the things I most often have found myself refactoring in my early JavaScript code are its in-code interfaces. Often they've had a contextually unpleasant C++ tinge to them; they've felt clunky or out of place in JavaScript.

For example, since C++ doesn't allow named function arguments, you only declare paremeter names in the function signature and then call the function without re-declaring the names:

```c++
int sum(const std::vector<int> &numbers = {}, const int initialValue = 0)
{
    // ...
}

sum({1, 2, 3}, 5);
```

I'd port code like that to JavaScript almost verbatim:

```javascript
function sum(numbers = [], initialValue = 0) {
    // ...
}

sum([1, 2, 3], 5);
```

The result isn't outright bad, but in neither case does the call to `sum()` tell the reader what the arguments mean. The person could assume that the list of values provides the numbers to be summed, and maybe by convention that `5` is the sum's initial value. But they'd need to look up the function signature to be sure.

By finagling with JavaScript's syntax (in this case, [object destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#object_destructuring)) you can have named arguments:

```javascript
function sum({numbers = [], initialValue = 0} = {}) {
    // ...
}

sum({numbers: [1, 2, 3], initialValue: 5});
```

I think the call to `sum()` now looks and feels more like JavaScript. Even if you disagree or prefer the C++-like way, you'll probably agree that the C++-style call is more obscure about intent. Besides being more explicit, the destructured version is more flexible to call: you can re-arrange the arguments, and default value initialization doesn't require a certain argument order either.

### Compensating for the lack of static typing

I'd been in the habit of using assertions in C++ to ensure - among other things - the validity of arguments passed to functions. Since JavaScript doesn't support static typing like C++ does, my asserting extended naturally to checking arguments' types as well. Something like this:

```javascript
function func(arg) {
    console.assert(typeof arg == "number", "'arg' must be of type 'number'");
}
```

Sometimes I'd feel I was being overly pedantic writing type checks in a language made to be dynamically typed. Not to mention each assertion took up more lines of code, required more time to write, and needed to be maintained when the intended type of an argument was changed. Maybe it was an anti-pattern?

Well, I've so far decided not to refactor away assertions from my JavaScript code. In fact, I still keep adding them to new code. Consider the following:

```javascript
function func(arg) {
    console.assert(typeof arg == "number", "'arg' must be of type 'number'");
}

const array = [1, 2, 3];

func(array[1]);
// Runs OK (arg === 2, which is fine)

func(array[1 / 2]);
// Assertion failure (arg === undefined, the array has no key "0.5")

func("1");
// Assertion failure (arg is not a number by strict comparison)
```

The second call to `func()` makes the mistake of assuming that integer division in JavaScript results in an integer. The assertion immediately catches the mistake.

The third call to `func()` is OK in the sense that the function could probably implicitly cast the string into a number and then execute correctly. But still, the function expects a number, so its receiving a string indicates that some other part of the codebase is misbehaving (maybe it's passing along a stringified number directly from a network response). In my opinion it's probably better for the long-term reliability of the application that you're alerted to this.

That said, these assertions are just a roundabout way of shoehorning in a kind of static typing. A more elegant way might be to use [TypeScript](https://www.typescriptlang.org/), which provides an additional syntax layer over JavaScript that includes native-like static types. With TypeScript, we could leave out the manual type assertions and let the TypeScript compiler handle type checking:

```typescript
// The argument is now of a TypeScript-specific 'number' type.
function func(arg: number) {}

const array = [1, 2, 3];

func(array[1]);
// No errors from TypeScript (the call is OK)

func(array[1 / 2]);
// No errors from TypeScript (but the index bug goes undetected)

func("1");
// TypeScript error: Argument of type 'string' is not assignable to parameter of type 'number'
```

I've used TypeScript for only a short while, but so far my impressions of it are mixed. Introducing static typing can increase the reliability and readability of your code, and TypeScript's interface system may get you to think about and define your in-code interfaces more clearly. On the other hand, its type system doesn't always feel solidly integrated with JavaScript, and from what I've seen it can add considerable complexity and/or compatibility issues to your build process.

### Easing into the messy package ecosystem

Among the biggest elephants in the room has to be JavaScript's sprawling package ecosystem.

As a C++ developer, I was used to working at low level, carefully managing the dependencies of my applications and often introducing no dependencies at all besides the standard library. By contrast, JavaScript is a high-level language where a typical project may depend on hundreds or thousands of external packages, and whose developer base seems more accepting of abstracting away the lower-level details behind those external dependencies.

When learning the GUI framework [React](https://reactjs.org/), it's typically recommended to use the [create-react-app](https://github.com/facebook/create-react-app) project bootstrapper. It sets up a React development environment in an automated way, saving you the trouble of manually doing the boilerplate. The downside is that it also makes your development environment dependent on over 1,500 extra packages.

When I started learning React (I was still fairly new to JavaScript overall), I thought the amount dependencies introduced by create-react-app was absolutely nuts. Why would a hello-world app need hundreds of miscellaneous dependencies? I chose to go against the grain of every React tutorial and learn the framework without packages or package managers:

```html
<div id="react-container"></div>
<script src="react.development.js"></script>
<script src="react-dom.development.js"></script>
<script>
    const container = document.getElementById("react-container");
    const component = React.createElement("span", {}, "Hello world");
    ReactDOM.render(component, container);
</script>
```

In this vein, there are no extra packages required, and I didn't even need to install a package manager. I could build full React apps this way, and they'd be lighter-weight and potentially more secure (or not). But I'd also be missing out on quality-of-life improvements like...

```javascript
// Creating a nested element with plain React.
divWithChild =
    React.createElement("div", {},
        React.createElement("span", {className: "label"}, "Hello world")
    );

// Creating the same element with JSX syntax (dependent on a transpiler).
divWithChild = (
    <div>
        <span className="label">
            Hello world
        </span>
    </div>
);
```

...not to mention I'd perpetually be working uphill and against the grain, because create-react-app and the like make things much easier, and most fellow JavaScript developers will be of that opinion.

I think easing into the apparent craziness of the JavaScript package ecosystem was the hardest part about transitioning from C++ to JavaScript. It meant no longer understanding certain aspects of my programs at lower levels. On the other hand, I wasn't supposed to be working at low level with a high-level language to begin with, and I could be more productive by accepting that.

## Future outlook

In my opinion, having first learnt C++ can give you an edge in becoming proficient in JavaScript. C++ offers a lot of freedom but requires a lot of responsibility; it motivates you to try to write well-behaving code because the alternative is a compromised application (or worse). If the motivation to build reliable code carries into JavaScript, it can elevate your work above some of the quirks of the language that might otherwise negatively affect the quality of your work - or at least you might get bitten by those quirks less often.

But experience of C++ can also be major baggage. You might be used to having lower-level control over your programs - for instance, insisting that dependencies be kept to a minimum. In the higher-level JavaScript this may become an anti-pattern that holds you back from being as effective of a developer as you could be.

I personally still use both languages. C++ gives me high performance, syntactic power, and few restrictions. JavaScript gives me first-class networking, a huge platform to distribute on, and a powerful GUI system in the triad of HTML5/CSS3/JavaScript; it can be an expressive, creative language and I often have fun using it.
