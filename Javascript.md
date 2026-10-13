*Q1. What are the data types in JavaScript?*

JavaScript data types define what kind of values a variable can hold and how those values behave in a program. They determine how data is stored in memory and how operations like comparison, calculation, and conversion work.

- Each data type has its own methods and operations that control how it can be used.
- Understanding data types helps prevent errors and makes code more efficient and reliable.

JavaScript Data Type Categories
JavaScript data types are categorized into Primitive and Non-Primitive types

![alt text](image-20.png)

Primitive Data Type
Primitive data types in JavaScript represent simple, immutable values stored directly in memory, ensuring efficiency in both memory usage and performance.

1. Number
The Number data type in JavaScript includes both integers and floating-point numbers. Special values like Infinity, -Infinity, and NaN represent infinite values and computational errors, respectively.

2. String
A String in JavaScript is a series of characters that are surrounded by quotes.

3. Boolean
The boolean type has only two values i.e. true and false.

4. Null
The special null value does not belong to any of the default data types. It forms a separate type of its own which contains only the null value.

5. Undefined
A variable that has been declared but not initialized with a value is automatically assigned the undefined value. It means the variable exists, but it has no value assigned to it.

6. Symbol (Introduced in ES6)
Symbols, introduced in ES6, are unique and immutable primitive values used as identifiers for object properties. They help create unique keys in objects, preventing conflicts with other properties.

7. BigInt (Introduced in ES2020)
BigInt is a built-in object that provides a way to represent whole numbers greater than 253. The largest number that JavaScript can reliably represent with the Number primitive is 253, which is represented by the MAX_SAFE_INTEGER constant.

Non-Primitive Data Types
The data types that are derived from primitive data types are known as non-primitive data types. It is also known as derived data types or reference data types.

1. Object
JavaScript objects are key-value pairs used to store data, created with {} or the new keyword. They are fundamental as nearly everything in JavaScript is an object.

2. Arrays
An Array is a special kind of object used to store an ordered collection of values, which can be of any data type.

3. Function
A function in JavaScript is a block of reusable code designed to perform a specific task when called.

4. Date Object
The Date object in JavaScript is used to work with dates and times, allowing for date creation, manipulation, and formatting.

5. Regular Expression
A RegExp (Regular Expression) in JavaScript is an object used to define search patterns for matching text in strings

Interesting Facts about Data Types

1. Dynamically Typed : JavaScript Variables are not bound to a specific data type. Mainly data type is stored with value (not with variable name) and is decided & checked at run time.

2. Everything is an Object (Sort of): In JavaScript, Functions are objects, arrays are objects, and even primitive values can behave like objects temporarily when you try to access properties on them.

3.  NaN is not equal to itself: NaN Stands for “Not-a-Number”, It is used to represent a computational error. NaN is technically of type number.

4. A Symbol is Never Equal to Another One : Symbol is a unique and immutable data type often used for creating private properties and methods. Symbols are never equal to any other Symbol.

5. Undefined and Null: undefined represents a variable that has been declared but not assigned, while null is an explicit assignment representing “no value”.

6. Integers are Floating are Numbers only. There is only one type number that covers both integers and floating point numbers.

7. A character is also a string. There is no separate type for characters. A single character is also a string.

*Q2. What is the difference between var, let, and const?*

1. Var

Before the advent of ES6, var declarations ruled. There are issues associated with variables declared with var, though. That is why it was necessary for new ways to declare variables to emerge. First, let's get to understand var more before we discuss those issues.

# Scope of var

Scope essentially means where these variables are available for use. var declarations are globally scoped or function/locally scoped. The scope is global when a var variable is declared outside a function. This means that any variable that is declared with var outside a function block is available for use in the whole window. var is function scoped when it is declared within a function. This means that it is available and can be accessed only within that function.

To understand further, look at the example below.

var greeter = "hey hi";

function newFunction() {
   var hello = "hello";
}

Here, greeter is globally scoped because it exists outside a function while hello is function scoped. So we cannot access the variable hello outside of a function.

# Hoisting of var

Hoisting is a JavaScript mechanism where variables and function declarations are moved to the top of their scope before code execution. So var variables are hoisted to the top of their scope and initialized with a value of undefined.

This means that if we do this:

console.log (greeter);
var greeter = "say hello"

it is interpreted as this:

var greeter;
console.log(greeter); // greeter is undefined
greeter = "say hello"

2. Let

let is now preferred for variable declaration. It's no surprise as it comes as an improvement to var declarations.

# let is block scoped

- A block is a chunk of code bounded by {}. A block lives in curly braces. Anything within curly braces is a block. So a variable declared in a block with let is only available for use within that block.

# let can be updated but not re-declared.

Just like var, a variable declared with let can be updated within its scope. Unlike var, a let variable cannot be re-declared within its scope. So while this will work:

let greeting = "say Hi";
greeting = "say Hello instead";

this will return an error:

let greeting = "say Hi";
let greeting = "say Hello instead"; // error: Identifier 'greeting' has already been declared

However, if the same variable is defined in different scopes, there will be no error:

let greeting = "say Hi";
if (true) {
    let greeting = "say Hello instead";
    console.log(greeting); // "say Hello instead"
}
console.log(greeting); // "say Hi"

Why is there no error? This is because both instances are treated as different variables since they have different scopes. This fact makes let a better choice than var. When using let, you don't have to bother if you have used a name for a variable before as a variable exists only within its scope. Also, since a variable cannot be declared more than once within a scope, then the problem discussed earlier that occurs with var does not happen.

# Hoisting of let

Just like var, let declarations are hoisted to the top. Unlike var which is initialized as undefined, the let keyword is not initialized. So if you try to use a let variable before declaration, you'll get a Reference Error.

3. Const

Variables declared with the const maintain constant values. const declarations share some similarities with let declarations.

# const declarations are block scoped

Like let declarations, const declarations can only be accessed within the block they were declared.

# const cannot be updated or re-declared

This means that the value of a variable declared with const remains the same within its scope. It cannot be updated or re-declared. So if we declare a variable with const, we can neither do this:

const greeting = "say Hi";
greeting = "say Hello instead"; // error: Assignment to constant variable.

nor this:

const greeting = "say Hi";
const greeting = "say Hello instead";// error: Identifier 'greeting' has already been declared

Every const declaration, therefore, must be initialized at the time of declaration. This behavior is somehow different when it comes to objects declared with const. While a const object cannot be updated, the properties of this objects can be updated. Therefore, if we declare a const object as this: 

const greeting = {
     message: "say Hi",
     times: 4
}

while we cannot do this:

greeting = {
    words: "Hello",
    number: "five"
} // error:  Assignment to constant variable.

we can do this:   greeting.message = "say Hello instead";

This will update the value of greeting.message without returning errors.

# Hoisting of const
Just like let, const declarations are hoisted to the top but are not initialized.

*Q3. What is the Execution Context?*

When the JavaScript engine scans a script file, it makes an environment called the Execution Context that handles the entire transformation and execution of the code.

During the context runtime, the parser parses the source code and allocates memory for the variables and functions. The source code is generated and gets executed.

There are two types of execution contexts: global and function. The global execution context is created when a JavaScript script first starts to run, and it represents the global scope in JavaScript. A function execution context is created whenever a function is called, representing the function's local scope.

Phases of the JavaScript Execution Context
There are two phases of JavaScript execution context:

1. Creation phase: In this phase, the JavaScript engine creates the execution context and sets up the script's environment. It determines the values of variables and functions and sets up the scope chain for the execution context.

2. Execution phase: In this phase, the JavaScript engine executes the code in the execution context. It processes any statements or expressions in the script and evaluates any function calls.

Everything in JS happens inside this execution context. It is divided into two components. One is memory and the other is code. It is
important to remember that these phases and components are applicable to both global and functional execution contexts.

*Q4. What is the Call Stack?*

To keep the track of all the contexts, including global and functional, the JavaScript engine uses a call stack. A call stack is also known as an 'Execution Context Stack', 'Runtime Stack', or 'Machine Stack'.

It uses the LIFO principle (Last-In-First-Out). When the engine first starts executing the script, it creates a global context and pushes it on the stack. Whenever a function is invoked, similarly, the JS engine creates a function stack context for the function and pushes it to the top of the call stack and starts executing it.

When execution of the current function is complete, then the JavaScript engine will automatically remove the context from the call stack and it goes back to its parent.

Let's see the following example:

function funcA(m,n) {
    return m * n;
}

function funcB(m,n) {
    return funcA(m,n);
}

function getResult(num1, num2) {
    return funcB(num1, num2)
}

var res = getResult(5,6);

console.log(res); // 30

In this example, the JS engine creates a global execution context that enters the creation phase.

First it allocates memory for funcA, funcB, the getResult function, and the res variable. Then it invokes getResult(), which will be pushed on the call stack.

Then getResult() will call funcB(). At this point, funcB's context will be stored on the top of the stack. Then it will start executing and call another function funcA(). Similarly, funcA's context will be pushed.

Once execution of each function is done, it will be removed from the call stack.

Call Stack

The call stack has its own fixed size depending on the system or browser. If the number of contexts exceeds the limit, then a stack overflow error will occur. This happens with a recursive function that has no base condition.

*Q5. What is the event loop in JavaScript runtimes?*

The event loop lets a JavaScript agent coordinate asynchronous operations without blocking its currently executing stack. Each agent runs one JavaScript job at a time; workers use separate agents and event loops.

Parts of the event loop
To understand it better, we need to understand all the parts of the system. These components are part of the event loop:

Call stack
The call stack keeps track of the functions being executed in a program. When a function is called, it is added to the top of the call stack. When the function completes, it is removed from the call stack. This allows the program to keep track of where it is in the execution of a function and return to the correct location when the function completes. As the name suggests, it is a stack data structure which follows last-in-first-out.

Web APIs/Node.js APIs
Hosts such as browsers and Node.js manage timers, networking, and file I/O outside the currently executing JavaScript stack. The implementation varies: an operation may use operating-system facilities, an evented subsystem, or a worker pool rather than one new thread per operation. When work becomes ready, the host schedules the relevant task or microtask.

Task queue / Macrotask queue / Callback queue
Task queues hold tasks that are ready to run. Browsers may maintain multiple task queues and choose among eligible queues according to the HTML event loop rules; "macrotask queue" is convenient informal terminology, not the specification's single queue.

Microtasks queue
The microtask queue holds promise reactions, queueMicrotask() callbacks, and other microtasks. At a microtask checkpoint, it drains until empty, including newly added microtasks; an unbounded stream can starve later tasks.

Event loop order

1. The host selects and runs one task. Synchronous function calls made by that task are pushed onto and popped from the call stack.

2. Host facilities handle timers, networking, and I/O outside the active JavaScript stack. When their results become ready, they arrange for tasks or promise reactions to be queued.

3. When the task finishes, the runtime performs a microtask checkpoint and drains the microtask queue, including microtasks queued by other microtasks.
4. In a browser, the host may then update rendering. It selects another eligible task according to host-defined scheduling rules and repeats the cycle.

This model is deliberately simplified: browsers can have multiple task queues, and Node.js divides work into event-loop phases. The key ordering rule is that each completed task is followed by a microtask checkpoint before another task runs.

Example

The example below mixes synchronous logs with two timer callbacks and two promise callbacks. The first timer's callback enqueues a microtask, and the first promise callback enqueues another timer — small additions that exercise every ordering rule the event loop applies, while keeping each line individually trivial to read.

console.log('Start');

setTimeout(() => {
  console.log('Timeout 1');
  Promise.resolve().then(() => console.log('Promise 2'));
}, 0);

Promise.resolve().then(() => {
  console.log('Promise 1');
  setTimeout(() => console.log('Timeout 3'), 0);
});

setTimeout(() => console.log('Timeout 2'), 0);

console.log('End');

// Console output:
// Start
// End
// Promise 1
// Timeout 1
// Promise 2
// Timeout 2
// Timeout 3

Three rules the trace makes explicit:

- Microtasks drain before any macrotask. Step 6 runs Promise 1 before either timer, even though both timers were scheduled before the promise callback ran.

- A macrotask that schedules a microtask interleaves. Step 7 runs Timeout 1 and enqueues Promise 2; step 8 runs Promise 2 before the next macrotask, not after. The event loop re-checks the microtask queue between every macrotask, which is why a single drain at the end of synchronous code is not enough to model behavior correctly.

- A microtask that schedules a macrotask appends to the queue. Step 6 runs Promise 1 and schedules Timeout 3; Timeout 3 then runs last, after both timers that were already in the macrotask queue. Microtasks cannot promote a macrotask to the front of the line.

*Q6. Explain event delegation in JavaScript?*

Event delegation is a design pattern in JavaScript used to efficiently manage and handle events on multiple child elements by attaching a single event listener to a common ancestor element. This pattern is particularly valuable in scenarios where you have a large number of similar elements, such as list items, and want to optimize event handling.

How event delegation works

1. Attach a listener to a common ancestor: Instead of attaching individual event listeners to each child element, you attach a single event listener to a common ancestor element higher in the DOM hierarchy.

2. Event bubbling: When an event occurs on a child element, it bubbles up through the DOM tree to the common ancestor element. During this propagation, the event listener on the common ancestor can intercept and handle the event.

3. Determine the target: Within the event listener, you can inspect the event object to identify the actual target of the event (the child element that triggered the event). Use event.target to find which specific child element was interacted with. (event.currentTarget refers to the ancestor the listener is attached to, not the child that triggered the event.)

4. Perform action based on target: Based on the target element, you can perform the desired action or execute code specific to that element. This allows you to handle events for multiple child elements with a single event listener.

Benefits of event delegation

- Fewer listeners: Event delegation reduces listener bookkeeping and can reduce memory use when dealing with a very large number of elements.
- Dynamic elements: It works with dynamically added or removed child elements, as the common ancestor continues to listen for events on them.

Example

// HTML:
// <ul id="item-list">
//   <li>Item 1</li>
//   <li>Item 2</li>
//   <li>Item 3</li>
// </ul>

const itemList = document.getElementById('item-list');

itemList.addEventListener('click', (event) => {
  if (event.target.tagName === 'LI') {
    console.log(`Clicked on ${event.target.textContent}`);
  }
});

In this example, a single click event listener is attached to the <ul> element. When a click event occurs on an <li> element, the event bubbles up to the <ul> element, where the event listener checks the target's tag name to identify whether a list item was clicked. It's crucial to check the identity of the event.target as there can be other kinds of elements in the DOM tree.

*Q7. Explain how `this` works in JavaScript?*

There's no simple explanation for this; it is one of the most confusing concepts in JavaScript because its behavior differs from many other programming languages. The one-liner explanation of the this keyword is that it is a dynamic reference to the context in which a function is executed.

A longer explanation is that this follows these rules:

1. If the new keyword is used when calling the function, meaning the function was used as a function constructor, the this inside the function is the newly-created object instance.

2. If this is used in a class constructor, the this inside the constructor is the newly-created object instance.

3. If apply(), call(), or bind() is used to call/create a function, this inside the function is the object that is passed in as the argument.

4. If a function is called as a method (e.g. obj.method()) — this is the object that the function is a property of.

5. If a function is invoked as a free function invocation, meaning it was invoked without any of the conditions present above, this is the global object. In the browser, the global object is the window object. If in strict mode ('use strict';), this will be undefined instead of the global object.

6. If multiple of the above rules apply, the rule that is higher wins and will set the this value.

7. If the function is an ES2015 arrow function, it ignores all the rules above and receives the this value of its surrounding scope at the time it is created.

*Q8. Describe the difference between a cookie, `sessionStorage` and `localStorage` in browsers*

Cookies, localStorage, and sessionStorage are browser storage mechanisms. Client storage is useful for state such as themes, personalized layouts, draft form data, and identifiers needed by a server session. Sensitive authentication credentials require a threat-model-specific design; they should not be placed in Web Storage by default.

These client-side storage mechanisms have the following common properties:

- Client-side JavaScript can read and modify the values, except for HttpOnly cookies.
- Key-value based storage.
- They are only able to store values as strings. Non-strings will have to be serialized into a string (e.g. JSON.stringify()) in order to be stored.

Use cases for each storage mechanism

Since cookies have a relatively low maximum size, it is not advisable to store all your client-side data within cookies. The distinguishing properties about cookies are that cookies are sent to the server on every HTTP request so the low maximum size is a feature that prevents your HTTP requests from being too large due to cookies. Automatic expiry of cookies is a useful feature as well.

With that in mind, cookies suit small values that the server needs, such as opaque session identifiers, analytics identifiers, consent choices, or language preferences used during server rendering. Sensitive cookies can benefit from HttpOnly, Secure, and SameSite; Expires or Max-Age controls persistence. The server must still validate and authorize every request.

localStorage and sessionStorage both implement the Web Storage API interface. Their quota is browser-dependent and can be exceeded, so applications should handle QuotaExceededError. Values stored in Web Storage are not automatically sent with HTTP requests.

While you can manually include values from Web Storage when making AJAX/fetch() requests, the browser does not include them in the initial request / first load of the page. Hence Web Storage should not be used to store data that is relied on by the server for the initial rendering of the page if server-side rendering is being used (typically authentication/authorization-related information). localStorage is most suitable for user preferences data that do not expire, like themes and layouts (if it is not important for the server to render the final layout). sessionStorage is most suitable for temporary data that only needs to be accessible within the current browsing session, such as form data (useful to preserve data during accidental reloads).

Cookies

Cookies are used to store small pieces of data on the client side that can be sent back to the server with every HTTP request.

- Storage capacity: Limited to around 4 KB per cookie. Browsers also limit how many cookies can be stored per domain.

- Lifespan: Cookies can have a specific expiration date set using the Expires or Max-Age attributes. Without one, they are session cookies, although browsers may restore session cookies as part of session restore.

- Access: Cookies are domain-specific and can be shared across different pages and subdomains within the same domain.

- Security: Cookies can be marked as HttpOnly to prevent access from JavaScript, reducing the risk of XSS attacks. They can also be secured with the Secure flag to ensure they are sent only when HTTPS is used.

localStorage

localStorage is used for storing data that persists even after the browser is closed and reopened. It is designed for long-term storage of data.

- Storage capacity: Typically around 5MB per origin (varies by browser).

- Lifespan: Data in localStorage persists until explicitly deleted by the user or the application.

- Access: Data is accessible within all tabs and windows of the same origin.

- Security: All JavaScript on the page has access to values within localStorage.

sessionStorage

sessionStorage is used to store data for the duration of the page session. It is designed for temporary storage of data.

- Storage capacity: Typically around 5MB per origin (varies by browser).

- Lifespan: Data in sessionStorage is cleared when the page session ends (i.e., when the browser or tab is closed). Reloading the page does not destroy data within sessionStorage.

- Access: Data is accessible only within the current tab (or browsing context). Different tabs share different sessionStorage objects even if they belong to the same browser window. In this context, window refers to a browser window that can contain multiple tabs.

- Security: All JavaScript on the same page has access to values within sessionStorage for that page.

Beyond these three: IndexedDB and Cache Storage

Modern apps frequently need more than what these three APIs offer. Two more are worth knowing:

- IndexedDB: an in-browser, asynchronous, transactional database. Use it for large structured data (offline app state, large user-generated content, search indexes), MBs to GBs of storage, and queryable data. Wrappers like Dexie.js and idb make the API more pleasant.

- Cache Storage (caches): paired with Service Workers, this stores HTTP request/response pairs for offline-capable apps and PWAs. It is not a general-purpose key-value store; it is specifically for caching network responses.

- localStorage is for simple key-value config only. If you find yourself JSON-stringifying complex nested data into localStorage, IndexedDB is usually a better fit.

*Q9. What's the difference between a JavaScript variable that is: `null`, `undefined` or undeclared?*

# Undeclared

An undeclared identifier has no binding in the visible scope chain. Reading it directly throws ReferenceError. In sloppy-mode scripts, assigning to an unresolvable identifier can create a property on the global object; strict mode throws instead. Avoid relying on either behavior. typeof identifier is the one operation that returns 'undefined' for an undeclared identifier, though that same result cannot distinguish it from a declared value containing undefined.

function foo() {
  x = 1; // Throws a ReferenceError in strict mode
}

foo();
console.log(x); // 1 (if not in strict mode)

Using the typeof operator on undeclared variables will give 'undefined'.

console.log(typeof y === 'undefined'); // true

# undefined

A variable that is undefined is a variable that has been declared, but not assigned a value. It is of type undefined. If a function does not return a value, and its result is assigned to a variable, that variable will also have the value undefined. To check for it, compare using the strict equality (===) operator or typeof which will give the 'undefined' string. Note that you should not be using the loose equality operator (==) to check, as it will also return true if the value is null.

let foo;
console.log(foo); // undefined
console.log(foo === undefined); // true
console.log(typeof foo === 'undefined'); // true

console.log(foo == null); // true. Wrong, don't use this to check if a value is undefined!

function bar() {} // Returns undefined if there is nothing returned.
let baz = bar();
console.log(baz); // undefined

# null

A variable that is null will have been explicitly assigned to the null value. It represents no value and is different from undefined in the sense that it has been explicitly assigned. To check for null, simply compare using the strict equality operator. Note that like the above, you should not be using the loose equality operator (==) to check, as it will also return true if the value is undefined.

const foo = null;
console.log(foo === null); // true
console.log(typeof foo === 'object'); // true

console.log(foo == undefined); // true. Wrong, don't use this to check if a value is null!

*Q10. What's the difference between .call and .apply in JavaScript?*

Both .call and .apply are used to invoke functions, and the first parameter will be used as the value of this within the function. However, .call takes in comma-separated arguments as the next arguments, while .apply takes in an array of arguments as the next argument.

An easy way to remember this is C for call and comma-separated and A for apply and an array of arguments.

function add(a, b) {
  return a + b;
}

console.log(add.call(null, 1, 2)); // 3
console.log(add.apply(null, [1, 2])); // 3

With ES6 syntax, we can invoke call using an array along with the spread operator for the arguments.

function add(a, b) {
  return a + b;
}

console.log(add.call(null, ...[1, 2])); // 3

# Use cases

Context management

.call and .apply can set the this context explicitly when invoking methods on different objects.

const person = {
  name: 'John',
  greet() {
    console.log(`Hello, my name is ${this.name}`);
  },
};

const anotherPerson = { name: 'Alice' };

person.greet.call(anotherPerson); // Hello, my name is Alice
person.greet.apply(anotherPerson); // Hello, my name is Alice

Function borrowing

Both .call and .apply allow borrowing methods from one object and using them in the context of another. This is useful when passing functions as arguments (callbacks) and the original this context is lost. .call and .apply allow the function to be invoked with the intended this value.

function greet() {
  console.log(`Hello, my name is ${this.name}`);
}

const person1 = { name: 'John' };
const person2 = { name: 'Alice' };

greet.call(person1); // Hello, my name is John
greet.apply(person2); // Hello, my name is Alice

# Alternative syntax to call methods on objects

.apply can be used with object methods by passing the object as the first argument followed by the usual parameters.

const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

Array.prototype.push.apply(arr1, arr2); // Same as arr1.push(4, 5, 6)

console.log(arr1); // [1, 2, 3, 4, 5, 6]

Deconstructing the above:

1. The first object, arr1 will be used as the this value.

2. .push() is called on arr1 using arr2 as an array of arguments because it's using .apply().

3. Array.prototype.push.apply(arr1, arr2) is equivalent to arr1.push(...arr2).

It may not be obvious, but Array.prototype.push.apply(arr1, arr2) mutates arr1. It's clearer to call methods using the OOP-centric way instead where possible.

