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

Introduction

The key prop is a special attribute you need to include when creating lists of elements in React. It is crucial for helping React identify which items have changed, been added, or removed, thereby optimizing the rendering process.

Why key is important
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

Rules for keys

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

If you cram an entire app's state into a single context, every change to any slice re-renders every consumer. Split it into smaller, focused providers — for example, separate AuthContext, ThemeContext, and CartContext — so that a cart update doesn't re-render every theme consumer. You can also split read and write APIs into separate contexts so components that only need to dispatch don't re-render when state changes.

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