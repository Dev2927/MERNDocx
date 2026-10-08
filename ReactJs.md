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

