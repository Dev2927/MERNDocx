Node.js

Q1. What is Node.js and how does it work?

Node.js is an open-source, cross-platform JavaScript Runtime Environment. Node.js is a software program that can execute JavaScript code. Put more properly, Node.js is a JavaScript runtime environment. It is an environment developed to make it possible to use JavaScript code for server-side scripting.

How Does Node.js Work

Node.js was written mostly with C/C++. As a program that is supposed to run web servers, Node.js needs to constantly interact with a device's operating system. Building Node.js with a low-level language like C made it easy for the software to access the operating system’s resources and use them to execute instructions. There are three main components we must understand to see how Node.js works. These components are:

1. V8 Engine: 

The V8 Engine is the JavaScript engine that interprets and runs JavaScript code in the Chrome browser. Some other browsers use a different engine, for example, Firefox uses SpiderMonkey, and Safari uses JavaScriptCore. Without the JavaScript engine, a computer can not understand JavaScript. The V8 engine contains a memory heap and call stack. They are the building blocks for the V8 engine. They help manage the execution of JavaScript code. The memory heap is the data store of the V8 engine. Whenever we create a variable that holds an object or function in JavaScript, the engine saves that value in the memory heap. To keep things simple, it is similar to a backpack that stores supplies for a hiker.
Whenever the engine is executing code and comes across any of those variables, it looks up the actual value from the memory heap – just like whenever a hiker is feeling cold and wants to start a fire, they can look into their backpack for a lighter.

The call stack is another building block in the V8 engine. It is a data structure that manages the order of functions to be executed. Whenever the program invokes a function, the function is placed on the call stack and can only leave the stack when the engine has handled that function. JavaScript is a single-threaded language, which means that it can only execute one instruction at a time. Since the call stack contains the order of instructions to be executed, it means that the JavaScript engine has just one order, one call stack.

![alt text](image.png)

2. Libuv:

Libuv is a C library used for performing Input/output (I/O) operations. I/O operations have to do with sending requests to the computer and receiving responses. These operations include reading and writing files, making network requests, and so on. This means that Libuv is cross-platform (can run on any operating system) and has a focus on Asynchronous I/O. The computer tends to take time to process I/O instructions, It can handle more than one I/O operation at once. This is what makes Node.js process I/O instructions efficiently despite being single-threaded. Whenever we pass a script to Node.js, the engine parses the code and starts processing it. The call stack holds the invoked functions and keeps track of the program. If the V8 engine comes across an I/O operation, it passes that operation over to Libuv. Libuv then executes the I/O operation. There are bindings that connect JavaScript functions to their actual implementation in Libuv. These bindings make it possible to use JavaScript code for I/O instructions. Node.js uses Libuv for the actual implementation but exposes Application Programming Interfaces (APIs). So, we can now use a Node.js API (which looks like a JavaScript function) to initiate an I/O operation. One interesting thing to note is that it is true that JavaScript is a single-threaded language, but Libuv—the low-level library Node.js uses— can make use of a thread pool (multiple threads) when executing instructions in the operating system. 

![alt text](image-1.png)

3. Event Loop:

The Event Loop in Node.js is a very important part of the process. From the name, we can see it is a loop. The loop starts running as Node.js begins executing a program. When we run our JavaScript program that contains some asynchronous code (like I/O instructions or timer-based actions), Node.js handles them using the Node.js APIs.
