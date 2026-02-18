*Q1. What is a Closure ?*

Ans: Closure is define as a function that remembers variables from its parent scope, even after the parent function has finished executing.

For Example : Here inner Function know or remember the varibale count even after executing.
function outer() {
  let count = 0;

  return function inner() {
    count++;
    return count;
  };
}

const counter = outer();
counter(); // 1
counter(); // 2

Advantages of Closures :
1) Data Privacy (Encapsulation) 

function authManager() {
  let token = "";

  return {
    setToken(value) {
      token = value;
    },
    getToken() {
      return token;
    }
  };
}

const auth = authManager();
auth.setToken("abc123");
console.log(auth.getToken()); 

In this function of Auth Token Storage 
here token variable can't directly access it can be set via method or fuction that give real protection 

2) Avoid Global Variables : 
  No pollution of global scope
  Safer large applications

3) Reusable Configured Functions:
  Write once, reuse many times
  Cleaner code

function createApi(baseUrl) {
  return function(endpoint) {
    return fetch(baseUrl + endpoint);
  };
}

const userApi = createApi("/api/users");
userApi("/list");

4) State Management in Libraries: Many libraries internally use closures to keep state without exposing it.

5) Used Heavily in Event Handlers: 
 An event handler is a function that runs when something happens:
  Button click
  Input change
  Mouse hover
  Key press

button.addEventListener("click", () => {
  console.log("Clicked");
});

Now clouser come in secene 
function setupButton() {
  let count = 0;

  document
    .getElementById("btn")
    .addEventListener("click", function () {
      count++;
      console.log(count);
    });
}
setupButton();
Question

After setupButton() finishes, shouldn't count disappear?
Normally yes. But it does not disappear.
Why?
Because the click handler closes over count.That is a closure.

setupButton() executed
   ↓
count created
   ↓
event handler remembers count
   ↓
setupButton() finishes
   ↓
count still alive because handler needs it

In Simple : Event handlers use closures because they need to remember variables from the component or function where they were created.

Real-Life Analogy ->
Think of a backpack
You give a worker (event handler) a backpack (closure) containing variables.
Even after you leave, the worker still has the backpack.

Disadvantages: 
1. Higher Memory Usage
2. Harder to Debug	
3. Performance Slightly Slower
   Accessing outer scope variable is slower than local variable.
   In normal apps → negligible
   In high-frequency loops → noticeable