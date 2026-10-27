*Q1. What is React? Describe the benefits of React*

React is an open-source JavaScript library maintained by Meta and the community for building user interfaces. It can power small interactive widgets, single-page applications, native applications through React Native, and server-rendered applications through a framework. React lets developers compose components that derive their output from props, state, and context.

Modern React is function-component- and Hooks-oriented. JSX, an HTML-like syntax extension to JavaScript, is the standard way to describe UI. React 19 added stable Server Components, the use API for reading promises and context, and Actions for form and data mutations. React Compiler 1.0 is a separate, optional build-time optimizer that can memoize compatible components and values when enabled.

Benefits of React
React's benefits come from its component model, declarative rendering, renderer ecosystem, and scheduling capabilities.

1. Component-based architecture

React encourages breaking a UI into reusable components. A component can encapsulate rendering, local state, and related behavior, which supports:

- Modularity and reuse: A component with an explicit prop contract can be composed in multiple features or applications.
- Local reasoning: Keeping related rendering and behavior together narrows the code involved in many changes.
- Focused tests: Components can be exercised through their visible output and interactions at a defined boundary.

2. Declarative rendering and reconciliation

Components return React elements that describe the desired UI. React reconciles a new description with the previous one and lets the renderer commit the required host changes. This gives applications a predictable declarative update model and avoids replacing unchanged DOM nodes. It has its own rendering and diffing costs, so it is not a guarantee that React is faster than carefully written imperative DOM code.

3. Large and active community

React has a mature ecosystem, which affects practical framework and library choices:

- Maintained documentation and learning material: The official documentation covers both fundamentals and current APIs.
- Third-party libraries and tools: Established options exist for routing, data fetching, testing, forms, and accessible UI primitives.
- A large hiring and support base: Many teams already have React experience, examples, and production knowledge to draw from.

*Q2. What is the difference between React Node, React Element, and a React Component?*

1. React node

A React Node is anything that can appear in a JSX child position. The set includes:

- React Elements (e.g. <div />, <MyComponent />)
- Strings, numbers, and bigints (rendered as text)
- Arrays and other iterables of React Nodes (arrays are commonly produced by .map(...))
- Fragments (<>...</> or <React.Fragment>)
- Portals (created with createPortal)
- Promises of React Nodes in rendering environments that support suspending on them
- null, undefined, false, and true — these are valid nodes that render nothing (they are skipped, not converted to text)

2. React element

A React Element is an immutable, plain JavaScript object that describes what should appear on screen. It includes a type (a string for host elements like 'div', or a component reference), props (including children), and a key. JSX is sugar for React.createElement (or, with the modern JSX transform, react/jsx-runtime's jsx function), which produces these objects.

const element = <div className="greeting">Hello, world!</div>;

// Roughly equivalent to:
const sameElement = React.createElement(
  'div',
  { className: 'greeting' },
  'Hello, world!',
);

React elements should be treated as opaque and immutable rather than inspected or modified directly. They are lightweight, short-lived descriptions that React reconciles with the previous output before the renderer commits host changes.

3. React component

A React Component is a function or class that React can use as an element type. It accepts props and returns React Nodes that describe the UI; an async Server Component may return a promise of a node. Component names are conventionally PascalCase so JSX can distinguish them from host elements (<button> is a DOM element; <Button> is a component).

- Function components are the standard form. They are plain functions of props that return a node:

function Welcome({ name }) {
  return <h1>Hello, {name}</h1>;
}

Hooks (useState, useEffect, useMemo, etc.) cover what used to require a class.

- Class components remain supported, but React recommends function components for new code. Legacy lifecycles such as componentWillMount, componentWillReceiveProps, and componentWillUpdate are deprecated, and Hooks are available only in functions.

class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

The mental model: a component is a recipe, an element is a single dish made from that recipe, and a node is anything React is willing to plate up — including "nothing."

*Q3. What is JSX and how does it work?*

JSX is syntax that a build tool transforms into calls producing React element descriptions; browsers do not execute JSX directly.

What is JSX?

JSX is a syntax extension for JavaScript that lets you describe UI trees with an HTML-like syntax. Although it was popularized by React, JSX itself is a separate spec and is also used by other libraries such as Preact and Solid. TypeScript supports it natively in .tsx files.

How does JSX work?

JSX is not valid JavaScript by itself. A compiler — typically Babel or the bundler's built-in transform (SWC, esbuild, Oxc) — converts JSX into ordinary JavaScript function calls before the browser sees it.

JSX syntax
JSX allows you to write HTML-like tags directly in your JavaScript code. For example:

const element = <h1>Hello, world!</h1>;

Transformation process

Since React 17 (2020), the default transform is the automatic JSX runtime. Instead of compiling to React.createElement, the compiler imports jsx / jsxs helpers from react/jsx-runtime and emits calls to those. A consequence is that you no longer need to import React from 'react' just to use JSX:

// Source
const element = <h1>Hello, world!</h1>;

// Output with the automatic runtime (conceptually)
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { children: 'Hello, world!' });

The older "classic" transform compiled the same JSX to React.createElement('h1', null, 'Hello, world!') and required React to be in scope. The classic form is still useful as a mental model for what JSX desugars to, but new projects should use the automatic runtime.

*Q4. What is the difference between state and props in React?*

State is component-owned memory, while props are read-only inputs supplied by the component's parent.

State

State is data a component owns and can change over time, usually in response to user interaction, network responses, or timers. When state changes, React schedules a re-render of that component so the UI reflects the new value.

- State is local: a parent cannot read a child's state directly.
- In function components, state is declared with the useState hook (or useReducer for more complex transitions).
- Setters queue state for a subsequent render and React batches updates where possible. The current handler still sees the state snapshot from the render that created it.
- The term updater function specifically refers to the setX(prev => next) form passed to a setter — not the setter itself. Use it whenever the next value depends on the previous one, so batched updates compose correctly.

import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // This works for one click, but repeated queued calls would reuse the same
  // render snapshot: const increment = () => setCount(count + 1);

  // Right: the updater function gets the latest value
  const increment = () => setCount((prev) => prev + 1);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
      <button
        onClick={() => {
          // Both updates apply; final count goes up by 2
          increment();
          increment();
        }}>
        Increment twice
      </button>
    </div>
  );
}

Props

Props (short for "properties") are the inputs a parent passes to a child. From the child's perspective they are read-only — you must not assign to them. From the system's perspective they are not "immutable" in any deep sense; the parent simply re-renders with a new value, and the child receives the new props on its next render.

function Parent() {
  const [name, setName] = useState('World');
  return (
    <>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <Greeting message={`Hello, ${name}!`} />
    </>
  );
}

function Greeting({ message }) {
  // Read-only here. Mutating `message` would be a bug.
  return <p>{message}</p>;
}

Props can carry data, JSX (children), and callbacks. Callback props are how children communicate upward — the child invokes the function, the parent updates its state, and new props flow down on the next render.

*Q5. What is the purpose of the `key` prop in React?*

Keys give sibling elements stable identity so reconciliation can preserve or reset the correct component state and DOM.

# Introduction

The key prop is a special attribute you need to include when creating lists of elements in React. It is crucial for helping React identify which items have changed, been added, or removed, thereby optimizing the rendering process.

# Why key is important

Stable identity affects correctness before it affects rendering efficiency:

1. Correct identity during reconciliation: When React diffs a list, the key is how it decides which previous element corresponds to which new one. Without keys (or with bad keys) React can still render the list, but it will associate the wrong component instance with the wrong data — which means the wrong internal state, refs, and DOM nodes get reused. The bigger risk is incorrect state association, not raw DOM operation count.

2. Efficient reordering: With stable keys, React can move existing DOM nodes instead of unmounting and remounting them when items are reordered.

3. Explicit state reset: Changing a component's key deliberately is the idiomatic way to force React to unmount the old instance and mount a brand-new one (see Resetting state with a key below).

How to use the key prop

When rendering a list of elements, you should provide a unique key for each element. This key should be stable, meaning it should not change between renders. Typically, you can use a unique identifier from your data, such as an id.

const items = [
  { id: 1, value: 'Item 1' },
  { id: 2, value: 'Item 2' },
  { id: 3, value: 'Item 3' },
];

function ItemList() {
  return (
    <ul>
      {items.map((item) => (
        <ListItem key={item.id} value={item.value} />
      ))}
    </ul>
  );
}

function ListItem({ value }) {
  return <li>{value}</li>;
}

# Rules for keys

A useful key must remain tied to the same logical sibling across renders:

- Unique among siblings, not globally: Keys only need to be unique within a single map() / array. Two unrelated lists can happily both contain a key="1".
- Stable across renders: The same piece of data should get the same key every render. Avoid Math.random() or Date.now().
- Keys are not passed to the child: key is consumed by React itself. If your child component also needs the id, pass it as a separate prop.

*Q6. What is the difference between controlled and uncontrolled React components?*

# Controlled components

A controlled input passes both value (or checked) and onChange to the element. React state holds the truth; every keystroke flows through a setter.

import { useState } from 'react';

function ControlledForm() {
  const [name, setName] = useState('');

  function handleSubmit(event) {
    event.preventDefault();
    alert('A name was submitted: ' + name);
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name:
        <input
          type="text"
          value={name}
          onChange={(event) => setName(event.target.value)}
        />
      </label>
      <input type="submit" value="Submit" />
    </form>
  );
}

# Uncontrolled components

An uncontrolled input keeps its value in the DOM. Seed the initial value with defaultValue (or defaultChecked for checkboxes/radios), and read the current value through a ref when you need it.

import { useRef } from 'react';

function UncontrolledForm() {
  const inputRef = useRef(null);

  function handleSubmit(event) {
    event.preventDefault();
    alert('A name was submitted: ' + inputRef.current.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name:
        <input type="text" defaultValue="" ref={inputRef} />
      </label>
      <input type="submit" value="Submit" />
    </form>
  );
}

defaultValue is only consulted on the initial render — changing it later does not update the DOM. Passing value without an onChange handler makes the input read-only and warns in development unless readOnly is intentional. The reverse is valid: an input with onChange but no value remains uncontrolled. Pick one mode per field and do not switch modes during the input's lifetime.

# <input type="file"> is always uncontrolled

File inputs cannot be controlled — their value is read-only for security reasons (a page must not be able to set the user's chosen file). Always read files via a ref or from the change/submit event, even in an otherwise controlled form.

function FileForm() {
  const fileRef = useRef(null);

  function handleSubmit(event) {
    event.preventDefault();
    const file = fileRef.current.files[0];
    // upload file...
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" ref={fileRef} />
      <button type="submit">Upload</button>
    </form>
  );
}

# Key differences

The ownership choice changes how values update and which use cases each style fits.

State management

Controlled and uncontrolled inputs store their current value in different places:

- Controlled: React state owns the value; the DOM mirrors it.
- Uncontrolled: The DOM owns the value; React reads it on demand.

*Q7. What are some pitfalls about using context in React?*

Context is convenient for distribution, but its broadcast update model and implicit dependencies can become costly when a value changes frequently or grows too broad.

# Unnecessary re-renders from an unstable provider value

When a context value changes by reference, every component that reads that context re-renders — even if it only uses a field that hasn't actually changed. The most common cause is constructing a new object inline as the provider's value, which makes a fresh reference on every parent render:

// Pitfall — `value` is a new object every render, so every consumer re-renders

function ParentComponent() {

  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  return (
    <MyContext.Provider value={{ user, setUser, theme, setTheme }}>
      <ChildComponent />
    </MyContext.Provider>
  );
}

Fix it by memoizing the value. React Compiler may generate equivalent memoization when enabled, but Context still propagates every genuine value change:

function ParentComponent() {
  
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  const value = useMemo(
    () => ({ user, setUser, theme, setTheme }),
    [user, theme],
  );

  return (
    <MyContext.Provider value={value}>
      <ChildComponent />
    </MyContext.Provider>
  );
}

Note: only consumers of the context re-render when the value changes — not "all components in the subtree." Components that don't call useContext/use(MyContext) are unaffected.

# React.memo doesn't stop context-driven re-renders

A common surprise: wrapping a consumer in React.memo does not prevent re-renders triggered by a context value change. memo only skips re-renders caused by changing props. If the component reads a context whose value changed, it re-renders regardless. The fix is to make the context value stable (above) or split the context.

# Putting too much unrelated state in one context

If you cram an entire app's state into a single context, every change to any slice re-renders every consumer. Split it into smaller, focused providers — for example, separate AuthContext, ThemeContext, and CartContext — so that a cart update doesn't re-render every theme consumer.You can also split read and write APIs into separate contexts so components that only need to dispatch don't re-render when state changes.

*Q8. What are the benefits of using hooks in React?*

Hooks let function components compose stateful behavior while keeping related setup, updates, and cleanup together.

# Why hooks were introduced

Before hooks (React 16.8, Feb 2019), the main ways to share stateful logic between components were higher-order components (HOCs) and render props. Repeated use of either pattern could produce deeply nested wrapper trees and obscure data flow in React DevTools. Class components also required this handling and often split one synchronization concern across componentDidMount, componentDidUpdate, and componentWillUnmount.

Hooks let components compose stateful logic through functions without adding wrapper components or relying on class lifecycle methods.

# Reusable logic via custom hooks

Custom hooks let you extract a related combination of state, Effects, and other hooks into a function that components can call. Unlike an HOC, each custom Hook adds no wrapper component to the rendered tree.

import { useSyncExternalStore } from 'react';

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function getServerSnapshot() {
  return true;
}

function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

function StatusBar() {
  const isOnline = useOnlineStatus();
  return <div>{isOnline ? 'Online' : 'Offline'}</div>;
}

This version uses useSyncExternalStore because online status is a browser value that changes outside React. The server snapshot also avoids reading navigator during server rendering and gives hydration a deterministic initial value.

# Simplified state management

useState adds local state to any function component without converting it to a class:

import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount((value) => value + 1)}>{count}</button>
  );
}

# Side effects without lifecycle methods

useEffect replaces componentDidMount, componentDidUpdate, and componentWillUnmount with a single API where setup and cleanup live next to each other instead of being scattered across three methods:

import { useEffect } from 'react';
import { createConnection } from './chat-api';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>Room: {roomId}</h1>;
}

# No more this

Function components and hooks have no this, so there's nothing to bind, no .bind(this) in constructors, and no surprises about what this refers to in a callback.

*Q9. What are the rules of React hooks?*

The Rules of Hooks preserve a consistent call order so React can associate each Hook call with the correct component state across renders.

# Always call hooks at the top level

Hooks must be called in the same order on every render. That means you cannot call them inside loops, conditions, nested functions, or after an early return. React identifies which useState/useEffect/etc. call corresponds to which piece of state purely by call order — break the order and React's internal bookkeeping desyncs.

import { useState } from 'react';

function Counter({ enabled }) {
  const [count, setCount] = useState(0);

  if (!enabled) return null;

  return (
    <button onClick={() => setCount((value) => value + 1)}>{count}</button>
  );
}

// Incorrect — hook inside an `if`
function ConditionalCounter({ enabled }) {
  if (enabled) {
    const [count, setCount] = useState(0); // hook order changes between renders
    return <div>{count}</div>;
  }
  return null;
}

// Incorrect — hook after an early return
function SelectableList({ items }) {
  if (items.length === 0) return null;
  const [selected, setSelected] = useState(null); // skipped on the early-return path
  return <List items={items} selected={selected} onSelect={setSelected} />;
}

To fix the early-return case, move the hook above the conditional:

import { useState } from 'react';

function SelectableList({ items }) {
  const [selected, setSelected] = useState(null);
  if (items.length === 0) return null;
  return <List items={items} selected={selected} onSelect={setSelected} />;
}

# The special case: use

Despite its name, React's use(resource) API is not a Hook. You may call it inside conditions and loops because it does not store state by call order:

import { use } from 'react';

function Comments({ shouldLoad, commentsPromise }) {
  if (shouldLoad) {
    const comments = use(commentsPromise);
    return <CommentList comments={comments} />;
  }
  return null;
}

use still must be called from a component or custom Hook. It also cannot be wrapped in try/catch; use an error boundary to handle a rejected promise.

# Only call hooks from React functions

Hooks can only be called from:

1. React function components.

2. Other custom hooks (which by convention must have a name starting with use).

Calling a hook from a regular utility function, a class component, or an event handler is not allowed.

import { useState } from 'react';

// Correct — function component
function MyComponent() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}

// Correct — custom hook (name starts with `use`)
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount((c) => c + 1);
  return { count, increment };
}

// Incorrect — plain function, not a component or hook
function regularFunction() {
  const [count, setCount] = useState(0); // violates the rules of hooks
}

The use prefix isn't cosmetic — it identifies a function as a custom Hook and lets the linter enforce call-order rules at its call sites. If a function that calls Hooks is instead named getCounter, the linter reports that Hooks are being called from a function that is neither a component nor a custom Hook.

*Q10. What is the difference between `useEffect` and `useLayoutEffect` in React?*

useEffect schedules synchronization work after React commits changes to the DOM. If the effect was not caused by an interaction, React generally lets the browser paint first. For interaction-driven effects, React may run it before paint so the event system can observe the result. If timing relative to paint is essential, use the appropriate browser scheduling API or useLayoutEffect rather than relying on useEffect always running afterward.

- It is the right default for data fetching, subscriptions, event listeners, and logging.

- The dependency array controls when React re-synchronizes: [a, b] means after the initial commit and after commits where a or b changed by Object.is; [] means after mounting; omitting the array means after every commit.

- In development Strict Mode, React runs one extra setup-and-cleanup cycle before the real setup. This surfaces missing cleanup logic; it is not a production behavior.

import { useEffect } from 'react';

function Example() {
  useEffect(() => {
    console.log('Mounted');
    return () => console.log('Cleanup on unmount');
  }, []); // [] deps: cleanup runs on unmount (plus the Strict Mode stress test in development)

  return <div>Hello, World!</div>;
}

In production, cleanup for an Effect with [] dependencies runs when the component unmounts. In development Strict Mode, React also performs the extra setup-and-cleanup stress test described above. With non-empty dependencies such as [userId], cleanup runs before the next setup when userId changes and again on unmount.

# Common use cases

Use useEffect for synchronization that does not need to block the browser's next paint:

- Fetching data from an API
- Setting up subscriptions (e.g., WebSocket connections)
- Logging or analytics tracking
- Adding and removing event listeners that do not affect layout

# useLayoutEffect

useLayoutEffect runs synchronously during the commit phase, after React has written to the DOM but before the browser paints. Because it blocks paint, anything you do inside it delays the first visible frame — but in return you can measure the just-committed DOM and make adjustments atomically, with no flicker.

- Use it when you need to read layout (e.g. getBoundingClientRect, offsetWidth) and then synchronously set state or style so the user never sees the "wrong" frame.

- Keep the body cheap — expensive work here stalls paint.

- Effects do not run during server rendering, and useLayoutEffect cannot contribute to server HTML because there is no layout to measure. Prefer useEffect when the first painted frame can be corrected later. If the content fundamentally depends on layout, render that content only after hydration. Libraries sometimes expose a useIsomorphicLayoutEffect alias that selects useEffect on the server and useLayoutEffect in the browser.

import { useLayoutEffect, useRef, useState } from 'react';

function Tooltip({ children }) {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // Measure and commit the height before the user sees a frame
    setHeight(ref.current.getBoundingClientRect().height);
  }, []);

  return (
    <div ref={ref} style={{ marginTop: -height }}>
      {children}
    </div>
  );
}

# Common use cases

Reserve useLayoutEffect for layout reads or writes that must complete in the same frame:

- Measuring a DOM node and writing back state/style in the same frame
- Positioning tooltips, popovers, or floating elements based on layout
- Fixing flicker caused by a measure-then-correct pattern

useInsertionEffect

useInsertionEffect runs before layout effects so CSS-in-JS libraries can inject <style> tags before layout is read. It may run before or after the DOM itself is updated, refs are not attached yet, and it cannot update state. Application code should almost never use it; reach for useEffect or useLayoutEffect first.

*Q11. Reconciliation and the virtual DOM. How does React’s diffing algorithm decide what to update (element type, key, same-position comparison)? Why do index keys cause bugs in reorderable lists with inputs, and what does React Fiber change compared to the old stack reconciler?*

React uses a process called **reconciliation** to compare the previous Virtual DOM with the new Virtual DOM and determine which parts of the actual DOM need to be updated. React uses a diffing algorithm to minimize unnecessary DOM operations and improve rendering performance.

There are three important rules in reconciliation:

**1. Element Type:** If the element type changes, React destroys the old element and creates a new one. For example, if a `div` changes to a `section`, React replaces the element and resets the state of its component subtree. If the type remains the same, React updates only the changed props and attributes.

**2. Same-Position Comparison:** React generally compares elements at the same position in the tree. If the element type and identity match, React can reuse the existing component and update its changed properties instead of recreating it.

**3. Keys in Lists:** Keys help React identify which list item corresponds to which item in the next render. If I use a unique, stable ID as a key, React can correctly identify items when they are added, removed, or reordered.

For example, when rendering a list of users, I would use `key={user.id}` rather than an array index.

**Why can index keys cause bugs?**

Suppose I have a list of three input fields: A, B, and C, with indexes 0, 1, and 2. If I remove A, B and C shift to indexes 0 and 1. React may reuse the existing component instances based on those indexes rather than the actual data identities. As a result, input values, focus, or component-local state can appear on the wrong item.

This happens because React treats the index as the identity, even though the underlying item has changed. Therefore, for reorderable lists, I prefer stable and unique IDs as keys. Index keys are generally acceptable for static lists that never change order or have items inserted or removed.

**What does React Fiber change?**

Before Fiber, React used the Stack Reconciler, which performed reconciliation synchronously. Once rendering started, it generally continued until the work was completed, potentially blocking the main thread during large updates.

React Fiber introduced a new reconciliation architecture that breaks rendering work into smaller units called fibers. This allows React to pause, resume, prioritize, or abandon rendering work when appropriate. It also enables features such as concurrent rendering and smoother user interfaces.

However, Fiber does not make every render automatically faster. Its main advantage is that React can schedule rendering work more intelligently, helping keep the interface responsive during complex updates.

**In summary:** Reconciliation determines what needs to change, keys help React preserve the correct identity of list items, and Fiber enables React to schedule rendering work more flexibly instead of processing the entire tree in one uninterrupted synchronous operation.

*Q12. Rendering behaviour and re-renders. What triggers a re-render, and why does a child re-render when its parent does? When do React.memo, useMemo and useCallback actually help, and how do you use the Profiler to prove it rather than guess?*

In React, a re-render happens when a component needs to calculate its updated UI. The most common triggers are **state updates, changes in props, and context updates**. A parent component re-rendering can also cause its child components to re-render.

**1. What triggers a re-render?**

There are four common scenarios:

- **State updates:** When I call a state setter like `setCount`, React schedules a re-render if the new state is different from the previous state according to `Object.is`.
- **Props changes:** When a parent passes updated props to a child, the child needs to render again to reflect those changes.
- **Context changes:** When a context value changes, components consuming that context can re-render.
- **Parent re-renders:** By default, when a parent renders again, React also evaluates its child components, even if their props have not changed.

React may batch multiple state updates into a single render, and a re-render does not necessarily mean that React will update the actual DOM. React compares the rendered output and commits only the necessary changes.

**2. Why does a child re-render when its parent re-renders?**

By default, React does not automatically check whether a child’s props have changed before rendering it. When the parent renders again, React generally evaluates the child again to determine its updated output.

For example, if a parent updates its counter, a child that only displays a static heading may also re-render, even though the heading has not changed.

This is where `React.memo` can help by allowing React to skip rendering the child when its props remain unchanged. However, state updates inside the child and relevant context updates can still trigger its re-render.

**3. When does React.memo help?**

`React.memo` is a higher-order component that memoizes a component's rendered result based on its props. By default, React compares each prop using `Object.is`.

I use it when a component renders frequently, its props remain the same across many parent renders, and skipping its render provides a measurable performance benefit.

For example, if a product list contains many expensive product cards, memoizing each card may prevent unchanged cards from rendering again when the parent updates unrelated state.

However, `React.memo` may not help if I pass new object, array, or function references on every render, because React sees those props as changed. It also adds comparison overhead, so I avoid applying it everywhere without evidence of a performance problem.

**4. When does useMemo help?**

`useMemo` caches the result of a calculation between renders until its dependencies change.

I use it when a calculation is expensive, such as filtering and sorting thousands of products, or when a stable object reference is needed to prevent unnecessary work in a memoized child.

For example:

```jsx
const filteredProducts = useMemo(() => {
  return products.filter(product =>
    product.name.toLowerCase().includes(search.toLowerCase())
  );
}, [products, search]);
```

React recalculates the filtered list when `products` or `search` changes. Otherwise, it can reuse the cached result.

I would not use `useMemo` for every simple calculation because memoization itself has a cost. Also, it is a performance optimization, not a guarantee that React will preserve the cached value indefinitely.

**5. When does useCallback help?**

`useCallback` caches a function reference between renders until its dependencies change.

It is particularly useful when passing callbacks to a child wrapped in `React.memo`, or when a function is a dependency of another Hook, such as `useEffect`.

For example:

```jsx
const handleDelete = useCallback((id) => {
  setProducts(prev =>
    prev.filter(product => product.id !== id)
  );
}, []);
```

If I pass `handleDelete` to a memoized product card, the function reference remains stable across parent renders. This can help React skip rendering the card when its other props are also unchanged.

Without `useCallback`, a new function would normally be created on each parent render, potentially defeating the benefit of `React.memo`.

**6. How do I use the React Profiler to prove a performance issue?**

I do not add memoization based on assumptions. I first reproduce the problem and measure the component's rendering behaviour.

My approach is:

1. Open React DevTools and select the **Profiler** tab.
2. Start recording and reproduce the interaction that causes the slowdown, such as typing in a search box or updating a product quantity.
3. Stop recording and inspect the commits, component render durations, and components that rendered during each commit.
4. Identify unnecessary work. For example, if changing a search field causes hundreds of unchanged product cards to render, I investigate whether their props are stable and whether `React.memo` can prevent those renders.
5. Apply the optimization and record the same interaction again under comparable conditions.
6. Compare the results, including commit duration and the number of components rendered, to verify whether the optimization actually improved performance.

I can also use the Profiler API with `onRender` to collect measurements programmatically. I keep in mind that development Strict Mode can invoke rendering logic more than once, so I compare results under consistent conditions and use a production build when validating real-world performance.

**In summary:** State, props, context, and parent renders can trigger component rendering. `React.memo` can skip renders when props are unchanged, `useMemo` caches expensive calculation results, and `useCallback` stabilizes function references. As a developer, I use these optimizations selectively and rely on React Profiler measurements to confirm that they reduce unnecessary work rather than adding complexity without a real benefit.

*Q13. useEffect in depth. Explain the dependency array, cleanup timing, and stale closures. What bugs come from missing deps, object/function deps that change every render, and race conditions in data fetching? Why does Strict Mode run effects twice in development?*

`useEffect` is a React Hook used to synchronize a component with external systems, such as APIs, event listeners, timers, subscriptions, or browser APIs. It runs after React commits the rendered UI to the screen.

The main things I focus on when using `useEffect` are the dependency array, cleanup function, stale closures, and preventing race conditions in asynchronous operations.

**1. How does the dependency array work?**

The dependency array determines when React needs to re-run an effect.

There are three common cases:

- **No dependency array:** The effect runs after every committed render.
- **Empty dependency array `[]`:** The effect runs after the initial mount, with an additional development-only setup and cleanup cycle when Strict Mode is enabled.
- **Dependencies `[userId, search]`:** The effect runs after the initial mount and again when any dependency changes according to `Object.is`.

For example, if I fetch user details based on `userId`, I include `userId` in the dependency array so the effect runs again when the selected user changes.

```jsx
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

I include every reactive value used by the effect, such as props, state, and variables or functions declared inside the component. If a dependency is missing, the effect can continue using an outdated value.

**2. When does the cleanup function run?**

The cleanup function is returned from inside `useEffect`. React runs it in two important situations:

- Before running the effect again because its dependencies have changed.
- When the component unmounts.

For example, when adding an event listener, I remove it during cleanup to prevent duplicate listeners and memory leaks.

```jsx
useEffect(() => {
  const handleResize = () => {
    console.log(window.innerWidth);
  };

  window.addEventListener("resize", handleResize);

  return () => {
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

When the component unmounts, React removes the listener. If the effect needs to run again because a dependency changes, React cleans up the previous effect before setting up the new one.

**3. What are stale closures, and why do they happen?**

A stale closure occurs when a callback captures values from an earlier render and continues using them after those values have changed.

For example:

```jsx
const [count, setCount] = useState(0);

useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);
  }, 1000);

  return () => clearInterval(timer);
}, []);
```

Here, the effect runs initially and captures the initial value of `count`, which is `0`. Because the dependency array is empty, the interval continues logging `0`, even after the state changes.

One solution is to include `count` in the dependency array, but that recreates the interval whenever the count changes. If I only need to update state based on its previous value, I can use a functional state update:

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1);
  }, 1000);

  return () => clearInterval(timer);
}, []);
```

This works because React provides the latest state to the updater function without requiring `count` as an effect dependency.

**4. What bugs come from missing dependencies?**

Missing dependencies can cause effects to use outdated props or state, fail to fetch updated data, or execute callbacks with incorrect values.

For example, if an effect fetches user information but omits `userId`, changing the selected user will not trigger the effect again.

My approach is to follow the `exhaustive-deps` ESLint rule instead of suppressing dependency warnings without a clear reason. If adding a dependency causes repeated effects, I examine the effect's design rather than simply removing the dependency.

**5. Why can object and function dependencies cause infinite loops?**

React compares dependencies using `Object.is`. Objects, arrays, and functions created during rendering generally receive new references on each render, even if their contents or behaviour appear unchanged.

For example:

```jsx
const options = { page: 1 };

useEffect(() => {
  fetchProducts(options);
}, [options]);
```

A new `options` object is created on every render. If the effect updates state, that state update causes another render, which creates another object and triggers the effect again. This can produce repeated requests or an infinite loop.

I normally fix this by creating the object inside the effect and depending on its primitive inputs:

```jsx
useEffect(() => {
  const options = { page };

  fetchProducts(options);
}, [page]);
```

For function dependencies, I first consider moving the function inside the effect or outside the component if it does not need reactive values. If a stable function reference is genuinely necessary, I can use `useCallback` with the correct dependencies.

**6. How do race conditions happen during data fetching?**

A race condition can occur when multiple requests are in progress and an older request finishes after a newer request.

For example, a user searches for "React" and then quickly searches for "React Native". If the second request finishes first but the first request finishes later, the older response might overwrite the newer results.

I can prevent this by cancelling the previous request using `AbortController` and cleaning it up when the effect reruns or the component unmounts.

```jsx
useEffect(() => {
  const controller = new AbortController();

  async function loadProducts() {
    try {
      const response = await fetch(
        `/api/products?search=${encodeURIComponent(search)}`,
        { signal: controller.signal }
      );

      if (!response.ok) {
        throw new Error("Failed to fetch products");
      }

      const data = await response.json();
      setProducts(data);
    } catch (error) {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    }
  }

  loadProducts();

  return () => controller.abort();
}, [search]);
```

When `search` changes, React cleans up the previous effect, aborting its request before starting the new effect. This prevents obsolete requests from updating the UI in typical fetch workflows. For asynchronous operations that cannot be cancelled, I can use an `ignore` flag or request identifier to ensure only the latest relevant result updates the state.

**7. Why does Strict Mode run effects twice in development?**

In React Strict Mode, React performs an additional effect setup-and-cleanup cycle during development to help detect unsafe side effects and missing cleanup logic.

For example, an effect might run its setup, immediately run its cleanup, and then run its setup again. This can make an API request appear to happen twice if the request is not cancelled or otherwise managed appropriately.

This behaviour is intentional and does not mean React normally runs the initial effect twice in production. I use it to verify that subscriptions are removed correctly, timers are cleared, and asynchronous work is handled safely. I do not disable Strict Mode just to hide these issues.

**In summary:** I use `useEffect` to synchronize React with external systems. I specify all reactive dependencies, clean up subscriptions and asynchronous work, avoid stale closures, and handle race conditions when fetching data. Strict Mode helps expose lifecycle bugs during development, while correct effect design ensures predictable behaviour in production.

*Q14. useRef, useLayoutEffect, forwardRef and imperative handles. When is a ref the right tool vs state? How does useLayoutEffect differ from useEffect (measure DOM before paint), and when do you use useImperativeHandle?*

In React, these APIs help me work with DOM elements, store values between renders, measure layouts, and expose specific methods from child components to parent components.

**1. When do I use useRef instead of state?**

I use `useState` when a value affects the UI and changing it should trigger a re-render. I use `useRef` when I need to store a value between renders without triggering a re-render, or when I need a reference to a DOM element.

For example, if I am building a search screen, I would use state to store the search text because the UI needs to update when the text changes. However, I would use a ref to focus the input programmatically or store a timer ID.

```jsx
const [search, setSearch] = useState("");
const inputRef = useRef(null);

const focusSearch = () => {
  inputRef.current?.focus();
};

return (
  <>
    <input
      ref={inputRef}
      value={search}
      onChange={(e) => setSearch(e.target.value)}
    />
    <button onClick={focusSearch}>Focus Search</button>
  </>
);
```

Here, `search` is state because it affects rendering, while `inputRef` provides access to the actual DOM element.

I can also use refs to store previous values, timer IDs, or mutable instance-like data. However, I avoid reading or writing `ref.current` during rendering to manage UI state because changing a ref does not trigger a re-render.

**2. What is useLayoutEffect, and how is it different from useEffect?**

Both hooks run after React commits changes to the DOM, but their timing is different.

- **`useEffect`:** Generally runs after the browser paints the updated screen, although React may run certain effects before paint depending on how the update was triggered.
- **`useLayoutEffect`:** Runs synchronously after DOM mutations but before the browser paints. This allows me to measure layout and synchronously update something before the user sees the result.

For example, if I need to position a tooltip based on the dimensions of an element, I can measure it using `useLayoutEffect` to avoid showing it in the wrong position for a frame.

```jsx
const tooltipRef = useRef(null);
const [height, setHeight] = useState(0);

useLayoutEffect(() => {
  if (tooltipRef.current) {
    setHeight(
      tooltipRef.current.getBoundingClientRect().height
    );
  }
}, []);

return (
  <div ref={tooltipRef}>
    Tooltip height: {height}px
  </div>
);
```

The layout effect measures the element before paint, and the state update allows React to render the correct result before the browser displays it.

However, `useLayoutEffect` blocks painting while it runs, so I avoid using it for ordinary data fetching, logging, or non-visual side effects. For most effects, `useEffect` is the better default.

**3. What is forwardRef, and when do I use it?**

`forwardRef` allows a parent component to pass a ref through a custom child component to an underlying DOM element or another ref-capable target.

For example, if I create a reusable input component, the parent might need to focus its input programmatically. Without forwarding the ref, the parent cannot directly access the underlying input through the custom component.

```jsx
const CustomInput = forwardRef(function CustomInput(
  props,
  ref
) {
  return <input {...props} ref={ref} />;
});

function App() {
  const inputRef = useRef(null);

  return (
    <>
      <CustomInput ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>
        Focus Input
      </button>
    </>
  );
}
```

This makes reusable components more flexible because the parent can access the underlying input when necessary.

One important version detail: in React 18 and earlier, `forwardRef` is the standard approach for passing refs into function components. In React 19, function components can receive `ref` as a prop, so `forwardRef` is no longer necessary for new components in many cases.

**4. What is useImperativeHandle, and when do I use it?**

`useImperativeHandle` lets a child component customize the value exposed through its ref. Instead of exposing the entire DOM element, I can expose only specific methods that the parent needs.

For example, suppose I build a reusable search input and want the parent to be able to focus or clear it without accessing its internal implementation.

```jsx
const SearchInput = forwardRef(function SearchInput(
  props,
  ref
) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current?.focus();
    },
    clear() {
      if (inputRef.current) {
        inputRef.current.value = "";
      }
    }
  }), []);

  return <input ref={inputRef} {...props} />;
});
```

The parent can now call `searchRef.current.focus()` or `searchRef.current.clear()` without directly accessing the internal input element.

I use this pattern for specialized reusable components such as custom inputs, video players, editors, or components that need a small imperative API. I avoid using it for ordinary application logic because React's declarative props and state are generally easier to maintain.

**5. What is the key difference between these APIs?**

- `useRef` stores mutable values or references without triggering re-renders.
- `useLayoutEffect` performs layout-related work after DOM mutations but before paint.
- `forwardRef` allows a parent-provided ref to reach a child component's underlying target in React 18 and earlier.
- `useImperativeHandle` controls which methods or values a child exposes through its ref.

**In summary:** I choose state for values that affect the UI and refs for mutable values or imperative access. I use `useLayoutEffect` only when I need to measure or adjust layout before paint, and I use `forwardRef` with `useImperativeHandle` when a reusable component needs to expose a controlled imperative API to its parent.

*Q15. Custom hooks and composition. Design useDebounce, useLocalStorage or useFetch with abort and cache. What are the rules of hooks and why do they exist (call order and the linked list of hook state)? When do you choose a custom hook vs render props vs a compound component?*

Custom Hooks allow me to extract reusable stateful logic from React components without duplicating code. For example, if multiple screens need debouncing, local storage, or API fetching, I can move that logic into custom Hooks and reuse it across components.

**1. How do you design a useDebounce Hook?**

I use `useDebounce` when I want to delay reacting to rapidly changing values, such as search input. Instead of making an API request on every keystroke, I wait until the user stops typing for a specified duration.

```jsx
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

Here, whenever `value` changes, the effect clears the previous timer and starts a new one. If the user continues typing, the timer keeps resetting. Once the user stops typing, the debounced value updates after the delay.

I can then use this Hook in a search component:

```jsx
const [search, setSearch] = useState("");
const debouncedSearch = useDebounce(search, 500);

useEffect(() => {
  if (debouncedSearch.trim()) {
    fetchProducts(debouncedSearch);
  }
}, [debouncedSearch]);
```

This reduces unnecessary API requests and improves the user experience.

**2. How do you design a useLocalStorage Hook?**

I use `useLocalStorage` when I need to persist simple client-side preferences, such as a theme, selected language, or table display settings, across page reloads.

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored !== null
        ? JSON.parse(stored)
        : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error("Failed to save value", error);
    }
  }, [key, value]);

  return [value, setValue];
}
```

The lazy initializer reads the stored value when the component initializes, and the effect persists changes.

In a production implementation, I would also consider server-side rendering, storage availability, cross-tab synchronization through the `storage` event, and changing the key dynamically. I would not use `localStorage` to store sensitive authentication credentials because JavaScript running on the page can access them.

**3. How would you design a useFetch Hook with abort and caching?**

For API fetching, I would handle loading, error, and data states. I would also use `AbortController` to cancel obsolete requests and a cache to avoid repeatedly requesting the same resource.

A simplified implementation could look like this:

```jsx
const cache = new Map();

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) {
      setData(null);
      setLoading(false);
      return;
    }

    const controller = new AbortController();
    let ignore = false;

    async function load() {
      setLoading(true);
      setError(null);

      try {
        if (cache.has(url)) {
          setData(cache.get(url));
          setLoading(false);
          return;
        }

        const response = await fetch(url, {
          signal: controller.signal
        });

        if (!response.ok) {
          throw new Error("Request failed");
        }

        const result = await response.json();

        if (!ignore) {
          cache.set(url, result);
          setData(result);
        }
      } catch (err) {
        if (!ignore && err.name !== "AbortError") {
          setError(err);
        }
      } finally {
        if (!ignore) {
          setLoading(false);
        }
      }
    }

    load();

    return () => {
      ignore = true;
      controller.abort();
    };
  }, [url]);

  return { data, loading, error };
}
```

This example illustrates the basic approach, but a production-ready Hook needs additional safeguards. For example, it should reset or identify data when the URL changes, define cache expiration and invalidation, handle HTTP errors, and avoid caching failed or inappropriate responses. The cache shown here is module-level and shared only within that JavaScript runtime; it has no expiration policy.

For complex applications, I would generally prefer **TanStack Query or SWR** instead of building a complete caching layer from scratch. These libraries provide features such as request deduplication, retries, stale-data handling, cache invalidation, and background refetching.

**4. What are the Rules of Hooks, and why do they exist?**

React has two main Rules of Hooks:

1. Call Hooks only at the top level of a React function component or a custom Hook. Do not call them inside loops, conditions, or nested functions.
2. Call Hooks only from React function components or custom Hooks, not from ordinary JavaScript functions or class components.

For example, this is incorrect:

```jsx
if (isLoggedIn) {
  const [user, setUser] = useState(null);
}
```

The problem is that the number or order of Hook calls could change between renders.

React associates Hook state with a component's position in the rendered tree and, in its current implementation, tracks Hook records in a linked list associated with the component's Fiber. Each Hook call corresponds to a position in that sequence.

If the first render calls `useState`, then `useEffect`, but the next render skips `useState`, React could associate subsequent Hook calls with the wrong state records. Maintaining a consistent call order lets React reliably match each Hook call with its stored state.

The correct approach is to call Hooks unconditionally at the top level and put conditions inside the effect or callback instead.

**5. When do you choose a custom Hook vs render props vs a compound component?**

I choose the pattern based on whether I need to reuse logic, customize rendering, or coordinate a group of related UI components.

**Custom Hooks:** I use these when I want to reuse behaviour and stateful logic without sharing a particular UI structure. Examples include `useDebounce`, `useAuth`, `useFetch`, and `useWindowSize`. They keep components smaller and separate business logic from presentation.

**Render props:** I use this pattern when a component needs to share behaviour but let the consumer control how the UI is rendered through a function prop. For example, a reusable data provider could pass loading state and results to a render function. It remains useful for certain flexible APIs, although custom Hooks often provide a simpler alternative for logic reuse.

**Compound components:** I use this pattern when several components form one coordinated UI and should share state or behaviour. For example, a Tabs component might contain `Tabs.List`, `Tabs.Trigger`, and `Tabs.Content`. The parent manages the active tab while the child components provide a declarative API.

In a real application, I might use a custom Hook to manage tab selection and compound components to expose the visual API.

**In summary:** I use custom Hooks to reuse stateful logic, follow the Rules of Hooks to preserve consistent state association, and choose render props or compound components when consumers need greater control over rendering or a coordinated UI structure. For data fetching, I also account for cancellation, stale responses, caching, and error handling rather than treating an API call as just a `fetch` inside an effect.

*Q16. State architecture. Where should state live: local, lifted, context, or an external store? Why does Context cause wide re-renders when its value changes, and how do you split contexts, use selectors, or move to Zustand/Redux Toolkit to avoid it?*

In React, state architecture means deciding where state should live, how components access it, and how updates should be managed efficiently. I choose the approach based on how many components need the data, how frequently it changes, and whether the data belongs to the server or the client.

**1. Where should state live?**

I generally divide state into four categories:

**Local state:** I use `useState` or `useReducer` when state belongs to a single component or a small component subtree. For example, a dropdown's open state, a form input, or a modal's visibility. Keeping state local reduces unnecessary dependencies between components.

**Lifted state:** When multiple sibling components need the same state, I move it to their nearest common parent and pass it down through props. For example, if a search input and product list need to share a search query, the parent can own that state. However, if lifting state creates excessive prop drilling across many levels, I consider Context or an external store.

**Context:** I use Context for values that many components need, such as the current authenticated user, theme, locale, or dependency configuration. It avoids passing the same props through multiple intermediate components. However, I avoid putting all frequently changing application state into one Context.

**External state management:** I use Zustand or Redux Toolkit when state is shared across many unrelated components, requires selective subscriptions, or needs structured update patterns, debugging, and predictable state transitions. Redux Toolkit is particularly useful when an application benefits from centralized actions, reducers, middleware, and Redux DevTools.

I also distinguish **server state from client state**. API data, caching, retries, and background refetching generally belong in a server-state library such as TanStack Query. UI state, such as a selected tab or open modal, usually belongs in React state or a client-state store.

**2. Why can Context cause wide re-renders?**

When a Context provider receives a different `value`, React notifies components that consume that Context. React compares the previous and next values using `Object.is`.

For example:

```jsx
const AppContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState("light");

  const value = { user, theme, setUser, setTheme };

  return (
    <AppContext.Provider value={value}>
      {children}
    </AppContext.Provider>
  );
}
```

Every time the provider renders, it creates a new object for `value`. Even if only `theme` changes, the Context value has a different reference, so consumers of this Context are notified and can re-render.

For example, a component that only needs `user` may re-render when `theme` changes because both values belong to the same Context.

One important distinction is that Context does not automatically re-render every component beneath the provider. The issue is that **consumers of a changed Context value cannot skip the update simply because they use only one property of that value**. `React.memo` alone does not prevent a component from receiving updated Context.

**3. How do I optimize Context?**

My first approach is to split Context according to the responsibility and update frequency of the data.

For example, instead of one large application Context, I could create separate `AuthContext`, `ThemeContext`, and `SettingsContext` providers. A theme update would then not notify components that consume only the authentication Context.

I can also separate state from dispatch when using `useReducer`, because the dispatch function has a stable identity:

```jsx
const [state, dispatch] = useReducer(reducer, initialState);

return (
  <StateContext.Provider value={state}>
    <DispatchContext.Provider value={dispatch}>
      {children}
    </DispatchContext.Provider>
  </StateContext.Provider>
);
```

Components that only dispatch actions can consume `DispatchContext` without subscribing to changes in the state Context.

Another option is to stabilize a Context value with `useMemo` and callbacks with `useCallback` when appropriate. This prevents unnecessary Context updates caused purely by newly created references, provided the dependencies have not changed. However, it does not prevent consumers from updating when the actual Context value changes.

**4. How do selectors help?**

A selector allows a component to subscribe to a specific part of a store rather than the entire state.

For example, suppose an application store contains the current user, theme, cart, and notifications. A component that only needs the cart count should not need to re-render whenever an unrelated notification changes.

With Zustand, I can select only the required value:

```jsx
const cartCount = useStore(
  state => state.cart.length
);

const theme = useStore(
  state => state.theme
);
```

These components subscribe to different selected values. When unrelated store state changes, a component can avoid re-rendering if its selected result remains equal according to the store's comparison behaviour. For computed objects or arrays, I need to consider selector equality and reference stability.

This is one reason I consider Zustand for applications with frequently changing shared client state: its selector-based subscriptions make it straightforward to subscribe to individual pieces of state without requiring a large Context value.

**5. When would I choose Zustand vs Redux Toolkit?**

**Zustand:** I choose Zustand when I need a lightweight shared store with minimal boilerplate and straightforward selector-based subscriptions. It works well for UI state, shopping carts, filters, and other shared client-side data.

**Redux Toolkit:** I choose Redux Toolkit when the application benefits from explicit actions and reducers, centralized state transitions, middleware, strong debugging support, and consistent conventions across a larger team. Its selector ecosystem also helps components subscribe to specific derived values. Redux Toolkit does not eliminate re-renders automatically; state subscriptions and selector equality still need to be designed correctly.

I would not introduce either library just because an application has several components. For a small feature, local state or a simple Context may be sufficient.

**6. How would I decide in a real application?**

For example, in a restaurant ERP application:

- A dropdown's open/closed state stays local.
- A search query shared between a toolbar and product list is lifted to their nearest common parent.
- The current authenticated user and theme can be provided through separate Contexts.
- Frequently updated shared client state, such as a shopping cart or active order draft, may live in Zustand or Redux Toolkit depending on the application's complexity.
- Menu items, inventory records, and API responses can be managed through TanStack Query when server-state caching and synchronization are needed.

I would then use React DevTools Profiler to identify components that render unnecessarily and verify whether splitting Contexts, introducing selectors, or changing the state location actually improves performance.

**In summary:** I keep state as close as practical to the components that need it. I use local state for isolated UI behaviour, lift state for related components, Context for shared values with appropriate update patterns, and Zustand or Redux Toolkit for more complex shared client state. For API data, I prefer a dedicated server-state solution. The goal is not to minimize every render, but to create a predictable architecture that avoids unnecessary coupling and expensive rendering work.

*Q17. Server state with TanStack Query or SWR. What problems does it solve over useEffect + useState (caching, deduping, stale-while-revalidate, retries, background refetch)? How do you do optimistic updates, pagination, infinite queries and cache invalidation after a mutation?*

Server state is data that comes from an external source, usually an API or database, and needs to be fetched, cached, synchronized, and updated in the frontend. Examples include product lists, user profiles, restaurant menus, and inventory records.

I use TanStack Query or SWR instead of managing everything manually with `useEffect` and `useState` because these libraries handle common data-fetching problems such as caching, request deduplication, retries, background refetching, and synchronization.

**1. What problems do TanStack Query and SWR solve?**

With `useEffect` and `useState`, I would need to manage loading states, errors, request cancellation, caching, duplicate requests, retries, and stale responses myself. As an application grows, this creates a lot of repetitive code.

TanStack Query and SWR provide these capabilities:

- **Caching:** Store fetched data by query key so components can reuse previously fetched results.
- **Request deduplication:** Combine equivalent requests that occur close together rather than unnecessarily requesting the same resource multiple times.
- **Stale-while-revalidate:** Display cached data immediately when available while fetching fresh data in the background.
- **Retries:** Retry failed requests according to configurable policies.
- **Background refetching:** Refresh data when configured events occur, such as window focus, network reconnection, or a query becoming active.
- **Request lifecycle management:** Track loading, success, and error states and coordinate updates to subscribed components.

For example, if multiple components need the same restaurant menu, TanStack Query can share the cached result instead of making each component independently manage an API request.

One important point is that cached data is not necessarily fresh data. I configure freshness, refetching, and cache retention according to the requirements of the feature.

**2. How do you fetch data using TanStack Query?**

I use `useQuery` for fetching and caching data. A query key uniquely identifies the data, and the query function performs the API request.

```jsx
const { data, isPending, isError, error } = useQuery({
  queryKey: ["products", restaurantId],
  queryFn: () => fetchProducts(restaurantId),
  staleTime: 60_000,
});
```

Here, `["products", restaurantId]` identifies the products for a particular restaurant. If the restaurant ID changes, TanStack Query treats it as a different query.

The `staleTime` of 60 seconds means the data is considered fresh for that period. It does not mean the cached data is deleted after 60 seconds; cache retention is controlled separately.

**3. How do you implement optimistic updates?**

An optimistic update means updating the UI immediately before the server confirms the operation. It makes the application feel faster, especially for actions such as toggling a favourite, updating a cart quantity, or changing an order status.

For example, when a user updates a product quantity, I can optimistically update the cached data, send the mutation to the server, and roll back if the request fails.

```jsx
const queryClient = useQueryClient();

const updateQuantity = useMutation({
  mutationFn: updateProductQuantity,

  onMutate: async ({ id, quantity }) => {
    await queryClient.cancelQueries({
      queryKey: ["products"],
    });

    const previousProducts = queryClient.getQueryData([
      "products",
    ]);

    queryClient.setQueryData(["products"], old =>
      old?.map(product =>
        product.id === id
          ? { ...product, quantity }
          : product
      )
    );

    return { previousProducts };
  },

  onError: (_error, _variables, context) => {
    queryClient.setQueryData(
      ["products"],
      context?.previousProducts
    );
  },

  onSettled: () => {
    queryClient.invalidateQueries({
      queryKey: ["products"],
    });
  },
});
```

The process is:

1. Cancel relevant in-flight queries so an older response does not immediately overwrite the optimistic state.
2. Save a snapshot of the previous cached data.
3. Update the cache immediately.
4. Roll back to the snapshot if the mutation fails.
5. Invalidate the relevant query after the mutation settles so the application can reconcile with the server's actual state.

In production, I ensure the query keys match the exact cached data shape and consider concurrent mutations. A simple snapshot rollback can overwrite another successful optimistic update if multiple mutations happen simultaneously. For high-contention data, I may need mutation sequencing, operation-specific rollback, or server-authoritative reconciliation.

**4. How do you handle pagination?**

For ordinary page-based pagination, I include the current page in the query key. Each page can then be cached independently.

```jsx
const { data, isPending, isPlaceholderData } = useQuery({
  queryKey: ["products", page],
  queryFn: () => fetchProducts({ page }),
  placeholderData: keepPreviousData,
});
```

`keepPreviousData` allows the previous page's data to remain visible while the next page loads, reducing flicker. I can disable the Next button while the requested page is not yet available or while placeholder data is being displayed.

For large datasets, I also consider cursor-based pagination because it can be more efficient and reliable than offset-based pagination when records are frequently inserted or removed.

**5. How do you implement infinite queries?**

I use `useInfiniteQuery` when users load more records as they scroll, such as an order history, product catalogue, or activity feed.

```jsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ["products"],
  queryFn: ({ pageParam }) =>
    fetchProducts({ cursor: pageParam }),

  initialPageParam: null,

  getNextPageParam: lastPage =>
    lastPage.nextCursor ?? undefined,
});
```

The API returns the current page of results along with a cursor for the next page. TanStack Query stores the pages together, and I call `fetchNextPage()` when the user reaches the bottom of the list.

I check `hasNextPage` before loading more data and use `isFetchingNextPage` to prevent unnecessary repeated calls. For large lists, I also use list virtualization to avoid rendering thousands of DOM elements simultaneously.

**6. How do you invalidate the cache after a mutation?**

After a mutation changes server data, I invalidate queries whose cached results may now be outdated.

For example, after creating a product:

```jsx
const createProduct = useMutation({
  mutationFn: createProductApi,

  onSuccess: () => {
    queryClient.invalidateQueries({
      queryKey: ["products"],
    });
  },
});
```

TanStack Query marks matching queries as stale and normally refetches active matching queries according to its behaviour and configuration. Inactive queries are generally refetched when they become active again.

I choose invalidation based on which data the mutation affects. If creating a product changes the product list and the inventory summary, I invalidate both relevant query keys. If the server returns the complete updated entity, I may update a specific cached query directly with `setQueryData` rather than refetching everything.

**7. TanStack Query vs SWR**

Both libraries provide caching, request deduplication, revalidation, and shared server-state management.

I generally choose **TanStack Query** when I need a comprehensive solution for mutations, optimistic updates, infinite queries, pagination, and more explicit cache management.

I consider **SWR** when I want a lightweight API centred on fetching, caching, and revalidation. SWR also supports mutation APIs, optimistic updates, and infinite loading, so it is not limited to basic data fetching.

**In summary:** TanStack Query and SWR remove much of the boilerplate involved in managing server state manually. I use query keys to identify cached data, freshness policies to control refetching, mutations for server updates, optimistic updates for responsive interactions, and targeted invalidation to keep the UI synchronized with the backend. I still rely on the server for data integrity because client-side caching and optimistic updates do not replace database transactions or server-side validation.

*Q18. Forms at scale. Compare controlled vs uncontrolled inputs. How do React Hook Form or Formik reduce re-renders, how do you integrate schema validation (Zod/Yup), and how do you handle dynamic field arrays, async validation and accessibility of error messages?*

In React, form architecture becomes important when a form contains many fields, conditional sections, dynamic inputs, validation rules, and asynchronous checks. For small forms, I can use React state directly, but for complex forms, I generally prefer React Hook Form because it minimizes unnecessary re-renders and provides good support for schema validation and dynamic field arrays.

**1. Controlled vs uncontrolled inputs**

A controlled input is one where React state manages the input's value. Whenever the user types, the change handler updates state, causing a re-render.

```jsx
const [email, setEmail] = useState("");

<input
  value={email}
  onChange={e => setEmail(e.target.value)}
/>
```

I use controlled inputs when the UI needs to react immediately to each change, such as live previews, conditional fields, or character counters. The downside is that frequent state updates can cause unnecessary rendering if the form is large and state is managed poorly.

An uncontrolled input stores its current value in the DOM, and React accesses it through a ref or form submission.

```jsx
const emailRef = useRef(null);

<input ref={emailRef} defaultValue="" />

const handleSubmit = () => {
  console.log(emailRef.current?.value);
};
```

This avoids updating React state on every keystroke. However, validation, tracking values, and coordinating dependent fields require additional handling if implemented manually.

**2. How do React Hook Form and Formik reduce re-renders?**

React Hook Form primarily uses uncontrolled inputs and refs to track field values. It subscribes components to specific form-state updates, so changing one field does not necessarily re-render the entire form.

For example, with `register`, I can connect an input to the form without controlling its value through React state on every keystroke.

Formik traditionally uses a controlled form-state model. It manages values, errors, touched fields, and submission through React state. This provides a straightforward programming model, but updates to form state can cause more component re-renders, depending on the component structure and how subscriptions or memoization are used.

My choice depends on the form's complexity and performance requirements. For a large form with many fields, I generally start with React Hook Form. For either library, I still profile the actual application instead of assuming that a library automatically eliminates every unnecessary render.

**3. How do you integrate schema validation using Zod or Yup?**

I prefer schema-based validation because it centralizes validation rules and makes them reusable across fields and submission logic.

For example, with React Hook Form and Zod:

```jsx
import { useForm } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

const schema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Enter a valid email"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

function RegistrationForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    resolver: zodResolver(schema),
    mode: "onBlur",
  });

  const onSubmit = async data => {
    await registerUser(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        autoComplete="email"
        aria-invalid={Boolean(errors.email)}
        aria-describedby={errors.email ? "email-error" : undefined}
        {...register("email")}
      />

      {errors.email && (
        <p id="email-error" role="alert">
          {errors.email.message}
        </p>
      )}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Submitting..." : "Register"}
      </button>
    </form>
  );
}
```

Here, `zodResolver` connects the Zod schema to React Hook Form. The schema validates the data, while the resolver maps validation errors into the form's error state.

I can use Yup in a similar way through `yupResolver`. I choose Zod when I want a schema API with strong TypeScript inference, while Yup is also a good choice, particularly in existing projects that already use it.

**4. How do you handle dynamic field arrays?**

Dynamic field arrays are useful when users can add or remove items, such as restaurant menu items, invoice lines, or employee details.

React Hook Form provides `useFieldArray` for this purpose.

```jsx
const { control, register } = useForm({
  defaultValues: {
    items: [{ name: "", quantity: 1 }],
  },
});

const { fields, append, remove } = useFieldArray({
  control,
  name: "items",
});

return (
  <>
    {fields.map((field, index) => (
      <div key={field.id}>
        <input
          {...register(`items.${index}.name`)}
          aria-label={`Item ${index + 1} name`}
        />

        <input
          type="number"
          {...register(`items.${index}.quantity`, {
            valueAsNumber: true,
          })}
          aria-label={`Item ${index + 1} quantity`}
        />

        <button type="button" onClick={() => remove(index)}>
          Remove
        </button>
      </div>
    ))}

    <button
      type="button"
      onClick={() => append({ name: "", quantity: 1 })}
    >
      Add Item
    </button>
  </>
);
```

The important detail is using `field.id` as the React key instead of the array index. This helps React preserve the correct identity of each field when items are removed or reordered.

I also validate the array structure in the schema, including required fields, minimum item counts, and constraints such as quantity being greater than zero.

**5. How do you handle asynchronous validation?**

Asynchronous validation is useful when a field requires a server check, such as verifying whether an email address or username is already registered.

I generally separate client-side validation from server-side validation. Basic rules such as email format should run locally, while uniqueness checks should call the backend.

For async validation, I debounce requests where appropriate, handle loading states, and prevent stale responses from overwriting newer results. I also avoid treating a client-side uniqueness check as authoritative because another user could claim the value before submission.

For example, I can use a debounced availability check while the user types, then still handle a server-side conflict during form submission. I ensure that failed network requests are presented as validation-service errors rather than incorrectly reporting that the input itself is invalid.

**6. How do you make form errors accessible?**

Accessibility is part of the form architecture, not just a styling decision.

I follow these practices:

- Associate every input with a visible label using `htmlFor` and `id`.
- Set `aria-invalid="true"` when a field contains a validation error.
- Connect the field to its error message using `aria-describedby`.
- Provide clear, specific error messages rather than relying only on red borders or icons.
- Use an appropriate live announcement mechanism, such as `role="alert"` for important validation errors, without making every keystroke excessively disruptive.
- On failed submission, focus the first invalid field or present a linked error summary for long forms.
- Use native input types and autocomplete attributes where appropriate.
- Ensure keyboard users can navigate all controls and understand which fields are required.

For example, an email error should explain what the user needs to correct, and the input should be programmatically associated with that message so assistive technology can identify it.

**7. How do you manage performance in very large forms?**

I use React Hook Form's field-level subscriptions, such as `useWatch` and `useFormState`, to subscribe only to the values or form state that a component needs. I avoid watching the entire form in a parent unless the whole form genuinely depends on every field.

I also choose validation timing deliberately. Validating on every keystroke can be useful for some interfaces, but validating on blur or submit may be more efficient for expensive rules. For large forms, I may divide the form into logical sections, use conditional rendering carefully, and avoid expensive calculations during every render.

**In summary:** I choose controlled inputs when the UI needs immediate React-driven updates and uncontrolled inputs when I want efficient form value management without a render for every keystroke. React Hook Form is often my choice for large forms because of its subscription-based architecture. I combine it with Zod or Yup for schema validation, `useFieldArray` for dynamic fields, carefully managed asynchronous validation, and accessible error messages. Finally, I profile the form and optimize based on measured bottlenecks rather than adding complexity prematurely.

*Q19. Performance and code splitting. How do React.lazy, Suspense and route-based splitting reduce bundle size? When do you virtualize lists (react-window), and how do you handle expensive renders with useTransition/useDeferredValue? How do you analyze a bundle and fix a heavy dependency?*

In React, performance optimization means reducing the amount of JavaScript users need to download, minimizing unnecessary rendering work, and keeping the UI responsive during expensive operations. I usually focus on bundle size, list rendering, expensive computations, and measuring performance before choosing an optimization.

**1. How do React.lazy and Suspense reduce bundle size?**

By default, if an application imports all its pages and components statically, their code can end up in the initial JavaScript bundle. This increases download, parsing, and execution time, especially on slower networks and devices.

`React.lazy` allows me to load a component dynamically, while `Suspense` displays a fallback UI until that component's code has loaded.

For example:

```jsx
import { lazy, Suspense } from "react";

const Reports = lazy(() => import("./pages/Reports"));

function App() {
  return (
    <Suspense fallback={<p>Loading reports...</p>}>
      <Reports />
    </Suspense>
  );
}
```

Here, the Reports component can be placed in a separate JavaScript chunk by the bundler. The browser downloads that chunk when React first needs to render the lazy component, rather than necessarily downloading it with the initial application code.

**2. Why is route-based code splitting useful?**

In a real application, such as a restaurant ERP, users may have separate Dashboard, Inventory, Reports, and Settings pages. Loading every page's code upfront is unnecessary if the user initially visits only the Dashboard.

I can split the application by route so each page loads when required.

```jsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Inventory = lazy(() => import("./pages/Inventory"));
const Reports = lazy(() => import("./pages/Reports"));

function AppRoutes() {
  return (
    <Suspense fallback={<div>Loading page...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/inventory" element={<Inventory />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  );
}
```

This can reduce the initial JavaScript payload and improve the time to interactive. The exact benefit depends on the bundler, shared dependencies, and how imports are structured. Code splitting does not necessarily reduce the total JavaScript downloaded if the user eventually visits every route; it primarily defers work until needed.

In production, I also use appropriately sized loading fallbacks, handle chunk-loading errors, and avoid splitting every tiny component because excessive chunks can introduce network overhead.

**3. When do I virtualize lists using react-window?**

When a list contains hundreds or thousands of records, rendering every item into the DOM can consume significant memory and increase rendering and layout costs.

Virtualization renders only the visible items and a small overscan region around them. As the user scrolls, the library updates which items are mounted.

For example, in an ERP inventory screen with 10,000 products, I would consider virtualization rather than rendering all 10,000 rows simultaneously.

```jsx
import { FixedSizeList } from "react-window";

function InventoryList({ products }) {
  return (
    <FixedSizeList
      height={500}
      width="100%"
      itemCount={products.length}
      itemSize={50}
      itemData={products}
    >
      {({ index, style, data }) => (
        <div style={style}>
          {data[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

This example uses the API of `react-window` versions that expose `FixedSizeList`; newer versions may use a different API, so I check the installed version before implementing it.

I consider virtualization when profiling shows that a large number of DOM nodes is causing slow rendering, scrolling, or layout. It is particularly useful for data tables, transaction histories, and searchable catalogues.

However, virtualization adds complexity around dynamic row heights, keyboard navigation, accessibility, and maintaining focus when items are unmounted. I avoid it for small lists where normal rendering is already fast.

**4. How do useTransition and useDeferredValue help with expensive renders?**

React provides these Hooks to keep urgent user interactions responsive while allowing less urgent rendering work to be processed in the background.

**useTransition:** I use this when I control the state update and want to mark an update as non-urgent.

For example, when a user selects a filter that causes a large table to re-render, I can keep the immediate input interaction responsive while marking the expensive results update as a transition.

```jsx
const [filter, setFilter] = useState("");
const [isPending, startTransition] = useTransition();

function handleFilterChange(value) {
  setFilter(value);

  startTransition(() => {
    setSelectedCategory(value);
  });
}
```

Here, the input's own value updates urgently, while the category update is marked as non-urgent. React can prioritize urgent updates over this transition. The actual performance benefit depends on how expensive the resulting rendering work is.

**useDeferredValue:** I use this when a value comes from somewhere else, and I want an expensive part of the UI to lag behind that value temporarily.

For example:

```jsx
const [search, setSearch] = useState("");
const deferredSearch = useDeferredValue(search);

const filteredProducts = products.filter(product =>
  product.name
    .toLowerCase()
    .includes(deferredSearch.toLowerCase())
);
```

The input can update immediately, while the expensive product-list rendering uses the deferred value. React can interrupt and restart the background rendering when newer input arrives.

For this optimization to be effective, the expensive list may also need memoization so it does not repeat expensive work during urgent renders when its relevant props have not changed.

An important distinction is that neither Hook automatically debounces network requests or makes an expensive calculation intrinsically faster. They help React schedule rendering work more intelligently. For search APIs, I would use debouncing separately if I need to reduce the number of requests.

**5. How do I analyze a bundle and identify a heavy dependency?**

I do not assume which dependency is responsible for a large bundle. I inspect the production build and use a bundle analyzer appropriate for the bundler.

For a Vite application, I can use `rollup-plugin-visualizer` to generate a treemap showing the relative size of modules and dependencies in the build output.

My workflow is:

1. Create a production build using the normal build command.
2. Generate a bundle visualization and identify the largest dependencies and chunks.
3. Check whether large libraries are imported unnecessarily, whether the application imports an entire package instead of specific modules, and whether duplicated dependencies exist.
4. Replace or optimize dependencies only after confirming the cost and checking the required functionality.
5. Apply code splitting where appropriate, then rebuild and compare the results.

For example, if a small dashboard imports a large charting library but only uses a simple bar chart, I would check whether a smaller alternative meets the requirements. If a PDF-generation library is needed only on the Reports page, I would consider dynamically importing it when that feature is opened.

I also distinguish between the raw JavaScript size and the compressed transfer size, such as gzip or Brotli. A large dependency in the build does not always have the same impact on network transfer, but its parsing and execution cost can still affect performance.

Finally, I validate the change using bundle metrics and browser performance tools rather than assuming that a smaller package automatically makes the entire application faster.

**In summary:** I use route-based code splitting to defer page code, virtualization to reduce the number of mounted DOM elements, and `useTransition` or `useDeferredValue` to keep urgent interactions responsive during expensive rendering. For bundle optimization, I inspect production build output, identify heavy dependencies, make targeted changes, and measure the results before and after.

*Q20. React 18/19 concurrency and rendering models. What do automatic batching, transitions and concurrent rendering change? Compare CSR, SSR, SSG and React Server Components: what runs where, what hydration is, and why hydration mismatches happen.*

React 18 introduced important improvements in how React schedules updates, including automatic batching and concurrent rendering capabilities. React 19 builds on this model and adds improvements such as Actions and better support for asynchronous workflows. I understand these concepts in two parts: how React schedules UI updates and where the application renders.

**1. What is automatic batching?**

Automatic batching means React groups multiple state updates into a single render instead of rendering separately for every update.

For example:

```jsx
function handleClick() {
  setCount(count => count + 1);
  setLoading(true);
  setError(null);
}
```

React can process these state updates together and commit the resulting UI in one render rather than triggering a separate render for each update.

Before React 18, automatic batching was primarily associated with React event handlers. With React 18's `createRoot` API, batching also applies to updates inside promises, timers, and native event handlers.

The benefit is reduced rendering work and fewer intermediate UI states. However, batching does not mean that every update across an entire application becomes one render; separate intentional events and commits can still occur.

**2. What are transitions and concurrent rendering?**

Concurrent rendering allows React to prepare UI updates without requiring every rendering task to finish in one uninterrupted synchronous operation. React can pause, resume, restart, or abandon render work when necessary to prioritize more urgent updates.

For example, in a product catalogue, typing into a search field should feel immediate, but rendering thousands of filtered products may be less urgent.

I can use `useTransition` to mark the expensive state update as non-urgent:

```jsx
const [search, setSearch] = useState("");
const [filter, setFilter] = useState("");
const [isPending, startTransition] = useTransition();

function handleChange(value) {
  setSearch(value);

  startTransition(() => {
    setFilter(value);
  });
}
```

The input updates urgently, while the filtering-related update is scheduled as a transition. React can prioritize urgent interactions and interrupt background rendering when newer work arrives.

A key distinction is that **concurrent rendering does not mean React executes component rendering in parallel on multiple JavaScript threads**. It means React can schedule and interrupt rendering work more flexibly. Rendering logic must remain pure because React may restart or discard an unfinished render.

In React 19, Actions also help coordinate asynchronous mutations and pending states, but they do not eliminate the need to design loading, error, and data-consistency behaviour correctly.

**3. Compare CSR, SSR, SSG, and React Server Components**

These concepts are related, but they describe different aspects of rendering. CSR, SSR, and SSG describe when and where HTML is produced, while React Server Components describe where particular React components execute.

**Client-Side Rendering (CSR):**

The browser downloads the HTML and JavaScript, executes the React application, and renders the UI on the client.

- Execution: Primarily in the browser.
- Advantage: Rich interactivity and straightforward client-side navigation.
- Disadvantage: The initial experience can be slower because the browser may need to download and execute JavaScript before rendering the complete UI.

I commonly use CSR for internal dashboards, admin panels, and applications where authentication and frequent interactions are central.

**Server-Side Rendering (SSR):**

The server renders React components into HTML for each request or when the framework otherwise performs a server render. The browser receives the HTML and can display the initial content before client-side hydration makes interactive components work.

- Execution: Initial rendering on the server; interactive client components also execute in the browser.
- Advantage: Faster initial content display in many cases and better support for search-engine crawling.
- Disadvantage: Server rendering adds infrastructure and request-time processing costs, and hydration still requires client JavaScript for interactive components.

I would consider SSR for public-facing pages where initial content delivery and discoverability matter.

**Static Site Generation (SSG):**

The HTML is generated at build time and served as a static file, often through a CDN.

- Execution: HTML generation during the build; client-side JavaScript runs in the browser for interactive parts.
- Advantage: Fast delivery, strong caching, and low request-time rendering cost.
- Disadvantage: Content can become outdated unless the site is rebuilt or uses a framework's revalidation mechanism.

I would use SSG for documentation, marketing pages, and content that does not need to be freshly rendered for every request.

**React Server Components (RSC):**

Server Components execute on the server and can access server-side resources, such as databases or filesystem APIs, without shipping their component implementation to the browser as client JavaScript.

They produce a serialized React Server Component payload that React and the framework use to construct the UI. Server Components can be combined with Client Components for interactivity.

- Execution: Server Components run on the server; Client Components can run on the server for initial HTML generation in an SSR framework and then run in the browser.
- Advantage: Less client-side JavaScript for server-only component logic and direct access to server-side resources.
- Disadvantage: They require an appropriate framework or server integration, and server-only components cannot use client Hooks such as `useState` or `useEffect`.

For example, a restaurant ERP's public menu could be rendered with server-side data access, while an interactive cart uses a Client Component.

A Server Component can fetch menu data directly on the server, while the client-side cart manages user interactions. This can reduce unnecessary client JavaScript and keep database access away from the browser.

Importantly, RSC is not simply another name for SSR. Server Components can be used with different delivery strategies, and their output can be combined with server-rendered HTML and Client Components.

**4. What is hydration?**

Hydration is the process in which React attaches its client-side logic to HTML that was already rendered on the server, allowing the page to become interactive without recreating all its existing DOM from scratch.

For example, an SSR page may initially display a product list as HTML. When the browser loads the JavaScript, React hydrates the relevant Client Components so buttons, event handlers, and state updates work.

The initial client render must match the server-rendered HTML for hydration to proceed correctly. React can then attach event handling and manage subsequent updates.

**5. Why do hydration mismatches happen?**

A hydration mismatch occurs when the HTML produced by the server differs from what React expects during the initial client render.

Common causes include:

- **Browser-only APIs:** Using `window`, `localStorage`, or other browser-dependent values to produce different initial markup on the server and client.
- **Non-deterministic values:** Rendering `Date.now()`, random values, or other changing data during the initial render.
- **Locale and timezone differences:** Formatting dates or numbers differently on the server and browser.
- **Invalid HTML nesting:** For example, placing a block-level element inside a paragraph in a way the browser corrects differently from React's expected tree.
- **Different initial data:** The server renders one data snapshot while the client immediately renders another.

To prevent these issues, I keep the initial server and client output deterministic, pass the same initial data to the client, and move genuinely browser-specific work into an effect when appropriate.

For example, instead of reading `localStorage` directly during the initial render, I can initialize with a consistent default and read the stored preference after mounting. This avoids a mismatch, although it may cause a brief visual change. If that change matters, I consider a server-provided preference or another strategy that produces the same initial value on both sides.

I do not use `suppressHydrationWarning` as a general fix because it only suppresses certain warnings; it does not resolve the underlying mismatch.

**6. How do I choose the right rendering model?**

For an internal restaurant ERP, I would usually start with CSR for authenticated operational screens because the interface is highly interactive and search-engine indexing is not a priority.

For the public website, menu pages, and marketing content, I would consider SSG or SSR depending on how frequently the content changes and whether it needs request-specific data.

Where a supported framework provides React Server Components, I would use them for server-side data access and content that does not need client interactivity, then isolate interactive features into Client Components.

**In summary:** Automatic batching reduces unnecessary commits, transitions let React prioritize urgent interactions, and concurrent rendering makes rendering work interruptible. CSR, SSR, and SSG describe different HTML delivery strategies, while React Server Components determine where component logic executes. Hydration connects server-rendered HTML to client-side interactivity, and preventing mismatches requires consistent initial output between server and client.

*Q21. Error boundaries and failure handling. What can and can’t an error boundary catch (render errors vs event handlers, async code, SSR)? How do you design a layered strategy with per-route boundaries, reset on navigation, fallback UIs and reporting to Sentry? Why are there still no hook-based error boundaries?*

**1. What is an error boundary?**

An error boundary is a React component that catches JavaScript errors thrown during rendering and lifecycle methods of its child component tree. Instead of allowing the entire application to crash, it displays a fallback UI for the failed section.

In class components, error boundaries are implemented using `static getDerivedStateFromError()` to update the UI and `componentDidCatch()` to log error details.

```jsx
import React from "react";

class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    // Send error and component stack to Sentry
    console.error("React error:", error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert">
          <h2>Something went wrong.</h2>
          <button
            onClick={() => this.setState({ hasError: false })}
          >
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

Here, `getDerivedStateFromError()` determines the fallback state, while `componentDidCatch()` is used for reporting and diagnostics.

**2. What can and cannot an error boundary catch?**

An error boundary catches errors that occur while React renders its descendant components, including errors thrown by their constructors and relevant lifecycle methods.

However, it does not automatically catch every JavaScript error.

- **Render errors:** Caught when a descendant throws during rendering.
- **Event handlers:** Errors thrown directly inside event handlers, such as `onClick`, are not caught. Handle them with `try/catch`.
- **Asynchronous code:** Errors from `setTimeout`, promises, or API calls are not automatically caught. Handle promise rejections explicitly or transfer the error into a React render path when appropriate.
- **Server-side rendering:** Traditional React error boundaries do not catch errors during server rendering. Server-side errors need to be handled by the framework or server runtime.
- **Errors inside the boundary itself:** A boundary generally cannot catch errors thrown by its own rendering logic. A parent boundary is needed to handle failures in a child boundary.
- **Effects:** Errors thrown asynchronously or directly inside `useEffect` are not automatically caught by an error boundary.

For example, an API request should handle its own failure:

```jsx
async function fetchUser() {
  try {
    const response = await fetch("/api/user");

    if (!response.ok) {
      throw new Error("Failed to fetch user");
    }

    return await response.json();
  } catch (error) {
    console.error("API error:", error);
    throw error;
  }
}
```

The `try/catch` handles the rejected request. If I want an error boundary to display a fallback, I need to represent that failure in React state and throw it during rendering, or use a data-fetching framework that integrates errors with boundaries.

**3. How would you design a layered error-handling strategy?**

In a production application, I would use multiple layers rather than one global error boundary.

- **Application-level boundary:** Wrap the main application to prevent an uncaught rendering error from leaving the entire UI unusable.
- **Per-route boundaries:** Wrap important routes independently, so a failure on the dashboard does not necessarily break the entire application.
- **Component-level boundaries:** Use them around isolated, failure-prone sections, such as charts, third-party widgets, or complex data visualizations.
- **API and event error handling:** Handle network errors, validation failures, rejected promises, and user-action errors explicitly. Error boundaries are not a replacement for this logic.
- **Fallback UI:** Display a useful message, a retry option where appropriate, and navigation back to a safe screen.
- **Error reporting:** Send unexpected errors to Sentry with relevant context, while avoiding passwords, tokens, and other sensitive data.

For example, a dashboard might have a route-level boundary around the page and a separate boundary around its analytics chart. If the chart crashes, the rest of the dashboard can remain usable.

**4. How do you reset an error boundary when navigating?**

Once an error boundary enters its fallback state, it usually remains there until its state is reset or the boundary is remounted.

With React Router, I can use the current route location as a reset key:

```jsx
import { useLocation } from "react-router-dom";
import { ErrorBoundary } from "react-error-boundary";

function RouteErrorBoundary({ children }) {
  const location = useLocation();

  return (
    <ErrorBoundary
      key={location.pathname}
      FallbackComponent={RouteErrorFallback}
    >
      {children}
    </ErrorBoundary>
  );
}
```

When the pathname changes, React remounts the boundary and resets its state. This is a simple approach, although it also resets the boundary when moving between routes that share the same wrapper. In more complex applications, I would use a boundary library's `resetKeys` feature or the router's native error-boundary mechanism.

I would not reset the boundary blindly on every render because that could create repeated crashes and make the fallback impossible to use.

**5. How do you integrate Sentry?**

I would report unexpected errors from `componentDidCatch()` or use Sentry's React integration. Sentry can capture the exception, React component stack, release information, and other diagnostic context.

A typical setup would initialize Sentry once at application startup and configure the error boundary to report exceptions. I would also distinguish unexpected application errors from expected failures, such as invalid form input or a normal `404`, to avoid noisy reporting.

For production, I would configure source maps, release tracking, environment tags, and appropriate user context without sending sensitive information.

**6. Why are there still no hook-based error boundaries?**

Hooks such as `useState` and `useEffect` cannot independently create an error boundary because React's error-boundary behavior is tied to specific class lifecycle APIs and the way React handles errors in the component tree.

There is no built-in hook that directly replaces `getDerivedStateFromError()` and `componentDidCatch()` with equivalent boundary behavior.

However, I can use a library such as `react-error-boundary`, which provides a functional-component-friendly API for defining fallbacks, resetting errors, and handling caught exceptions. The underlying error boundary is still implemented using a class component or framework-level mechanism.

**In summary:** I use error boundaries for render-time failures, explicit `try/catch` and state handling for asynchronous operations and event handlers, route-level isolation to limit the impact of failures, and Sentry for production monitoring. The goal is not just to prevent crashes, but to keep unaffected parts of the application usable and make failures diagnosable.

*Q22. useReducer, state machines and complex state. When is useReducer better than several useState calls? How do you model a multi-step checkout or async flow as explicit states (idle, loading, success, error) to prevent impossible states, and how does this compare with XState?*

### 1. When is `useReducer` better than multiple `useState` calls?

I use `useState` for simple, independent state values, such as a modal's visibility, an input value, or a selected tab. However, when multiple state values are related and change together, I prefer `useReducer`.

`useReducer` centralizes state transitions in a reducer function. Instead of updating different state variables across multiple event handlers, I dispatch actions that describe what happened, and the reducer determines the next state.

For example, in a checkout flow, I might need to manage the current step, cart data, shipping information, payment status, and errors. Using multiple `useState` calls can make the logic difficult to maintain because one action may need to update several values consistently.

```jsx
import { useReducer } from "react";

const initialState = {
  quantity: 1,
  status: "idle",
  error: null,
};

function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return {
        ...state,
        quantity: state.quantity + 1,
      };

    case "SUBMIT":
      return {
        ...state,
        status: "loading",
        error: null,
      };

    case "SUCCESS":
      return {
        ...state,
        status: "success",
      };

    case "ERROR":
      return {
        ...state,
        status: "error",
        error: action.message,
      };

    case "RESET":
      return initialState;

    default:
      return state;
  }
}

function Checkout() {
  const [state, dispatch] = useReducer(reducer, initialState);

  async function handleSubmit() {
    dispatch({ type: "SUBMIT" });

    try {
      await submitOrder();
      dispatch({ type: "SUCCESS" });
    } catch (error) {
      dispatch({
        type: "ERROR",
        message: error.message,
      });
    }
  }

  return (
    <div>
      <p>Quantity: {state.quantity}</p>

      {state.status === "loading" && <p>Placing order...</p>}
      {state.status === "success" && <p>Order placed!</p>}
      {state.status === "error" && <p>{state.error}</p>}

      <button onClick={handleSubmit}>Place order</button>
    </div>
  );
}
```

Here, the reducer handles synchronous state transitions, while the event handler performs the asynchronous API request and dispatches the result.

The main advantages are predictable state transitions, centralized business logic, easier testing, and fewer inconsistent updates. However, I would not use `useReducer` for every component; for simple independent values, `useState` is usually clearer.

### 2. How do you model asynchronous states explicitly?

For an asynchronous operation, I model its status using a finite set of states, such as `idle`, `loading`, `success`, and `error`.

A common mistake is maintaining separate booleans:

```jsx
const [isLoading, setIsLoading] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [hasError, setHasError] = useState(false);
```

This can create impossible combinations, such as `isLoading === true` and `isSuccess === true` simultaneously.

Instead, I use a single status field:

```jsx
const initialState = {
  status: "idle",
  data: null,
  error: null,
};

function reducer(state, action) {
  switch (action.type) {
    case "FETCH_START":
      return {
        status: "loading",
        data: null,
        error: null,
      };

    case "FETCH_SUCCESS":
      return {
        status: "success",
        data: action.payload,
        error: null,
      };

    case "FETCH_ERROR":
      return {
        status: "error",
        data: null,
        error: action.payload,
      };

    case "RESET":
      return initialState;

    default:
      return state;
  }
}
```

Now the operation has one explicit status instead of several independent booleans. The UI can render according to that status.

For stronger guarantees, I would also validate transitions. For example, a `FETCH_SUCCESS` action should only be accepted while the request is loading. In TypeScript, I can go further by using a discriminated union so that a loading state cannot accidentally contain successful data, or an error state cannot exist without an error object.

### 3. How would you model a multi-step checkout?

For a checkout with shipping, payment, and order confirmation, I would treat each step as an explicit state and define the transitions allowed between them.

For example:

- `shipping`: The user enters and validates the shipping address.
- `payment`: The shipping step has been completed, and payment details are collected.
- `processing`: The order or payment request is being processed.
- `success`: The order has been confirmed.
- `error`: The current operation failed and may be retried.

The important point is that transitions should be controlled by business rules, not just by which button the user clicks.

```jsx
const initialState = {
  step: "shipping",
  shipping: null,
  payment: null,
  order: null,
  error: null,
};

function checkoutReducer(state, action) {
  switch (action.type) {
    case "SHIPPING_SUBMITTED":
      if (state.step !== "shipping") return state;

      return {
        ...state,
        shipping: action.payload,
        step: "payment",
        error: null,
      };

    case "PAYMENT_SUBMITTED":
      if (state.step !== "payment") return state;

      return {
        ...state,
        payment: action.payload,
        step: "processing",
        error: null,
      };

    case "ORDER_SUCCESS":
      if (state.step !== "processing") return state;

      return {
        ...state,
        order: action.payload,
        step: "success",
        error: null,
      };

    case "ORDER_FAILURE":
      if (state.step !== "processing") return state;

      return {
        ...state,
        step: "error",
        error: action.payload,
      };

    case "RETRY":
      if (state.step !== "error") return state;

      return {
        ...state,
        step: "processing",
        error: null,
      };

    default:
      return state;
  }
}
```

This approach makes the permitted transitions explicit and prevents actions from moving the checkout into arbitrary steps.

In a real checkout, I would also validate the shipping and payment data, prevent duplicate submissions, and handle payment idempotency on the server. A React reducer only controls client-side state; it cannot guarantee that a payment is processed exactly once or that the backend order is valid.

For more complex flows, I would consider separating the error state by operation, since a shipping validation error and a payment-processing failure require different recovery paths.

### 4. What is a state machine, and how is it different from `useReducer`?

A finite state machine models an application as a finite set of states and explicitly defined transitions between those states. A transition occurs when an event is received and its conditions are satisfied.

For example:

`idle → loading → success`

or:

`idle → loading → error → loading`

A reducer can implement a state machine, but `useReducer` itself is only a React state-management API. It does not automatically enforce a formal state-machine model. The developer must define valid states, transitions, and guards.

A state machine makes complex business workflows easier to reason about because it explicitly defines what can happen next, which events are valid, and what should happen when an event arrives in the wrong state.

### 5. How does this compare with XState?

XState is a state-machine and statechart library designed for workflows with more complex transition logic.

With `useReducer`, I manually implement the reducer, transition rules, and side-effect coordination. With XState, I can define states, events, guarded transitions, invoked asynchronous actors, and nested or parallel states in a structured machine.

For example, a checkout machine could define `shipping`, `payment`, `processing`, `success`, and `error` as states, with explicit transitions between them. XState can also invoke a payment service when entering `processing` and transition to `success` or `error` based on its result.

I would choose based on complexity:

- **`useState`:** Simple, mostly independent component state.
- **`useReducer`:** Related state values and moderately complex transitions within a component.
- **`useReducer` with a disciplined state-machine design:** Workflows where I need explicit states and transition rules but do not need a dedicated state-machine library.
- **XState:** Large workflows with many conditional transitions, nested or parallel states, cancellation, retries, or coordinated asynchronous actors.

**In summary:** I use `useReducer` when related state changes need to be centralized and predictable. I model asynchronous operations and multi-step workflows with explicit states rather than unrelated booleans, and I enforce valid transitions to avoid impossible states. When the workflow becomes large enough that manually managing transitions and side effects is difficult, I consider XState.

*Q23. Component patterns. Compare HOCs, render props, compound components and custom hooks. How would you build a reusable Modal, Tabs or Select with the compound component pattern and context? What are the typical prop-drilling and over-abstraction traps?*

### 1. Compare HOCs, Render Props, Compound Components, and Custom Hooks

In React, component patterns help us reuse logic, share state, and design flexible APIs. I choose a pattern based on whether I need to reuse behavior, share state between related components, or compose a complex UI.

**1. Higher-Order Components (HOCs)**

A HOC is a function that takes a component and returns an enhanced component. It was commonly used to share logic before hooks were introduced.

```jsx
function withLoading(Component) {
  return function EnhancedComponent({ isLoading, ...props }) {
    if (isLoading) return <p>Loading...</p>;

    return <Component {...props} />;
  };
}
```

I would use a HOC when integrating with an existing codebase that already uses this pattern or when a library provides component enhancement through HOCs.

The main drawbacks are wrapper nesting, harder debugging, and potential prop-name collisions. I also need to handle static properties and refs carefully when wrapping components.

**2. Render Props**

A render prop passes a function to a component, allowing the consumer to control how shared behavior is rendered.

```jsx
function MouseTracker({ render }) {
  const [position, setPosition] = React.useState({
    x: 0,
    y: 0,
  });

  return (
    <div
      onMouseMove={(event) =>
        setPosition({ x: event.clientX, y: event.clientY })
      }
    >
      {render(position)}
    </div>
  );
}

// Usage
<MouseTracker
  render={({ x, y }) => <p>{x}, {y}</p>}
/>
```

This pattern is useful when the same behavior needs different UI implementations. However, deeply nested render props can make JSX difficult to read, and changing function identities can affect memoization.

**3. Compound Components**

Compound components work together as a coordinated group. A parent component manages shared state, and child components consume it through context.

For example:

```jsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="profile">Profile content</Tabs.Content>
  <Tabs.Content value="settings">Settings content</Tabs.Content>
</Tabs>
```

This pattern is useful for tabs, modals, accordions, menus, and selects because it provides a flexible, readable API without passing every prop through every component.

Its main trade-off is that the components become coupled to a shared context and must be used within the correct parent.

**4. Custom Hooks**

Custom hooks extract reusable stateful logic into functions that can be shared across components.

```jsx
function useToggle(initialValue = false) {
  const [value, setValue] = React.useState(initialValue);

  const toggle = () => setValue((previous) => !previous);

  return { value, toggle, setValue };
}
```

A custom hook is ideal when I want to reuse behavior without sharing a particular UI structure. For example, `useDebounce`, `usePagination`, and `useFetch` can be reused by different components.

Unlike a HOC or compound component, a custom hook does not automatically share one state instance between all consumers. Each component calling the hook gets its own hook state unless the hook connects to a shared store or context.

**My approach:** I prefer custom hooks for reusable logic, compound components for coordinated UI, and HOCs or render props when they fit an existing API or a specific composition requirement.

### 2. How would you build a reusable Modal using the compound component pattern?

I would keep the modal's open state in the parent, expose it through context, and let child components such as `Modal.Trigger`, `Modal.Content`, and `Modal.Close` consume that shared state.

This allows the consumer to control the layout without passing `isOpen` and `setIsOpen` through multiple component levels.

```jsx
import React, { createContext, useContext, useState } from "react";

const ModalContext = createContext(null);

function useModalContext() {
  const context = useContext(ModalContext);

  if (!context) {
    throw new Error("Modal components must be used inside <Modal>");
  }

  return context;
}

function Modal({ children, defaultOpen = false }) {
  const [isOpen, setIsOpen] = useState(defaultOpen);

  return (
    <ModalContext.Provider value={{ isOpen, setIsOpen }}>
      {children}
    </ModalContext.Provider>
  );
}

function Trigger({ children }) {
  const { setIsOpen } = useModalContext();

  return (
    <button onClick={() => setIsOpen(true)}>
      {children}
    </button>
  );
}

function Content({ children }) {
  const { isOpen, setIsOpen } = useModalContext();

  if (!isOpen) return null;

  return (
    <div className="modal-backdrop">
      <div
        role="dialog"
        aria-modal="true"
        className="modal-content"
      >
        {children}
        <button onClick={() => setIsOpen(false)}>
          Close
        </button>
      </div>
    </div>
  );
}

Modal.Trigger = Trigger;
Modal.Content = Content;

export default Modal;
```

Usage:

```jsx
<Modal>
  <Modal.Trigger>Open settings</Modal.Trigger>

  <Modal.Content>
    <h2>Settings</h2>
    <p>Update your preferences here.</p>
  </Modal.Content>
</Modal>
```

The parent owns the state, and the children consume it through context. The consumer can arrange the trigger and content naturally without manually wiring state into every child.

This is a basic implementation. In production, I would also implement Escape-key handling, focus trapping, focus restoration, appropriate accessible labels, scroll locking, and portal rendering where necessary. I would also consider controlled usage through `open` and `onOpenChange` when the parent needs to manage the modal state.

### 3. How would you apply the same pattern to Tabs?

Tabs are another good example because the tab triggers and content panels must share the selected value.

The parent manages the active tab, while the child components read that value and update it through context.

```jsx
const TabsContext = React.createContext(null);

function useTabs() {
  const context = React.useContext(TabsContext);

  if (!context) {
    throw new Error("Tabs components must be used inside <Tabs>");
  }

  return context;
}

function Tabs({ children, defaultValue }) {
  const [activeTab, setActiveTab] = React.useState(defaultValue);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}

function List({ children }) {
  return <div role="tablist">{children}</div>;
}

function Trigger({ value, children }) {
  const { activeTab, setActiveTab } = useTabs();
  const selected = activeTab === value;

  return (
    <button
      type="button"
      role="tab"
      id={`tab-${value}`}
      aria-selected={selected}
      aria-controls={`panel-${value}`}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
}

function Content({ value, children }) {
  const { activeTab } = useTabs();

  if (activeTab !== value) return null;

  return (
    <div
      role="tabpanel"
      id={`panel-${value}`}
      aria-labelledby={`tab-${value}`}
    >
      {children}
    </div>
  );
}

Tabs.List = List;
Tabs.Trigger = Trigger;
Tabs.Content = Content;
```

Usage:

```jsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="profile">
    Profile information
  </Tabs.Content>

  <Tabs.Content value="settings">
    Account settings
  </Tabs.Content>
</Tabs>
```

This is the core pattern, not a complete accessible tabs implementation. For production, I would also implement arrow-key navigation, focus management, and a consistent strategy for hidden panels. I would support controlled state if consumers need to synchronize tabs with routing or external state.

For a complex `Select`, the same architecture can coordinate the selected option, open state, highlighted option, search query, and option list. However, keyboard navigation, focus behavior, screen-reader announcements, and positioning make a production select substantially more complex. I would usually build on a well-tested accessibility primitive rather than implement all of that from scratch.

### 4. What are prop drilling and over-abstraction traps?

**Prop drilling** occurs when props must pass through several intermediate components that do not use those values themselves.

For example, passing `selectedTab` from a page through a layout and several wrappers just to reach a deeply nested tab trigger can make the component tree difficult to maintain.

I would address this based on the actual problem:

- Lift state to the nearest common parent when only a few components need it.
- Use context when many related descendants need the same state.
- Use a state-management library when state must be shared broadly or requires more advanced coordination.
- Use custom hooks to reuse behavior, rather than introducing context unnecessarily.

Context is not automatically the right solution for every prop. Passing a simple `size` or `variant` prop explicitly can be clearer than creating a context for it.

**Over-abstraction** occurs when components, hooks, or configuration layers are introduced without providing meaningful reuse or flexibility.

For example, building a generic component framework for a single modal may add unnecessary indirection. On the other hand, a shared compound `Modal` makes sense if several parts of an application need different modal layouts with the same behavior.

I also watch for these issues:

- **Excessive context:** Broad context updates can cause many consumers to re-render. Split unrelated contexts and keep context values stable where useful.
- **Unclear APIs:** Too many optional props and implicit behaviors make components difficult to understand.
- **Premature generalization:** Do not build a universal `Select` before understanding the actual requirements.
- **Hidden coupling:** Compound children should clearly document that they require a parent provider.
- **Unnecessary wrappers:** Every abstraction should reduce duplication, improve consistency, or make the consumer API clearer.

### 5. How do I choose the right pattern in a real project?

I start with the problem rather than the pattern. If I only need to reuse logic, I create a custom hook. If a component needs to enhance another component consistently, a HOC may be appropriate. If consumers need control over rendering, render props can work well. If multiple UI pieces share state and form one logical component, I consider compound components with context.

**In summary:** Custom hooks are best for reusable behavior, compound components are useful for flexible coordinated UI, and HOCs and render props remain valid tools for particular composition needs. I avoid both prop drilling and unnecessary abstractions by choosing the simplest API that supports the real requirements of the application.

*Q24. Portals, focus management and accessibility. How do portals work, and how do events bubble through them? For a modal or dropdown, what must you handle (focus trap, Esc to close, aria- roles, restoring focus, scroll lock)? How do you test accessibility with axe and keyboard-only navigation?*

### 1. What are React Portals, and why do we use them?

A React portal allows me to render a component's DOM elements into a different DOM node while keeping that component in the same React tree.

I typically use portals for modals, dropdowns, tooltips, and popovers because these elements can otherwise be affected by parent containers with `overflow: hidden`, stacking contexts, or restrictive positioning.

For example, a modal rendered inside a container might get clipped because the container has `overflow: hidden`. By rendering the modal into a dedicated DOM node, such as `#modal-root`, I can avoid that particular DOM layout constraint.

```jsx
import { createPortal } from "react-dom";

function Modal({ children }) {
  const modalRoot = document.getElementById("modal-root");

  if (!modalRoot) return null;

  return createPortal(
    <div className="modal-backdrop">
      <div role="dialog" aria-modal="true">
        {children}
      </div>
    </div>,
    modalRoot
  );
}
```

In the HTML, I would have a separate portal target:

```html
<div id="root"></div>
<div id="modal-root"></div>
```

The important distinction is that the portal changes where the DOM is rendered, but it does not change the component's position in the React tree.

**How do events bubble through portals?**

React events from a portal bubble according to the React component tree, not the DOM tree.

For example, if a modal is rendered into `#modal-root` but is logically inside a React parent with an `onClick` handler, clicking inside the modal can trigger that parent's React handler.

This can surprise developers because the modal may be outside the parent in the actual DOM. If necessary, I handle the event explicitly, for example by stopping propagation on an appropriate boundary or by structuring the parent handler to ignore modal interactions.

Also, a portal does not automatically solve accessibility, focus management, or every stacking-context issue. It only changes the DOM placement.

### 2. What accessibility requirements must a modal handle?

A modal is not accessible simply because it has `role="dialog"`. I need to manage keyboard interaction, focus, accessible naming, and the relationship between the modal and the rest of the page.

**Focus management**

When the modal opens, focus should move into it, usually to the first appropriate interactive element or a specifically chosen element such as the dialog heading when that better supports the content.

While a modal dialog is open, keyboard focus should remain within it. This is commonly called a focus trap.

When the modal closes, focus should return to the element that opened it, provided that element still exists and is available.

**Escape key**

For a modal that supports keyboard dismissal, pressing `Escape` should close it. I would ensure the listener is cleaned up when the modal closes or unmounts.

For nested overlays, I would also define which overlay handles Escape so that one key press does not unintentionally close multiple layers.

**ARIA roles and labels**

For a modal dialog, I would use:

- `role="dialog"` to identify the dialog.
- `aria-modal="true"` to indicate that the dialog is modal.
- `aria-labelledby` to reference its visible title, or `aria-label` when an appropriate visible title is unavailable.
- `aria-describedby` when additional text meaningfully describes the dialog.

I would also ensure the background content is unavailable to interaction while the modal is open, typically using an appropriate `inert` strategy or an established dialog primitive. `aria-modal="true"` alone does not enforce focus trapping or prevent background interaction.

**Scroll locking**

When a modal opens, I would generally prevent the background page from scrolling and restore the previous scroll behavior when it closes. I would also consider mobile browser behavior, nested modals, and scrollbar layout shifts.

### 3. How would you implement Escape handling and focus restoration?

Here is a simplified example showing the core lifecycle behavior:

```jsx
import { useEffect, useRef } from "react";
import { createPortal } from "react-dom";

function Modal({ open, onClose, children }) {
  const dialogRef = useRef(null);
  const previousFocusRef = useRef(null);

  useEffect(() => {
    if (!open) return;

    previousFocusRef.current = document.activeElement;

    const dialog = dialogRef.current;

    // Move focus into the dialog.
    dialog?.focus();

    const handleKeyDown = (event) => {
      if (event.key === "Escape") {
        onClose();
      }
    };

    document.addEventListener("keydown", handleKeyDown);

    return () => {
      document.removeEventListener("keydown", handleKeyDown);

      // Restore focus to the element that opened the modal.
      const previousFocus = previousFocusRef.current;

      if (previousFocus?.isConnected) {
        previousFocus.focus();
      }
    };
  }, [open, onClose]);

  if (!open) return null;

  const modalRoot = document.getElementById("modal-root");
  if (!modalRoot) return null;

  return createPortal(
    <div className="modal-backdrop">
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
      >
        <h2 id="modal-title">Confirmation</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    modalRoot
  );
}
```

This demonstrates the basic pattern, but it is **not a complete production-ready modal**. A production implementation also needs a focus trap, background interaction management, scroll locking, and careful handling of unmounting and nested dialogs.

I would normally use a well-tested accessible dialog primitive rather than implementing every focus-management edge case from scratch. I would also keep `onClose` stable or manage the effect dependencies appropriately so the effect does not repeatedly restore and recapture focus during normal renders.

### 4. How do dropdowns differ from modals?

A dropdown or popover does not always need modal behavior. The correct interaction depends on what it represents.

For a simple dropdown menu, I would generally handle:

- Opening and closing through the trigger.
- Moving focus into the menu when appropriate.
- Arrow-key navigation between menu items.
- `Escape` to close and return focus to the trigger.
- Outside-click dismissal where appropriate.
- Correct roles, accessible names, and expanded state on the trigger.

For a menu button, I would typically use `aria-haspopup="menu"` and `aria-expanded`. The menu itself would use `role="menu"`, with items using the appropriate menu-item roles and keyboard behavior.

However, a list of ordinary navigation links is not automatically an ARIA menu. A searchable select or combobox also requires different semantics and keyboard behavior. I choose roles based on the widget's actual interaction model rather than adding ARIA attributes indiscriminately.

### 5. How do you test accessibility with axe?

I use automated accessibility testing to catch common issues such as missing accessible names, invalid ARIA relationships, and certain contrast or semantic problems.

For example, with `jest-axe` and React Testing Library:

```jsx
import { render } from "@testing-library/react";
import { axe, toHaveNoViolations } from "jest-axe";
import Modal from "./Modal";

expect.extend(toHaveNoViolations);

test("modal has no detectable accessibility violations", async () => {
  const { container } = render(
    <Modal open={true} onClose={() => {}}>
      <button>Confirm</button>
    </Modal>
  );

  const results = await axe(container);

  expect(results).toHaveNoViolations();
});
```

In a real project, I would ensure the test environment includes the portal target and that the assertion scans the rendered portal content as well. I would also test the page with the modal both open and closed.

Automated tools cannot detect every accessibility problem. A modal may pass an axe scan and still have broken keyboard navigation or incorrect focus restoration, so I combine automated tests with interaction tests.

### 6. How do you test keyboard-only navigation?

I manually test the component without using a mouse and add automated tests for critical keyboard behavior.

For a modal, I verify that:

1. I can open it using the keyboard.
2. Focus moves inside the dialog when it opens.
3. `Tab` and `Shift+Tab` keep focus within the modal.
4. `Escape` closes it when dismissal is supported.
5. Focus returns to the original trigger after closing.
6. Background controls cannot be interacted with while the modal is active.
7. Every interactive control has a visible focus indicator.

For a dropdown or menu, I test opening with the appropriate key, arrow-key navigation, selection, dismissal, and focus restoration.

I would automate these behaviors with React Testing Library and `user-event`, and use browser-level tests where necessary to validate real focus and scrolling behavior.

**In summary:** Portals solve DOM-placement problems, but accessibility requires a separate, deliberate implementation. For modals and dropdowns, I focus on correct semantics, predictable keyboard behavior, focus containment and restoration, background interaction, and scroll management. I use axe to detect common accessibility violations, but I also test the full interaction flow with keyboard-only navigation because automated scans alone are insufficient.

*Q25. Routing with React Router. How do nested routes, Outlet, loaders/actions and lazy route modules work? How do you implement protected routes, redirect after login, and preserve the intended URL? What causes unnecessary remounts when route layouts are structured badly?*

### 1. How do nested routes and `Outlet` work?

React Router allows me to map URLs to React components. With nested routes, I can create a shared layout for a group of pages instead of repeating the same header, sidebar, and navigation in every component.

The parent route renders the common layout, and the `Outlet` component renders whichever child route matches the current URL.

For example, in an admin dashboard, `/dashboard/users` and `/dashboard/settings` can share the same sidebar and header while displaying different page content.

```jsx
import {
  BrowserRouter,
  Routes,
  Route,
  Outlet,
} from "react-router-dom";

function DashboardLayout() {
  return (
    <div>
      <header>Admin Dashboard</header>

      <aside>Sidebar navigation</aside>

      <main>
        <Outlet />
      </main>
    </div>
  );
}

function Users() {
  return <h2>Users</h2>;
}

function Settings() {
  return <h2>Settings</h2>;
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/dashboard" element={<DashboardLayout />}>
          <Route path="users" element={<Users />} />
          <Route path="settings" element={<Settings />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

When the user visits `/dashboard/users`, React Router renders `DashboardLayout` and places `Users` inside its `Outlet`. When the user navigates to `/dashboard/settings`, the shared layout can remain mounted while the outlet content changes.

This improves code reuse, keeps the route hierarchy organized, and helps preserve shared layout state across navigation.

### 2. How do loaders and actions work?

In React Router's data-router APIs, a **loader** retrieves the data required by a route, while an **action** handles mutations such as submitting a form, creating a record, or updating a user.

A loader runs for a matching route when its data needs to be loaded or revalidated. React Router manages the associated navigation state and exposes the result through `useLoaderData()`.

An action handles a mutation submitted through mechanisms such as `<Form>` or `useSubmit()`. After a successful mutation, React Router can revalidate relevant loaders so the UI reflects the updated data.

```jsx
import {
  createBrowserRouter,
  RouterProvider,
  useLoaderData,
  Form,
  redirect,
} from "react-router-dom";

async function usersLoader() {
  const response = await fetch("/api/users");

  if (!response.ok) {
    throw new Response("Failed to load users", {
      status: response.status,
    });
  }

  return response.json();
}

async function createUserAction({ request }) {
  const formData = await request.formData();

  const response = await fetch("/api/users", {
    method: "POST",
    body: JSON.stringify({
      name: formData.get("name"),
    }),
    headers: {
      "Content-Type": "application/json",
    },
  });

  if (!response.ok) {
    throw new Response("Failed to create user", {
      status: response.status,
    });
  }

  return redirect("/users");
}

function UsersPage() {
  const users = useLoaderData();

  return (
    <>
      <h2>Users</h2>

      {users.map((user) => (
        <p key={user.id}>{user.name}</p>
      ))}

      <Form method="post">
        <input name="name" required />
        <button type="submit">Create user</button>
      </Form>
    </>
  );
}

const router = createBrowserRouter([
  {
    path: "/users",
    loader: usersLoader,
    action: createUserAction,
    element: <UsersPage />,
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

This illustrates the data-router approach: the loader provides the initial route data, and the action handles the form mutation.

In a production application, I would also handle validation errors, authentication, authorization, pending states, and error boundaries. I would use the router's mechanisms for displaying action and loader errors rather than assuming every request succeeds.

**Important distinction:** Loaders and actions are features of React Router's data APIs, such as `createBrowserRouter`. They are not automatically available in the same way when using only the basic `<BrowserRouter>` and `<Routes>` setup.

### 3. How do lazy route modules work?

Route-level code splitting allows me to load a page's code only when it is needed instead of including every route component in the initial JavaScript bundle.

React Router supports route-level `lazy` modules in data routers. The router can load the module containing the route's component and related route properties when that route is matched.

```jsx
import {
  createBrowserRouter,
  RouterProvider,
} from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/dashboard",
    lazy: () => import("./routes/dashboard"),
  },
  {
    path: "/settings",
    lazy: () => import("./routes/settings"),
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

For example, `routes/dashboard.jsx` can export the route's component and loader:

```jsx
export async function loader() {
  const response = await fetch("/api/dashboard");

  if (!response.ok) {
    throw new Response("Dashboard data failed to load", {
      status: response.status,
    });
  }

  return response.json();
}

export function Component() {
  return <h1>Dashboard</h1>;
}
```

The route module is dynamically imported when needed. The router can then use its exported route properties.

I use this approach for large dashboards, reporting pages, and other routes that do not need to be part of the initial bundle. For smaller applications, I avoid excessive code splitting because every additional chunk can introduce network overhead.

### 4. How do you implement protected routes?

A protected route restricts access to pages that require authentication. In a basic client-side implementation, I can check whether the user has an authenticated session and redirect unauthenticated users to the login page.

```jsx
import {
  Navigate,
  Outlet,
  useLocation,
} from "react-router-dom";

function ProtectedRoute() {
  const isAuthenticated = useAuthStatus();
  const location = useLocation();

  if (isAuthenticated === null) {
    return <p>Checking authentication...</p>;
  }

  if (!isAuthenticated) {
    return (
      <Navigate
        to="/login"
        replace
        state={{ from: location }}
      />
    );
  }

  return <Outlet />;
}
```

Here, `useAuthStatus()` represents an application-specific authentication hook. The `null` state means authentication is still being checked, so the application does not redirect prematurely.

I can protect a group of routes by nesting them under this component:

```jsx
<Routes>
  <Route path="/login" element={<Login />} />

  <Route element={<ProtectedRoute />}>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/users" element={<Users />} />
    <Route path="/settings" element={<Settings />} />
  </Route>
</Routes>
```

This is useful for client-side navigation, but it is not a security boundary. The backend must independently authenticate requests and enforce authorization for every protected API operation. Hiding a page does not prevent someone from calling its API directly.

For more complex applications using data routers, I can enforce authentication in route loaders or use a shared parent loader. However, parent and child loaders may run in parallel, so I would not rely on a parent loader alone to prevent every child loader from executing. Sensitive backend operations must remain protected on the server.

### 5. How do you redirect after login and preserve the intended URL?

Suppose a user tries to open `/settings/billing` without being authenticated. I want to redirect them to `/login`, then return them to `/settings/billing` after successful login.

I preserve the intended location in the router's navigation state:

```jsx
import {
  useLocation,
  useNavigate,
} from "react-router-dom";

function Login() {
  const location = useLocation();
  const navigate = useNavigate();

  const from = location.state?.from;

  async function handleLogin(credentials) {
    await loginUser(credentials);

    const destination =
      from?.pathname
        ? `${from.pathname}${from.search || ""}${from.hash || ""}`
        : "/dashboard";

    navigate(destination, { replace: true });
  }

  return (
    <button onClick={() => handleLogin({})}>
      Log in
    </button>
  );
}
```

The example focuses on the redirect logic; a real login form would pass actual credentials and handle loading, validation, and authentication errors.

Using `replace: true` replaces the login page in the browser history, so pressing Back does not immediately return the user to the login page.

There is also an important security consideration: if the destination comes from a query parameter or another externally controlled source, I validate it as a safe internal path before redirecting. Otherwise, an attacker might exploit an unvalidated redirect to send users to a malicious external URL.

### 6. What causes unnecessary remounts in React Router?

A remount occurs when React treats a component as a new component instance rather than an existing one. This resets its local state, reruns mount effects, and can trigger unnecessary data fetching.

Common causes include:

**1. Defining components inside another component**

```jsx
function App() {
  function DashboardLayout() {
    return <Outlet />;
  }

  return (
    <Routes>
      <Route
        path="/dashboard/*"
        element={<DashboardLayout />}
      />
    </Routes>
  );
}
```

`DashboardLayout` receives a new function identity whenever `App` renders. React can treat it as a different component type, causing the previous instance to unmount and the new one to mount.

I define route components at module scope instead.

**2. Changing keys unnecessarily**

If I write `key={location.pathname}` on a shared layout, navigating between paths can force React to remount the entire layout. That may reset form values, sidebar state, scroll positions, and other local state.

Keys should reflect a genuine need to reset a component, not be added indiscriminately.

**3. Replacing shared layouts across routes**

If routes use different parent layouts when they could share the same stable layout, navigation may destroy and recreate large parts of the component tree. Nested routes and a shared `Outlet` help preserve common UI.

**4. Conditional component types**

Switching between different wrapper component types around the same page can also reset the subtree. I keep the component structure stable where state should persist.

**How do I debug it?** I use React DevTools and the React Profiler to identify unexpected mounts, inspect component identities, and check whether route changes are resetting state or rerunning effects.

**In summary:** I use nested routes and `Outlet` for shared layouts, loaders and actions for route-level data operations, and lazy route modules for code splitting. For protected routes, I preserve the intended location and redirect after authentication while keeping authorization on the backend. Finally, I avoid unnecessary remounts by keeping component types and keys stable and structuring shared layouts correctly.

*Q26. Authentication in a SPA. Compare storing tokens in memory, localStorage and HttpOnly cookies, with the XSS and CSRF trade-offs of each. How do you implement silent refresh, handle multiple tabs logging out, and avoid a flash of the wrong UI while auth state is loading?*

### 1. Compare storing tokens in memory, `localStorage`, and `HttpOnly` cookies.

In a React SPA, authentication usually involves an access token, which authorizes API requests, and sometimes a refresh token, which obtains new access tokens when the current ones expire.

The main consideration is where credentials are stored, how they are sent to the backend, and what happens if an attacker executes JavaScript in the application.

**1. In-memory storage**

I can keep the access token in React state, a context, or another in-memory store.

- **Advantages:** The token disappears when the page is fully reloaded or the JavaScript context is destroyed. It is not automatically attached to requests to another origin.
- **Disadvantages:** The user may need to obtain a new token after a reload. An XSS attacker can still access an in-memory token while the application is running.
- **Use case:** Short-lived access tokens when the application can securely obtain a new token through a refresh mechanism.

**2. `localStorage`**

`localStorage` persists data across reloads and browser sessions.

- **Advantages:** Simple to implement and convenient for persisting client-side data.
- **Disadvantages:** Any JavaScript running in the origin can read its contents. If an attacker exploits XSS, they may steal a stored token and reuse it outside the browser.
- **Use case:** Non-sensitive preferences are generally appropriate. I avoid storing long-lived authentication credentials here when a safer architecture is available.

**3. `HttpOnly` cookies**

An `HttpOnly` cookie cannot be read directly by JavaScript, although the browser can send it with matching requests.

A secure session cookie commonly uses `HttpOnly`, `Secure`, and an appropriate `SameSite` policy.

- **Advantages:** JavaScript cannot directly extract the cookie value, reducing the risk of token theft through XSS.
- **Disadvantages:** Cookies are automatically attached to matching requests, which introduces CSRF considerations. XSS can still perform authenticated actions from the compromised page even if it cannot read the cookie.
- **Use case:** Browser-based applications that use a backend-managed session or a refresh-token cookie.

My preferred approach for a first-party SPA is often a **backend-managed session with a secure `HttpOnly` cookie**, or a short-lived access token held in memory with a carefully designed refresh mechanism. The exact choice depends on the backend architecture and deployment model.

### 2. What are the XSS and CSRF trade-offs?

**Cross-Site Scripting (XSS)** occurs when an attacker can execute malicious JavaScript in the application's origin. That script may access page data, make authenticated requests, or steal credentials accessible to JavaScript.

Keeping tokens in `localStorage` makes credential extraction particularly straightforward. In-memory storage reduces persistence but does not make the token inaccessible to a running XSS payload. `HttpOnly` cookies prevent direct JavaScript reads of the cookie, but they do not prevent an XSS payload from issuing authenticated requests.

I reduce XSS risk by avoiding unsafe HTML injection, sanitizing untrusted HTML when necessary, using contextual output escaping, applying a restrictive Content Security Policy (CSP), and keeping dependencies updated.

**Cross-Site Request Forgery (CSRF)** occurs when a browser is tricked into sending an authenticated request to a site where the user has an active session.

Cookies can be sent automatically, so cookie-based authentication requires appropriate CSRF defenses. Depending on the deployment, these can include `SameSite` cookies, CSRF tokens, and `Origin` or `Referer` validation for state-changing requests.

A bearer token stored in memory and attached explicitly through an `Authorization` header is less exposed to conventional CSRF attacks because another site generally cannot read the token and attach that header. However, XSS remains a major risk.

Also, `SameSite` is based on site boundaries, not simply origin boundaries, so it should not be treated as a complete defense in every cross-origin deployment.

### 3. How do you implement silent token refresh?

Silent refresh allows the application to obtain a new access token without forcing the user to log in again while their session remains valid.

A typical flow is:

1. The user logs in, and the backend establishes a session or issues the necessary credentials.
2. The SPA uses a short-lived access token for API requests, if that is the chosen architecture.
3. When the access token expires or a refresh is needed, the SPA calls a refresh endpoint.
4. The browser sends the refresh cookie automatically when cookie scope and credential settings permit it.
5. The backend validates the session or refresh token and issues a new access token.
6. The SPA updates its in-memory token and retries the original request once.

For a cookie-based design, I would keep the refresh credential in an `HttpOnly`, `Secure` cookie and avoid exposing it to application JavaScript.

A simplified example of refreshing an in-memory access token is:

```jsx
let accessToken = null;
let refreshPromise = null;

async function refreshAccessToken() {
  if (!refreshPromise) {
    refreshPromise = fetch("/api/auth/refresh", {
      method: "POST",
      credentials: "include",
    })
      .then(async (response) => {
        if (!response.ok) {
          throw new Error("Session expired");
        }

        const data = await response.json();
        accessToken = data.accessToken;

        return accessToken;
      })
      .finally(() => {
        refreshPromise = null;
      });
  }

  return refreshPromise;
}
```

Here, `credentials: "include"` is relevant when the API requires cookies across origins; same-origin requests generally use cookies by default. Cross-origin cookie authentication also requires appropriate CORS and cookie configuration.

The shared `refreshPromise` prevents multiple simultaneous requests in the same JavaScript context from starting duplicate refresh operations.

In production, I would also handle refresh failures, retry the original API request at most once, and clear authentication state if the session is no longer valid. The backend should implement refresh-token rotation or appropriate session controls where applicable, detect reuse when supported, and revoke sessions when necessary.

**Important:** I would not refresh on every `401` without checking the failure reason. A `401` can indicate an expired access token, invalid credentials, or another authentication failure. Blind retries can create loops.

### 4. How do you prevent concurrent refresh requests?

Imagine five API requests fail because the access token expires at approximately the same time. If each request starts a refresh independently, refresh-token rotation may cause race conditions or invalidate one of the requests.

I use a single-flight pattern: the first request starts the refresh, and the other requests await the same promise. Once the refresh succeeds, the waiting requests retry with the updated access token.

I also distinguish authentication failures from authorization failures. A `403 Forbidden` generally means the request is not permitted, so refreshing the token is not automatically the right response.

For applications with multiple tabs, the coordination problem is broader because each tab has a separate JavaScript context. A shared in-memory promise cannot coordinate refresh operations across tabs.

### 5. How do you handle logout across multiple tabs?

If the user logs out in one tab, I want other open tabs to update their authentication state rather than continuing to display protected UI.

For a token-based SPA, I can use the browser's `storage` event to broadcast a logout signal:

```jsx
function logout() {
  accessToken = null;

  // The backend should also invalidate the session
  // or revoke the refresh credential.
  localStorage.setItem("auth-event", JSON.stringify({
    type: "logout",
    timestamp: Date.now(),
  }));
}

window.addEventListener("storage", (event) => {
  if (!event.newValue) return;

  try {
    const message = JSON.parse(event.newValue);

    if (message.type === "logout") {
      accessToken = null;
      // Clear application auth state and sensitive caches.
    }
  } catch {
    // Ignore malformed messages.
  }
});
```

The `storage` event fires in other same-origin tabs, not in the tab that made the change. Therefore, the tab initiating logout must update its own state directly.

I would not rely on this browser event as the actual security mechanism. The backend must invalidate the session or refresh credential so a tab that misses the event cannot continue obtaining new access tokens.

For more advanced coordination, I can use `BroadcastChannel`, which provides direct messaging between same-origin tabs. I would still treat server-side session validity as authoritative.

### 6. How do you avoid showing the wrong UI while authentication is loading?

A common bug occurs when the application initially assumes the user is logged out before checking an existing session. It briefly renders the login page and then switches to the dashboard when authentication succeeds. This is often called an authentication flash.

I solve this by modeling authentication with three explicit states:

- `loading`: The application is checking the current session.
- `authenticated`: The user has a valid session.
- `unauthenticated`: No valid session exists.

I do not treat the initial state as unauthenticated until the session check completes.

```jsx
import {
  createContext,
  useContext,
  useEffect,
  useState,
} from "react";

const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [auth, setAuth] = useState({
    status: "loading",
    user: null,
  });

  useEffect(() => {
    let cancelled = false;

    async function initializeAuth() {
      try {
        const response = await fetch("/api/auth/me", {
          credentials: "include",
        });

        if (!response.ok) {
          if (response.status === 401) {
            if (!cancelled) {
              setAuth({
                status: "unauthenticated",
                user: null,
              });
            }
            return;
          }

          throw new Error("Unable to verify session");
        }

        const user = await response.json();

        if (!cancelled) {
          setAuth({
            status: "authenticated",
            user,
          });
        }
      } catch {
        if (!cancelled) {
          // Do not silently treat a network failure as logout.
          setAuth({
            status: "error",
            user: null,
          });
        }
      }
    }

    initializeAuth();

    return () => {
      cancelled = true;
    };
  }, []);

  return (
    <AuthContext.Provider value={{ auth, setAuth }}>
      {children}
    </AuthContext.Provider>
  );
}

function ProtectedRoute() {
  const { auth } = useContext(AuthContext);

  if (auth.status === "loading") {
    return <p>Checking session...</p>;
  }

  if (auth.status === "error") {
    return <p>Unable to verify your session. Please retry.</p>;
  }

  if (auth.status === "unauthenticated") {
    return <Navigate to="/login" replace />;
  }

  return <Outlet />;
}
```

This example omits the imports for `Navigate` and `Outlet` and assumes the provider is mounted above the protected route; I would include those imports in the actual implementation.

The key principle is that a network failure is not necessarily proof that the user is logged out. I distinguish a confirmed unauthenticated response from a temporary server or network failure and provide an appropriate retry path.

**In summary:** I choose credential storage based on the threat model, prefer secure `HttpOnly` cookies for refresh credentials when appropriate, and defend against both XSS and CSRF. I implement single-flight refresh to prevent request races, coordinate logout across tabs while keeping the backend authoritative, and use an explicit loading state to avoid rendering the wrong authentication UI before session initialization finishes.

*Q27. Testing React apps. What does React Testing Library encourage vs Enzyme-style testing? How do you mock the network with MSW instead of mocking fetch, test async UI with findBy/waitFor, test custom hooks, and decide what belongs in unit, integration and Playwright/Cypress E2E tests?*

### 1. What does React Testing Library encourage compared with Enzyme?

React Testing Library (RTL) encourages testing components from the user's perspective rather than testing their internal implementation.

With Enzyme-style testing, developers often inspect component instances, internal state, props, or implementation details. This can make tests brittle because refactoring a component may break the tests even when the user-facing behavior remains unchanged.

With RTL, I render the component, interact with it as a user would, and assert what appears on the screen.

For example, instead of checking whether a component's internal state is `true`, I check whether clicking a button actually displays the expected message.

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { useState } from "react";

function ToggleMessage() {
  const [visible, setVisible] = useState(false);

  return (
    <>
      <button onClick={() => setVisible((value) => !value)}>
        Toggle message
      </button>

      {visible && <p>Hello, user!</p>}
    </>
  );
}

test("displays the message when clicked", async () => {
  const user = userEvent.setup();

  render(<ToggleMessage />);

  await user.click(
    screen.getByRole("button", { name: /toggle message/i })
  );

  expect(
    screen.getByText("Hello, user!")
  ).toBeInTheDocument();
});
```

This test verifies observable behavior without depending on how the component stores its state.

I generally prefer queries in this order:

- `getByRole` with an accessible name for buttons, inputs, dialogs, and other semantic elements.
- `getByLabelText` for form controls.
- `getByText` when visible text is the most appropriate identifier.
- `getByTestId` when the element cannot be reliably identified through its user-facing semantics.

`getBy*` queries are synchronous and fail if the element is missing. `queryBy*` is useful for asserting that an element is absent. `findBy*` waits for an element to appear asynchronously.

**My rule:** I test what the user can observe and do, not React's internal implementation.

### 2. How do you mock network requests with MSW instead of mocking `fetch`?

MSW, or Mock Service Worker, intercepts network requests at the network boundary and returns controlled responses. Instead of replacing `fetch` inside every test, I let the application make its normal HTTP request and configure MSW to handle it.

This keeps the test closer to real application behavior. It also allows me to simulate success responses, server errors, delayed responses, and different API scenarios without changing the component implementation.

For example, using MSW 2.x in a Vitest or Jest environment:

```jsx
// src/test/server.js
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

export const server = setupServer(
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Dev" },
      { id: 2, name: "Alex" },
    ]);
  })
);
```

Then I configure the server in the test setup:

```jsx
// src/test/setup.js
import "@testing-library/jest-dom";
import { beforeAll, afterAll, afterEach } from "vitest";
import { server } from "./server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

For Jest, I would use Jest's equivalent lifecycle setup and import the same MSW server.

Now I can test a component that fetches users:

```jsx
import { render, screen } from "@testing-library/react";
import { useEffect, useState } from "react";

function Users() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(false);

  useEffect(() => {
    let cancelled = false;

    async function loadUsers() {
      try {
        const response = await fetch("/api/users");

        if (!response.ok) {
          throw new Error("Request failed");
        }

        const data = await response.json();

        if (!cancelled) setUsers(data);
      } catch {
        if (!cancelled) setError(true);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    loadUsers();

    return () => {
      cancelled = true;
    };
  }, []);

  if (loading) return <p>Loading users...</p>;
  if (error) return <p role="alert">Unable to load users</p>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

The test can verify the result of the real component logic:

```jsx
import { render, screen } from "@testing-library/react";
import { expect, test } from "vitest";
import { server } from "./test/server";
import { http, HttpResponse } from "msw";

test("renders users returned by the API", async () => {
  render(<Users />);

  expect(screen.getByText("Loading users...")).toBeInTheDocument();

  expect(
    await screen.findByText("Dev")
  ).toBeInTheDocument();

  expect(screen.getByText("Alex")).toBeInTheDocument();
});

test("shows an error when the API fails", async () => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json(
        { message: "Server error" },
        { status: 500 }
      );
    })
  );

  render(<Users />);

  expect(await screen.findByRole("alert")).toHaveTextContent(
    "Unable to load users"
  );
});
```

The examples assume the test environment resolves `/api/users` to the intended local request URL. In Node-based tests, I configure a suitable base URL or use an absolute API URL if required.

MSW is especially useful because I can override a handler for an individual test without rewriting application code or globally mocking `fetch`.

### 3. What is the difference between `findBy*`, `waitFor`, and `getBy*`?

These queries serve different purposes.

**`getBy*`** queries immediately. I use it when the element should already be present.

```jsx
expect(screen.getByRole("button", { name: "Save" }))
  .toBeInTheDocument();
```

**`findBy*`** returns a promise and retries until the element appears or the timeout expires. I use it for asynchronous UI changes, such as API results or a success notification.

```jsx
expect(await screen.findByText("User saved"))
  .toBeInTheDocument();
```

**`waitFor`** retries a callback until it stops throwing or times out. I use it when I need to wait for a more complex condition, especially an assertion about a mocked function.

```jsx
await waitFor(() => {
  expect(onSave).toHaveBeenCalledTimes(1);
});
```

I avoid wrapping every assertion in `waitFor`. It should only be used when the condition genuinely needs to be awaited. For an element that appears asynchronously, `findBy*` is usually simpler.

I also avoid putting interactions inside `waitFor`, because the callback can execute multiple times and accidentally repeat the interaction.

### 4. How do you test custom hooks?

For hooks that are tightly coupled to React state or effects, I use `renderHook` from React Testing Library. It lets me exercise the hook and inspect its returned values without building a complete UI component.

For example, consider a simple `useCounter` hook:

```jsx
import { useState } from "react";

function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);

  return {
    count,
    increment: () => setCount((value) => value + 1),
    decrement: () => setCount((value) => value - 1),
  };
}
```

Its test:

```jsx
import { renderHook, act } from "@testing-library/react";
import { expect, test } from "vitest";

test("increments the counter", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

I use `act()` when directly triggering updates outside RTL's built-in interaction helpers, so React can process the associated updates before assertions.

For hooks that depend on context, I provide a wrapper through `renderHook`'s `wrapper` option. For asynchronous hooks, I test loading, success, error, and cleanup behavior, using MSW when the hook performs network requests.

However, I do not automatically test every hook in isolation. If the hook exists only to support a component, an integration test through the component may provide more meaningful coverage.

### 5. How do you divide unit, integration, and end-to-end tests?

I choose the test level based on what I need to verify and how much of the real system should participate.

**Unit tests**

These focus on a small, isolated piece of logic.

Examples include:

- Reducers and state transitions.
- Data transformation functions.
- Validation rules.
- Formatting utilities.
- Pure business logic.

They are fast and useful for checking many edge cases. I avoid excessive mocking that makes a unit test verify only its own assumptions.

**Integration tests**

These verify that multiple parts of the application work together.

For React applications, RTL and MSW are a strong combination. I might test a form that validates user input, submits an API request, displays a loading state, and then shows a success or error message.

These tests give me confidence that components, hooks, and API-handling logic interact correctly without requiring a running production backend.

**End-to-end (E2E) tests**

I use Playwright or Cypress to test the application in a real browser, including routing, authentication flows, forms, and critical user journeys.

For example, an E2E test might:

1. Open the login page.
2. Sign in with a test account.
3. Navigate to the dashboard.
4. Create a user.
5. Verify that the new user appears in the list.

E2E tests can reveal issues involving browser behavior, route configuration, cookies, frontend-backend integration, and deployment configuration that unit tests cannot detect as reliably.

Playwright is particularly useful for cross-browser testing and isolated browser contexts. Cypress also provides a strong browser-based testing workflow. I choose based on the team's existing tooling and requirements.

### 6. How do you decide what deserves an E2E test?

I prioritize business-critical flows, security-sensitive behavior, and functionality where failures have a high impact.

For a SaaS dashboard, I would prioritize login and logout, permission enforcement, creating and editing important records, payment or checkout flows, and major navigation paths.

I would not duplicate every unit-test assertion in E2E tests because browser tests are generally slower and more expensive to maintain.

My testing strategy would be:

- Unit tests for detailed business logic and edge cases.
- Integration tests for component behavior, forms, API states, and user interactions.
- A smaller set of reliable E2E tests for critical journeys across the actual application.

I would also include accessibility checks and test important error states, not just the happy path.

**In summary:** I prefer React Testing Library because it tests observable behavior rather than implementation details. I use MSW to simulate realistic network responses, `findBy*` for asynchronous elements, and `waitFor` for conditions that require retrying. I use `renderHook` where isolated hook testing adds value, and distribute coverage across unit, integration, and E2E tests based on risk and the boundaries I need to verify.

*Q28. TypeScript with React. How do you type props with generics (a generic List<T> or Table<T>), discriminated unions for variant props, ComponentProps<'button'> for extending native elements, event types, forwardRef, and context with a non-null default? When is as casting a smell?*

### 1. How do you type React component props?

In React with TypeScript, I define an interface or type for component props so that TypeScript can validate what a component receives and what it returns.

For simple components, I use an interface or type alias depending on the design. Both work well for ordinary props.

```tsx
interface ButtonProps {
  label: string;
  disabled?: boolean;
  onClick: () => void;
}

function Button({ label, disabled = false, onClick }: ButtonProps) {
  return (
    <button disabled={disabled} onClick={onClick}>
      {label}
    </button>
  );
}
```

Here, `label` and `onClick` are required, while `disabled` is optional. TypeScript catches missing required props and incorrect value types at compile time.

For reusable components, I use generics when the component should work with different data types while preserving type safety.

### 2. How do you type a generic `List<T>` or `Table<T>`?

Suppose I need a reusable list component that can render users, products, orders, or any other data. Instead of using `any`, I define a generic type parameter `T`.

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  getKey: (item: T) => React.Key;
}

function List<T>({
  items,
  renderItem,
  getKey,
}: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={getKey(item)}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}
```

Usage:

```tsx
interface User {
  id: number;
  name: string;
}

const users: User[] = [
  { id: 1, name: "Dev" },
  { id: 2, name: "Alex" },
];

<List
  items={users}
  getKey={(user) => user.id}
  renderItem={(user) => <span>{user.name}</span>}
/>;
```

TypeScript infers `T` as `User`, so `user.name` and `user.id` are type-checked. If I access a property that does not exist, TypeScript reports an error.

For a reusable table, I can make the column definitions generic as well:

```tsx
interface Column<T> {
  key: string;
  header: string;
  render: (row: T) => React.ReactNode;
}

interface TableProps<T> {
  rows: T[];
  columns: Column<T>[];
  getRowKey: (row: T) => React.Key;
}

function Table<T>({
  rows,
  columns,
  getRowKey,
}: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((column) => (
            <th key={column.key}>{column.header}</th>
          ))}
        </tr>
      </thead>

      <tbody>
        {rows.map((row) => (
          <tr key={getRowKey(row)}>
            {columns.map((column) => (
              <td key={column.key}>
                {column.render(row)}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

The key advantage is that the row type flows through the table and its column renderers, so each column receives the correct data type.

In a production table, I might further constrain column keys using `keyof T` or use a typed accessor function. I would choose that additional complexity when it provides useful compile-time guarantees.

### 3. How do discriminated unions help with variant props?

Discriminated unions are useful when a component has multiple variants with different required props. They prevent invalid combinations that can occur when all props are made optional in one large interface.

For example, a notification component might have a link variant and a button variant.

```tsx
type ActionProps =
  | {
      variant: "link";
      href: string;
      onClick?: never;
    }
  | {
      variant: "button";
      onClick: () => void;
      href?: never;
    };

function Action(props: ActionProps) {
  if (props.variant === "link") {
    return <a href={props.href}>Open</a>;
  }

  return <button onClick={props.onClick}>Open</button>;
}
```

When `variant` is `"link"`, TypeScript requires `href`. When it is `"button"`, TypeScript requires `onClick`. The `never` properties also help reject incompatible combinations when the object is checked by TypeScript.

This is safer than defining `href?` and `onClick?` as unrelated optional properties because that design would permit a variant with neither property or potentially both.

I use discriminated unions for buttons, inputs, dialogs, and other components whose behavior and required props depend on a selected variant.

### 4. How do you extend native HTML element props?

If I build a reusable button, I usually want it to support standard HTML properties such as `type`, `disabled`, `title`, `aria-label`, and native event handlers.

Rather than manually declaring every possible property, I use `React.ComponentPropsWithoutRef<"button">` or `React.ComponentProps<"button">`, depending on whether the component handles refs separately.

```tsx
import type { ComponentPropsWithoutRef } from "react";

type ButtonProps = ComponentPropsWithoutRef<"button"> & {
  variant?: "primary" | "secondary";
};

function Button({
  variant = "primary",
  className,
  ...props
}: ButtonProps) {
  return (
    <button
      {...props}
      className={`button button-${variant} ${className ?? ""}`}
    />
  );
}
```

Now consumers can pass native props such as `disabled`, `type`, `onClick`, and `aria-label` without me redefining them.

I use `ComponentPropsWithoutRef<"button">` when the component does not expose a ref. If I want to forward a ref, I use an appropriate ref-aware prop type and implementation.

I also take care with prop spreading and precedence. For example, I should not accidentally override a consumer's `disabled` value or `onClick` handler without an intentional API decision.

### 5. How do you type React events?

I use React's event types based on the element and event being handled.

For example:

```tsx
function SearchInput() {
  const handleChange = (
    event: React.ChangeEvent<HTMLInputElement>
  ) => {
    console.log(event.currentTarget.value);
  };

  const handleSubmit = (
    event: React.FormEvent<HTMLFormElement>
  ) => {
    event.preventDefault();
  };

  const handleClick = (
    event: React.MouseEvent<HTMLButtonElement>
  ) => {
    console.log(event.currentTarget.disabled);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button type="submit" onClick={handleClick}>
        Search
      </button>
    </form>
  );
}
```

Common types include:

- `React.ChangeEvent<HTMLInputElement>` for input changes.
- `React.FormEvent<HTMLFormElement>` for form submission.
- `React.MouseEvent<HTMLButtonElement>` for mouse events.
- `React.KeyboardEvent<HTMLInputElement>` for keyboard events.
- `React.FocusEvent<HTMLInputElement>` for focus events.

TypeScript can often infer the event type directly from an inline JSX handler. I explicitly annotate it when extracting a handler into a separate function or when inference is unclear.

I also understand the distinction between `event.target` and `event.currentTarget`: `target` is the originating element, while `currentTarget` is the element whose handler is executing.

### 6. How do you use `forwardRef`?

Refs allow a parent component to access a DOM element or an imperative API exposed by a child. For reusable inputs and buttons, forwarding a ref is useful when the parent needs to focus the input, measure the element, or integrate with a form library.

In React 18 and earlier, `forwardRef` is the standard approach:

```tsx
import { forwardRef } from "react";

interface InputProps {
  label: string;
  placeholder?: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  function Input({ label, placeholder }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} placeholder={placeholder} />
      </label>
    );
  }
);
```

Usage:

```tsx
const inputRef = React.useRef<HTMLInputElement>(null);

<Input
  ref={inputRef}
  label="Email"
  placeholder="Enter email"
/>;
```

The generic arguments specify the referenced element type and the component's props.

**React 19 note:** React 19 allows function components to receive `ref` as a regular prop, so `forwardRef` is no longer necessary for new React 19-only components. However, it remains common in React 18 codebases and libraries that support earlier versions.

I expose refs only when there is a real need for DOM access or imperative behavior. For complex components, I may use `useImperativeHandle` to expose a small, deliberate API rather than the entire underlying DOM node.

### 7. How do you type context without an unsafe non-null default?

A common issue is creating context with `null` but then repeatedly checking for `null` in every consumer.

Instead of using a fake default object or casting away the nullability, I create a nullable context and expose a custom hook that throws a clear error when the provider is missing.

```tsx
import {
  createContext,
  useContext,
  useState,
  type ReactNode,
} from "react";

interface AuthContextValue {
  user: { id: string; name: string } | null;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<AuthContextValue["user"]>(null);

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);

  if (context === null) {
    throw new Error("useAuth must be used within AuthProvider");
  }

  return context;
}
```

Now components calling `useAuth()` receive a non-null `AuthContextValue`, and using the hook outside its provider fails immediately with a meaningful error.

I avoid initializing context with `{}` cast as the expected type because that hides a real configuration error and can lead to runtime failures.

### 8. When is `as` casting a smell?

TypeScript's `as` operator tells the compiler to treat a value as a particular type. It does not validate or convert the value at runtime.

For example:

```tsx
const user = apiResponse as User;
```

This may silence a compiler error even if the API response does not actually match the `User` type. If the server returns an unexpected shape, the application can still fail at runtime.

I consider repeated casting a warning sign when it is used to:

- Bypass type errors instead of fixing the underlying types.
- Force nullable values into non-null types.
- Treat unvalidated API responses as trusted domain objects.
- Make incompatible component props compile.
- Cast event handlers or refs to unrelated types.

For external data, I prefer runtime validation with a schema library such as Zod, or explicit validation logic, before treating the data as a trusted TypeScript type.

There are legitimate uses of assertions, such as narrowing a value when the compiler cannot infer a fact that has already been established. But I keep those assertions narrow and justified.

**In summary:** I use generics for reusable data-driven components, discriminated unions for valid variant combinations, native prop types to preserve HTML functionality, and explicit event and ref types for safer component APIs. For context, I use a custom hook that checks the provider rather than an unsafe cast. I treat `as` as a tool for specific cases, not as a way to silence TypeScript errors.

*Q29. Styling architecture. Compare CSS Modules, Tailwind and CSS-in-JS (styled-components/Emotion) in terms of runtime cost, SSR, theming and design tokens. How do you build a themeable, accessible design system with variants (e.g. cva), and avoid specificity wars and style leakage?*

In React, I choose a styling approach based on **runtime performance, maintainability, SSR requirements, theming, and the complexity of the design system**. The three common approaches are CSS Modules, Tailwind CSS, and CSS-in-JS libraries such as styled-components and Emotion.

### 1. CSS Modules vs Tailwind vs CSS-in-JS

**CSS Modules** scope class names locally to a component, preventing accidental style conflicts. They are a good choice when I want traditional CSS, reusable stylesheets, and minimal runtime overhead. The CSS is generally processed at build time, so there is no need to generate styles dynamically for every component render. For theming, I can use CSS custom properties and theme classes on a root element.

**Tailwind CSS** uses utility classes directly in JSX. Its build process detects the classes used in the application and generates the required CSS, so production stylesheets can remain small when unused utilities are removed. It avoids most naming and specificity problems because I compose predefined utility classes instead of creating many custom selectors. It is particularly effective for rapidly building consistent interfaces, although complex components can become difficult to read if their class lists are excessively long. I use component abstractions and tools such as `clsx` and `tailwind-merge` where appropriate.

**CSS-in-JS**, such as styled-components and Emotion, lets me define styles alongside JavaScript components. It supports dynamic styling based on props, theme objects, and component state. However, depending on the library and configuration, it can introduce runtime work for evaluating styles, generating class names, and managing style rules. SSR also requires correct style extraction or collection to prevent flashes of unstyled content and hydration issues. Some libraries offer build-time or zero-runtime options, so the actual cost depends on the implementation.

| Concern | CSS Modules | Tailwind CSS | CSS-in-JS |
|---|---|---|---|
| Runtime cost | Very low | Very low | Can have additional runtime overhead |
| Style isolation | Locally scoped classes | Utility classes; custom CSS still needs care | Usually generated or scoped styles |
| SSR | Straightforward with CSS bundling | Straightforward when CSS is built correctly | May require SSR-specific setup |
| Dynamic styling | CSS variables and classes | Conditional utility classes and CSS variables | Convenient prop-driven styles |
| Theming | CSS custom properties | CSS variables and theme configuration | Theme providers or CSS variables |
| Best fit | CSS-first applications | Consistent, utility-driven UI | Highly dynamic component styling |

My default for a new React application is often **Tailwind CSS with CSS variables and reusable component variants**, or CSS Modules when the team prefers conventional CSS. I choose CSS-in-JS when its dynamic styling or existing ecosystem provides enough value to justify its complexity.

### 2. Building a themeable design system

I separate the design system into three layers: **design tokens, reusable primitives, and higher-level components**.

**First, define design tokens.** Tokens represent the shared visual decisions of the application: colors, spacing, typography, border radii, shadows, and focus indicators. I prefer CSS custom properties because they work with CSS Modules, Tailwind, and CSS-in-JS and allow theme changes without regenerating every component's styles.

```css
:root {
  --color-primary: #2457d6;
  --color-text: #172033;
  --color-surface: #ffffff;
  --color-border: #d7dce5;
  --color-focus: #78a6ff;

  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-4: 1rem;

  --radius-control: 0.5rem;
}

[data-theme="dark"] {
  --color-primary: #8aafff;
  --color-text: #f1f4fa;
  --color-surface: #171c28;
  --color-border: #3b4353;
  --color-focus: #a8c5ff;
}
```

Components consume these semantic tokens rather than hardcoding colors throughout the application. This makes global design changes easier and helps keep the interface consistent.

**Second, build accessible primitives.** I create reusable components such as `Button`, `Input`, `Checkbox`, `Dialog`, and `Select`. Each component owns its appropriate behavior, accessibility requirements, and visual states. For example, a button should support keyboard interaction, expose its disabled state correctly, retain a visible focus indicator, and use a native `<button>` element rather than a clickable `<div>`.

**Third, expose intentional variants.** Instead of letting every consumer provide arbitrary CSS, I define a controlled API for size, visual intent, and state. This keeps the design system consistent while still supporting legitimate differences between components.

### 3. Using CVA for component variants

Class Variance Authority (`cva`) helps define a component's base classes, variants, and combinations of variants in one place. It works especially well with Tailwind.

```tsx
import { cva, type VariantProps } from "class-variance-authority";
import { clsx } from "clsx";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[var(--color-focus)] disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        primary: "bg-[var(--color-primary)] text-white",
        secondary:
          "border border-[var(--color-border)] bg-[var(--color-surface)] text-[var(--color-text)]",
        danger: "bg-red-600 text-white",
        ghost: "bg-transparent text-[var(--color-text)]",
      },
      size: {
        sm: "h-8 px-3 text-sm",
        md: "h-10 px-4 text-sm",
        lg: "h-12 px-6 text-base",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "md",
    },
  }
);

type ButtonProps =
  React.ComponentPropsWithoutRef<"button"> &
  VariantProps<typeof buttonVariants>;

function Button({
  variant,
  size,
  className,
  ...props
}: ButtonProps) {
  return (
    <button
      className={clsx(buttonVariants({ variant, size }), className)}
      {...props}
    />
  );
}
```

Now consumers can use a consistent API:

```tsx
<Button variant="primary" size="lg">
  Save changes
</Button>
```

CVA is a **class composition tool, not a styling engine**. It does not automatically solve accessibility, theming, or conflicting CSS classes. I also use `tailwind-merge` when consumers are allowed to override Tailwind utilities, because ordinary class concatenation does not guarantee that conflicting utility classes resolve as intended.

For complex combinations, CVA also supports compound variants. For example, a particular size and variant combination can receive additional classes without adding conditional logic throughout the component.

### 4. Accessibility is part of the component contract

I don't treat accessibility as an optional styling concern. Every design-system component should define its expected behavior and states.

For example:

- Buttons need visible keyboard focus, correct disabled behavior, and appropriate accessible names.
- Inputs need programmatically associated labels, useful validation messages, and appropriate `aria-invalid` or descriptive relationships when necessary.
- Dialogs need correct dialog semantics, accessible naming, sensible focus placement, keyboard handling, and focus restoration.
- Color combinations must maintain adequate contrast, and information must not depend on color alone.
- Hover styles should have appropriate keyboard-focus equivalents.

I validate these behaviors with keyboard testing, accessibility checks, and component tests rather than relying only on visual inspection.

### 5. Avoiding specificity wars and style leakage

I use a few consistent rules to prevent styles from becoming difficult to maintain.

**Keep selectors shallow.** Prefer a class or utility class over deeply nested selectors such as `.page .panel .content button`. Deep selectors create unnecessary dependencies on the DOM structure and make future overrides harder.

**Define ownership clearly.** A component should primarily style its own elements. I avoid broad global rules such as `button { ... }` or `.card div { ... }` when they could unexpectedly affect unrelated components. Global CSS should mainly contain resets, typography defaults, and token definitions.

**Use CSS Modules or predictable class conventions.** CSS Modules prevent accidental class-name collisions, while Tailwind encourages composition from a shared utility vocabulary. Neither eliminates every conflict, but both reduce the need for escalating selector specificity.

**Use cascade layers when appropriate.** CSS `@layer` lets a team establish an explicit precedence order for reset styles, third-party styles, components, and utilities. This is more predictable than repeatedly adding IDs or `!important` to defeat existing declarations.

**Keep override mechanisms intentional.** I avoid making every component support arbitrary style overrides. I expose documented variants and use `className` overrides only where there is a real need. If a component allows overrides, I document how conflicts are resolved.

**Treat third-party CSS as an integration boundary.** I scope or isolate it where practical, avoid relying on its internal selectors, and ensure its stylesheet ordering is predictable.

### 6. SSR, hydration, and theming considerations

For server-rendered React applications, I ensure that the server and client produce compatible markup and styling.

With CSS Modules and Tailwind, styles are generally generated during the build and included through the application's CSS pipeline. With runtime CSS-in-JS, I follow the library's SSR integration so styles are collected and emitted with the server-rendered HTML. Incorrect configuration can cause missing styles, flashes of unstyled content, or hydration-related problems.

For theme switching, I prefer semantic CSS variables on the document root or an application container. The selected theme should be applied early enough to avoid a flash of the wrong theme. If the initial theme depends on a cookie or user preference, the server and client should agree on that initial value wherever possible.

I also avoid generating different initial class names or markup from nondeterministic values, because server and client rendering must remain consistent.

### Interview conclusion

My approach is to keep styling predictable and design decisions centralized. I use CSS Modules when conventional CSS and local isolation are priorities, Tailwind when utility-driven development and consistent visual composition are priorities, and CSS-in-JS when dynamic styling justifies its runtime and SSR considerations. For a scalable design system, I combine semantic design tokens, accessible reusable primitives, and controlled variants through tools such as CVA. I prevent CSS conflicts through shallow selectors, clear ownership, predictable cascade rules, and limited override mechanisms rather than relying on increasingly specific selectors or `!important`.

*Q30. React 19 and modern data flow. What do Actions, useActionState, useOptimistic, useFormStatus and the use hook change about forms and async data? How does Suspense for data fetching coordinate loading states, and how does useSyncExternalStore prevent tearing with external stores?*

React 19 improves how React handles asynchronous operations, form submissions, optimistic updates, and reading asynchronous or external data. The main idea is to let React coordinate pending states, errors, and UI updates instead of manually managing every request with `useState` and event handlers.

### 1. Actions in React 19

An **Action** is a function used to perform an operation that may involve asynchronous work, such as submitting a form, updating a profile, or saving a record to an API.

React 19 supports passing functions to a form's `action` prop. React handles the submission lifecycle more naturally, including pending states when the Action is used with supported form and transition mechanisms.

```tsx
async function updateProfile(formData: FormData) {
  "use server"; // Only in frameworks that support Server Functions.

  const name = formData.get("name");
  await saveProfile({ name });
}

function ProfileForm() {
  return (
    <form action={updateProfile}>
      <input name="name" required />
      <button type="submit">Save profile</button>
    </form>
  );
}
```

The `"use server"` directive is framework-specific and is relevant when defining a Server Function. A regular client-side Action does not need this directive.

Compared with a traditional form handler, Actions can reduce boilerplate around submission, pending UI, and form-data extraction. They do not automatically replace API validation, authorization, error handling, or database transactions. Those responsibilities remain part of the application architecture.

### 2. `useActionState`: managing submission results

`useActionState` lets me associate an Action with a piece of state representing its result. It returns three values:

- The current state, usually the latest result returned by the Action.
- A wrapped Action function that I can pass to a form.
- A pending flag indicating whether that Action is processing.

It is useful for validation errors, success messages, and server responses from form submissions.

```tsx
import { useActionState } from "react";

type FormState = {
  message: string;
};

async function saveName(
  _previousState: FormState,
  formData: FormData
): Promise<FormState> {
  const name = String(formData.get("name") ?? "").trim();

  if (!name) {
    return { message: "Name is required." };
  }

  await saveNameToApi(name);

  return { message: "Name saved successfully." };
}

function NameForm() {
  const [state, formAction, isPending] = useActionState(
    saveName,
    { message: "" }
  );

  return (
    <form action={formAction}>
      <input name="name" required />

      <button disabled={isPending}>
        {isPending ? "Saving..." : "Save"}
      </button>

      <p aria-live="polite">{state.message}</p>
    </form>
  );
}
```

Here, I don't need a separate `useState` for the message and another one for the loading flag. React coordinates the Action state and pending status.

In production, I would still handle rejected requests appropriately and return structured validation results rather than relying only on thrown errors. The backend must validate input independently.

### 3. `useOptimistic`: immediate UI feedback

`useOptimistic` lets the UI display an anticipated state while an asynchronous Action is running. Instead of waiting for the server to confirm a change, I update the visible interface optimistically.

Consider adding a comment. The comment can appear immediately while the request is in progress.

```tsx
import { useOptimistic, useState } from "react";

type Comment = {
  id: string;
  text: string;
};

function Comments({ initialComments }: {
  initialComments: Comment[];
}) {
  const [comments, setComments] = useState(initialComments);

  const [optimisticComments, addOptimisticComment] =
    useOptimistic(
      comments,
      (currentComments, newComment: Comment) => [
        ...currentComments,
        newComment,
      ]
    );

  async function submitComment(formData: FormData) {
    const text = String(formData.get("comment") ?? "").trim();
    if (!text) return;

    const optimisticComment = {
      id: crypto.randomUUID(),
      text,
    };

    addOptimisticComment(optimisticComment);

    const savedComment = await createComment(text);

    setComments((current) => [...current, savedComment]);
  }

  return (
    <>
      <form action={submitComment}>
        <input name="comment" required />
        <button type="submit">Post comment</button>
      </form>

      <ul>
        {optimisticComments.map((comment) => (
          <li key={comment.id}>{comment.text}</li>
        ))}
      </ul>
    </>
  );
}
```

This illustrates the concept; a production implementation should also handle request failures and avoid duplicating a comment when reconciling the optimistic item with the server response.

The important distinction is that **optimistic state is temporary UI state, not confirmed server state**. When the Action finishes, React recalculates the optimistic view from the latest real state. If the request fails, the application should show an error and let the optimistic change disappear or provide an explicit retry mechanism.

I use this pattern for comments, likes, toggles, and lightweight updates where immediate feedback improves the experience. For payments, inventory deductions, and other high-impact operations, I never treat the optimistic UI as proof that the operation succeeded.

### 4. `useFormStatus`: submission state in child components

`useFormStatus` lets a component read the status of its nearest parent form without passing a pending prop through every intermediate component.

It is especially useful when submit buttons are reusable design-system components.

```tsx
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

function ContactForm() {
  async function submitForm(formData: FormData) {
    await sendContactRequest(formData);
  }

  return (
    <form action={submitForm}>
      <input name="email" type="email" required />
      <SubmitButton />
    </form>
  );
}
```

A critical detail is that `SubmitButton` must be rendered **inside the form** whose status it needs to read. Calling `useFormStatus` in the same component that creates the form does not make it read that form's status, because the hook reads the nearest parent form.

`useFormStatus` reports form submission information, including `pending`, `data`, `method`, and `action`. It does not replace application-level state management for unrelated requests.

### 5. The `use` API: reading Promises and context

React's `use` API lets a component read a resource during rendering. Two important use cases are reading a Promise and reading context.

For example, a component can read a Promise that resolves to user data:

```tsx
import { Suspense, use } from "react";

type User = {
  name: string;
};

function UserDetails({ userPromise }: {
  userPromise: Promise<User>;
}) {
  const user = use(userPromise);

  return <h2>{user.name}</h2>;
}

function UserPage({ userPromise }: {
  userPromise: Promise<User>;
}) {
  return (
    <Suspense fallback={<p>Loading user...</p>}>
      <UserDetails userPromise={userPromise} />
    </Suspense>
  );
}
```

When the Promise is pending, React can suspend rendering and show the nearest Suspense fallback. When it resolves, React retries rendering with the result. If it rejects, the error must be handled through an appropriate error boundary or another supported error-handling mechanism.

An important practical rule is to avoid creating a new Promise on every render. The Promise should come from a stable source, such as a framework-supported data layer or a cached resource. Otherwise, repeated renders may continually suspend.

Unlike ordinary hooks, `use` can be called conditionally and inside loops, subject to React's rules for using it. It can also read context, but context must still be read in a valid component or custom-hook context.

### 6. How Suspense coordinates data loading

Suspense is a rendering coordination mechanism. It does not automatically fetch data simply because a component is wrapped in `<Suspense>`.

A supported data source must suspend rendering—for example, by reading a Promise with `use`, using a Suspense-enabled framework data API, or loading a lazy component with `React.lazy`.

```tsx
function Dashboard() {
  return (
    <main>
      <h1>Dashboard</h1>

      <Suspense fallback={<p>Loading analytics...</p>}>
        <Analytics />
      </Suspense>

      <Suspense fallback={<p>Loading activity...</p>}>
        <RecentActivity />
      </Suspense>
    </main>
  );
}
```

If `Analytics` suspends, React can show its fallback without necessarily replacing the entire dashboard. `RecentActivity` has its own boundary and can resolve independently.

This creates **independent loading regions**. I place boundaries around meaningful sections rather than wrapping the whole application in one large fallback. That way, a slow widget does not unnecessarily hide content that is already available.

Suspense also integrates with React's rendering and transition mechanisms. During a transition, React can keep already revealed content visible while preparing an update, depending on the boundary and update pattern.

For server rendering, Suspense can also support streaming HTML in frameworks that integrate with React's server-rendering APIs. The framework determines how data loading, streaming, and hydration are orchestrated.

For ordinary client-side fetching, `useEffect` plus `useState` does not automatically activate Suspense. I use a Suspense-compatible data source or framework integration when I want loading to be coordinated by Suspense.

### 7. `useSyncExternalStore`: consistent reads from external stores

React applications sometimes consume state from outside React's own state system—for example, a custom store, browser API, or a state-management library.

A naive implementation might subscribe in `useEffect`, call `setState` when updates arrive, and read the store during render. Under concurrent rendering, the store could change between reads. Different parts of the UI might then render values from different versions of the same state. This inconsistency is called **tearing**.

`useSyncExternalStore` provides a standard subscription interface that allows React to check external snapshots and keep rendering consistent.

```tsx
import { useSyncExternalStore } from "react";

type StoreState = {
  count: number;
};

let state: StoreState = { count: 0 };
const listeners = new Set<() => void>();

function subscribe(callback: () => void) {
  listeners.add(callback);

  return () => {
    listeners.delete(callback);
  };
}

function getSnapshot() {
  return state;
}

function increment() {
  state = { count: state.count + 1 };
  listeners.forEach((listener) => listener());
}

function Counter() {
  const snapshot = useSyncExternalStore(
    subscribe,
    getSnapshot
  );

  return (
    <div>
      <p>Count: {snapshot.count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

React subscribes to changes and reads the current snapshot. It can recheck that snapshot around rendering and commit, and coordinate updates when the external state changes during a concurrent render. This helps prevent a committed UI from combining inconsistent versions of the store.

There are two important implementation requirements:

- `subscribe` must register a listener and return a cleanup function.
- `getSnapshot` must return a stable snapshot while the underlying state has not changed. Returning a freshly created object on every call can cause repeated renders or warnings.

For server rendering, I can provide a third argument, `getServerSnapshot`, which returns the snapshot used during server rendering and initial hydration. That snapshot must be consistent with the data used to produce the server-rendered HTML.

I would use this hook when integrating a genuinely external store or browser subscription. For state owned entirely by a React component, `useState` or `useReducer` is usually simpler.

### 8. Putting the APIs together

Consider a product-editing form in an ERP application:

1. An Action submits the updated product to the API.
2. `useActionState` manages the result, such as validation errors or a success message.
3. `useFormStatus` disables the submit button and displays a pending label.
4. `useOptimistic` can immediately update the visible product name while the request is running.
5. A Suspense-enabled data source can coordinate loading of product details or related dashboard sections.
6. `useSyncExternalStore` can safely subscribe to an external real-time store when integrating updates from a WebSocket-backed state layer.

These APIs solve different problems; they are complementary rather than interchangeable.

### Interview conclusion

React 19 makes asynchronous UI flows more declarative. Actions provide a structured way to perform mutations, `useActionState` manages action results, `useOptimistic` improves perceived responsiveness, and `useFormStatus` exposes submission state to nested form components. The `use` API lets React read Promises and context during rendering, while Suspense coordinates the loading experience for compatible resources. Finally, `useSyncExternalStore` provides a concurrency-safe integration point for external state sources and helps prevent tearing.

The key is to use each API for its intended purpose: **Actions for mutations, Suspense for supported rendering-time dependencies, and `useSyncExternalStore` for external subscriptions**—while retaining proper error handling, server-side validation, and clear ownership of application state.

