*Q1. what is the difference between old and new react native architecture*

The Old Architecture: The Bridge Model
In the old architecture, React Native works with two separate worlds:

- JavaScript World — Where the app logic runs
- Native World — Where the UI and system functions work

Since these two worlds use different languages (JavaScript vs Swift/Java/Kotlin), they need a way to communicate. This is where The Bridge comes in.

How the Bridge Works?

- The JavaScript side sends JSON messages through the Bridge to tell the native side what to do.
- The native side reads these messages and executes them (e.g., renders buttons, updates text).
- If a user interacts with the UI (like tapping a button), the native side sends a message back to JavaScript through the Bridge.

🔹 Example: If you create a login screen in React Native with an input field and a button, this is what happens:

1. JavaScript sends a message via the Bridge:

“Create an input field with these styles”
“Create a button with these properties”

2. The Native side receives the message and renders the UI.

3. When the user enters text, the Native side sends data back to JavaScript through the Bridge.

Threads in React Native (Old Architecture)
React Native apps run on multiple threads:

1. JavaScript Thread — Executes all React Native logic and handles UI updates.
2. Native Thread — Runs platform-specific code (Swift, Java/Kotlin) for UI elements.
3. Shadow Thread — Processes layout using Yoga Engine (translates CSS styles into native layout).

Disadvantages of the Old Architecture

1. Performance Issues
Communication between JavaScript and Native is slow because messages are passed asynchronously through the Bridge.
Scrolling may lag if UI updates arrive too late.

2. No Shared Memory
JavaScript and Native don’t share memory.
Instead of referencing a data object, React Native copies it in JSON format, which slows things down.

3. Heavy Native Module Loading
React Native loads all native modules at startup, even if some aren’t needed, which increases load time.

4. Harder to Contribute
The React Native codebase is complex, making it difficult for developers to contribute fixes or improvements.

The New Architecture: Fabric & TurboModules

To solve these issues, React Native introduced a New Architecture with Fabric and TurboModules.

What is Fabric?
Fabric removes the Bridge and introduces a faster, more efficient communication model between JavaScript and Native.

How is Fabric Better?

Uses synchronous communication (not async messages like before).
Allows JavaScript and Native to share memory, reducing the need to copy data.
Improves UI responsiveness and smoothness.

What are TurboModules?

TurboModules allow React Native to load only the native modules that are needed, instead of loading everything at startup.

How are TurboModules Better?

- Reduces app startup time.
- Saves memory by loading modules on demand.

*Q2. What Happens When You Click on a React Native App Icon?*

1. App Launches & UI Manager Starts

When you tap the app icon, the operating system launches the app and starts React Native.
The UI Manager initializes all native components like text fields, buttons, images, etc.

2. JavaScript Bundle Loads

React Native loads MainBundle.js, which contains all JavaScript code for the app.
If you check the terminal while the app is starting, you’ll see logs showing “Loading MainBundle.js”.

3. Views are Created via the Bridge (Old Architecture) or Fabric (New Architecture)

Old Architecture: JavaScript sends JSON messages to the Bridge to create UI elements.
New Architecture: Fabric directly interacts with native code, making UI rendering faster.

4. Layout Calculation (Using Yoga Engine)

React Native doesn’t use standard CSS like the web.
Instead, it uses Yoga Engine to calculate layouts and positions of elements.
The Shadow Thread handles this layout processing.

5. UI Appears on Screen

Finally, the UI Manager displays the UI based on the calculated layout.
You see the app fully loaded and ready to use! 🎉

What Happens When an App Launches?

UI Manager initializes native components.
JavaScript loads MainBundle.js.
Layout is calculated using Yoga Engine.
UI appears on screen.

What is in MainBundle.js?

In a React Native application, MainBundle.js (or index.bundle on Android) is the JavaScript bundle that contains the entire app’s JavaScript code.

It is generated when the app is built and includes:

1. All JavaScript Code — The entire React Native application logic.
2. React & React Native Core APIs — The fundamental libraries needed for rendering.
3. Metro Bundler Transformations — The optimized and minified JavaScript code.
4. Third-Party Libraries — Any external libraries you have installed (like Axios, Redux, etc.).
5. Native Bridge Calls (if using Old Architecture) — The code that communicates with native components.

How is MainBundle.js Created?

When you run:

npx react-native run-android

React Native’s Metro Bundler compiles and bundles all your JavaScript files into a single minified file, MainBundle.js.

Metro Bundler’s Role
Metro Bundler does the following:

- Compiles ES6+ JavaScript into compatible ES5 syntax.
- Combines multiple JavaScript files into one bundle.
- Minifies the code (for production).
- Handles Hot Reloading during development.

*Q3. What is React Native and how does it differ from React?*

React Native extends the award-winning React library, making it possible to build native mobile applications using familiar web technologies.

Differences from React

- Platform Scope: React is tailored for web development, while React Native is exclusive to building iOS and Android applications.

- Rendering Engine: React uses the browser's DOM for visualization, whereas React Native achieves a parallel outcome through native platform rendering.

- Component Style: While most of the component-building strategies and lifecycles between React and React Native are analogous, the controls manage the considerable difference in rendering and event handling. For instance, React uses simple buttons and divs, whereas React Native leverages platform-compliant components like Button, View, and Text.

- Integration with APIs: While React targets web APIs, React Native consolidates connectivity with native mobile device features and APIs. This extension makes it feasible to tap into mechanism such as Camera, GPS, and Fingerprint sensors.

*Q4. What are components in React Native?*

In React Native, components are building blocks that encapsulate UI and logic, making app development modular and efficient. There are two types of components: Base Components and Custom Components.

# Base Components

These are core UI elements provided by React Native, directly corresponding to native views or controls. They are optimized for performance and interactive consistency.

- Text: Displays readable text.
- View: A container that supports layout with styles, such as flexbox.
- Image: Displays images.

# Custom Components

These are created by developers and can be composed of both base and custom components, offering a higher level of abstraction. Custom components are reusable, promote a consistent design, and streamline UI updates.

Text_Example.jsx

Here is the React Native code:

import React from 'react';
import { Text } from 'react-native';

const CustomText = ({ children }) => (
  <Text style={{ fontFamily: 'Roboto-Bold', color: 'darkslategray' }}>{children}</Text>
);

export default CustomText;

# Component Nesting and Tree Structure

React Native applications are tree-structured with multiple components nested within one another. This composition allows for consistent and quick alterations across the app. Whether it's a Text, View, or custom component, each is a node in the component tree, visually impacting the app. Devs split the UI into smaller, self-contained parts to simplify maintenance and testing.

# Core Principles of Building Components

1. Reusability: Both base and custom components are designed for reuse in different parts of the application, further expanding the idea of modular development.

2. Autonomy: Each component should be self-sufficient, not heavily reliant on external data or functionality. This promotes easier maintenance and testing.

3. UI Focus: Components should either cater to UI or some specific functionality, but never both. This separation ensures a better code structure and maintainability.

4. Loose Prop Types: Custom components should generally avoid having too many mandatory props to allow for flexibility in their usage. They can, instead, rely on sensible defaults.

5. Integrative Mindset: When designing components, developers must have a holistic approach, keeping in mind how everything will come together in the UI.

Managing Component State

State management in components revolves around keeping track of changing data within that component. It's common in interactive UIs and involves data binding and conditional rendering. In React Native, components invoke a useState hook to integrate reactive state management.

*Q5. Explain the purpose of the render() function in a React Native component?*

The render() function, which is mandatory for all React and React Native components, is a gateway for JSX, receiving, processing, and returning the JSX layout. This function is like a workbench where the developer prepares the visual representation.

# JSX: Visual Blueprint

JSX is HTML-like markup within JavaScript that provides a structured description of the visual layout. It's like a visual blueprint for the component.

The render() function leverages this blueprint, converting the JSX elements into the actual visual UI components.

Here's a simple example:

// JSX Blueprint
let myJSX = (
  <View>
    <Text>Hello, World!</Text>
  </View>
);

// 'render()' Function
let render = () => {
  let uiComponent = (
    <View>
      <Text>Hello, World!</Text>
    </View>
  );

  // Visual representation
  return uiComponent;
};

# Code Maintenance

Having the render() function separates declarative structure from actual evaluatives and provides a clear workflow, making the code easier to maintain and understand.

# Virtual DOM Interaction

React Native employs a virtual DOM to optimize and streamline UI updates. When the state or props of a component change, render() is called to ensure the virtual DOM is in sync. The virtual DOM then identifies and applies only the necessary updates to the actual UI, reducing redundancy and rendering time.

# Performance Optimization: Conditional Rendering

Conditional rendering, controlled by if-else, is facilitated within the render() method, allowing for context-aware UI updates that ensure sensible resource and display utilization.

# Side-Effects Handling: Lifecycle Methods

The render() method is just one of several lifecycle methods. Accurate handling of these methods through render() and controlled component updates ensures proper data-fetching and side-effect management.

# UI Interactivity: Integrating JSX with Methods

JSX elements link visual representation with the logic behind user interactions—this is powered by methods like onPress, which, again, correspond to changes in state or prop triggers, leading back to, you guessed it, the trustworthy render() function.

*Q6. What is JSX and how is it used in React Native?*

JSX is a syntax extension for JavaScript, especially popular in React and React Native for expressing your UI components concisely.

It effectively lets you write XML-style code directly in your JavaScript files, making component definition and nesting visually intuitive.

# JSX Transpiling

The Babel transpiler lies at the heart of JSX functionality, converting JSX into regular JavaScript for compatibility with web and mobile platforms.

*Q7. Can you list some of the core components in React Native?*

# Core Components

1. View: The basic container that supports layout with Flexbox.
2. Text: For displaying text.
3. Image: For displaying images either from the local file system or the network.

# Specialized Components

1. ScrollView: For displaying a scrollable list of components.
2. Listview (deprecated): A high-performance, cross-platform list view.
3. TextInput: An input component with optional prompts, as well as a variety of keyboard types, enabling text input.

# User Interface

1. Button: A UI component that enables a user to interact with the application.
2. Picker: A dropdown list that displays a picker interface.

# Basic Functionality Components

1. ActivityIndicator: Displays a rotating circle, indicating that the app is busy performing an operation.
2. Slider: Lets the user select a value by sliding the thumb on the bar.
3. Switch: Used for the on/off state.

# List Views

1. FlatList: A core virtualized list component supporting both vertical and horizontal scrolls. It's memory-efficient and only renders the elements on-screen. It also supports dynamic loading.

2. SectionList: Much like FlatList, but also allows you to section your data.

*Q8. What is the significance of the Flexbox layout in React Native?*

# Components Designed for Flexbox

React Native provides specific components empowered by Flexbox, including:

1. Container Components: These are Views and Touchables that house inner elements being placed using Flexbox.

2. Content Components: These are core layout components that handle the arrangement of inner items. Examples include the Text component.

# Core Flexbox Components

1. View: The foundation of Flexbox layout in React Native

2. Text: A specialized View primarily for text-related elements that supports Flexbox

3. Image: A Flexbox-capable View for image elements

4. ScrollView: A container for components that are larger than its size, offering various scrolling methods. It uses Flexbox to arrange these scrollable components.

5. FlatList and SectionList: These specialized components efficiently render large lists and areas as per the screen's dimensions, leveraging the innate performance of the native interface.

6. VirtualizedList: A low-level, high-performance list. Both FlatList and SectionList are built on top of VirtualizedList.

# Flexbox Properties in React Native

1. Direction: Establishes the principal axis of the layout. Options are row and column, with the latter being the default.

2. Alignment: Determines the position of items along the secondary axis. Common settings include flex-start, center, and flex-end. The setting stretch is also available, which extends the components to fill the empty space.

3. Order: Flexbox allows for reordering of elements. This property defines the display order, with the default being 0.

4. Proportional Sizing: Rather than specifying explicit dimensions, items can be sized proportionally to the remaining space, making the layout adaptable to various screen sizes.

5. Gutters: Flexbox in React Native can handle gutters between items effortlessly.

export default function App() {
  return (
    <View style={styles.container}>
      <View style={styles.box} />
      <View style={styles.box} />
      <View style={styles.box} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    flexDirection: 'row',
    justifyContent: 'space-evenly',
    alignItems: 'center',
  },
  box: {
    width: 50,
    height: 50,
    backgroundColor: 'lightgrey',
  },
});

*Q9. Explain the concept of hot reloading in React Native.*

Hot Reloading innovatively speeds up the development process by rendering updates to the running app in real-time. While preserving the app's current state, it allows developers to observe changes in layout, live.

# Key Benefits

- Enhanced Productivity: Avoiding repetitive recompilations and reinstalls.
- Quick Updates: View real-time changes with contextual data.
- Error Localization: Identify issues when and where they occur.

# Mechanism

- State Retention: Unlike a full reload, hot reloading keeps the app's state intact.
- Partial Update: Only files that have been changed are updated, reducing the time needed.

if (__DEV__) {
    const { activateKeepAwake } = require('expo-keep-awake');
    activateKeepAwake();
}

*Q10. What is a TouchableHighlight in React Native?*

TouchableHighlight is a React Native component optimized for touch interactions. It uses the platform's native feedback effect when touched, making it ideal for buttons, tabs, or other interactive elements.

# Key Features

- Accessibility: It automatically handles accessibility states such as focus and press, adhering to WAI-ARIA and native mobile accessibility guidelines.

- Visual Feedback: Upon touch, it provides a visual indication like opacity changes or highlighting, depending on the platform.

- On Press Event: Executes a function when the component is pressed or activated using keyboard or assistive devices.


import { TouchableHighlight, Text, View } from 'react-native';

const CustomButton = ({ label }) => (
  <TouchableHighlight
    style={{ backgroundColor: 'green', padding: 10, margin: 10, borderRadius: 5 }}
    underlayColor="lime"
    onPress={() => alert('Button pressed!')}
  >
    <Text style={{ color: 'white' }}>{label}</Text>
  </TouchableHighlight>
);

const App = () => (
  <View>
    <CustomButton label="Press me!" />
  </View>
);

*Q11. FlatList performance. A list of 5,000 items janks on mid-range Android. Walk through your debugging order: keyExtractor, getItemLayout, windowSize, removeClippedSubviews, initialNumToRender, memoized renderItem, and when you’d move to FlashList and why.*

I would debug the issue step by step rather than changing multiple `FlatList` props at once, because I first need to identify whether the jank is caused by excessive rendering, expensive row components, or virtualization.

**First, I would profile the list.** I would reproduce the issue on a mid-range Android device, preferably in a release build, and check frame drops, JavaScript thread activity, and expensive component renders. I would also check whether images, nested components, or API-driven state updates are contributing to the problem.

**Second, I would check `keyExtractor`.** Every item should have a stable, unique key, preferably a database ID, rather than an array index. This helps React identify items correctly during updates and avoids unnecessary remounting when the list changes.

**Third, I would optimize row rendering.** I would move the row into a separate component wrapped with `React.memo` and memoize `renderItem` using `useCallback` where appropriate. I would also avoid creating unnecessary objects or functions inside the row and make sure props remain stable. However, I would not expect memoization to solve the problem if each row performs expensive work or receives changing props on every render.

**Fourth, I would use `getItemLayout` if the rows have fixed heights.** It allows `FlatList` to calculate item positions without measuring each row dynamically. This can improve scrolling and jumping to specific indexes. I would not use it with variable-height rows unless I could calculate their dimensions accurately.

**Fifth, I would tune the virtualization settings.** I would adjust `windowSize` to control how many viewports are rendered around the visible area. A smaller window can reduce memory usage and rendering work, but it may cause blank areas during fast scrolling. A larger window can make scrolling smoother at the cost of memory and additional rendering.

I would also tune `initialNumToRender` so the initial screen renders enough items without doing unnecessary work. I would test `removeClippedSubviews` as well, particularly on Android, because it can reduce native view work by detaching off-screen views. However, I would verify the result carefully because it can cause visual issues in some layouts.

**Sixth, I would investigate the row itself.** I would check for large images, expensive calculations, deeply nested views, unnecessary state updates, and broad context changes that cause many rows to re-render. I would fix these issues before assuming that `FlatList` itself is the bottleneck.

**Finally, I would consider FlashList.** If the list still struggles after profiling and optimizing row rendering and virtualization, I would evaluate Shopify's FlashList. It recycles views more aggressively, reducing the overhead of repeatedly creating and destroying cells, which can improve performance for large lists. I would test it with the actual data and target devices rather than switching libraries blindly.

Overall, my approach would be to measure first, fix the actual bottleneck, tune virtualization, and then compare `FlatList` with FlashList using real-device performance metrics.

*Q12. Re-render control. When do React.memo, useCallback and useMemo help, and when do they add overhead for nothing? How would you find unnecessary re-renders in a screen using React DevTools Profiler or Flipper?*

I would first clarify that `React.memo`, `useCallback`, and `useMemo` are performance optimizations, not things I would apply to every component by default.

**First, `React.memo`** is useful when a component receives the same props frequently and rendering it is relatively expensive. It performs a shallow comparison of props and skips re-rendering when those props haven't changed. For example, in a list of products, I could wrap the product row in `React.memo` so that updating one product doesn't force every unchanged row to render again.

However, if the component is very small, its props change on every render, or it frequently receives new object and function references, `React.memo` may provide little benefit. The comparison itself also has a cost.

**Second, `useCallback`** memoizes a function reference between renders, provided its dependencies haven't changed. I would use it when passing callbacks to memoized child components or when a stable function reference is required by a dependency-based hook. For example, I could memoize a `renderItem` callback passed to a `FlatList`.

I wouldn't wrap every event handler in `useCallback`, because creating a function is generally inexpensive. If nothing depends on the function's reference identity, memoizing it may not improve performance. I would also ensure the dependency array is correct to avoid stale values.

**Third, `useMemo`** memoizes the result of a calculation. I would use it for genuinely expensive operations, such as filtering or sorting a large dataset, when the inputs remain unchanged across renders. It can also help preserve object or array references passed to memoized children.

I wouldn't use it for simple calculations, such as adding two numbers or formatting a small value, because the memoization overhead and extra complexity may exceed the cost of recalculating the result.

**Next, I would identify unnecessary re-renders using the React DevTools Profiler.** I would reproduce the interaction that causes lag, record a profiling session, and inspect the commits to see which components rendered, how frequently they rendered, and how much time they consumed. I would look for components that repeatedly render when their relevant data hasn't changed.

If a product row renders whenever an unrelated screen state changes, I would inspect its props and determine whether a changing object reference, callback, context value, or parent update is responsible. I would then apply the appropriate fix and profile the screen again.

In React Native, I could also use Flipper where the project's React Native version and installed tooling support the relevant plugins. However, I wouldn't rely on Flipper being available in every setup; React DevTools Profiler is my primary choice for analyzing React rendering behavior. For deeper investigation, I would correlate those results with Android performance tools to distinguish JavaScript rendering delays from native UI or image-rendering bottlenecks.

**Finally, I would validate the optimization.** I would compare profiling results before and after the change, check whether unnecessary renders decreased, and confirm that interactions became smoother without introducing stale data or incorrect UI.

My rule is simple: I don't add memoization because a component re-renders. I add it when profiling shows that avoiding the render or calculation is worth the cost.

*Q13. Hermes and startup time. What does Hermes change (bytecode precompilation, memory, GC)? How do you measure and reduce cold-start time, for example with inline requires, lazy screens and a smaller bundle?*

Hermes is a JavaScript engine optimized for React Native. Its main goal is to improve application performance, particularly startup time and memory usage on mobile devices.

**First, bytecode precompilation.** In a typical Hermes release build, JavaScript is compiled into Hermes bytecode during the build process instead of requiring the device to parse and compile the entire JavaScript bundle at runtime. This can reduce startup work. However, it doesn't eliminate JavaScript execution, module initialization, React component rendering, or native initialization.

**Second, memory usage and garbage collection.** Hermes is designed to reduce memory overhead on mobile devices, and its runtime and garbage collector manage JavaScript objects and reclaim memory that is no longer reachable. Actual memory usage depends on the application, retained objects, bundle size, and allocation patterns. I would measure memory rather than assume Hermes automatically makes every app use less memory.

**To optimize cold-start time, I would follow a measurement-first approach.**

First, I would establish a baseline using a release build on a physical Android device. I would measure time from launching the application to the first usable screen, distinguishing native startup, JavaScript initialization, and screen rendering where possible. I would use Android profiling tools such as Perfetto or Android Studio System Trace, along with React Native performance tools, to identify the expensive phase. I would repeat the measurements because startup time can vary between runs.

Second, I would check the JavaScript bundle and module initialization. If the application imports many screens, analytics SDKs, or utility libraries at startup, those modules may add initialization work even when the user doesn't immediately need them.

I would use **inline requires** where appropriate so modules are initialized when first needed rather than eagerly during startup. I would also implement **lazy screen loading**, particularly for less frequently accessed screens, so the initial navigation path loads only what is necessary. I would be careful with dependencies and navigation configuration because some modules may still be loaded eagerly.

Third, I would reduce unnecessary bundle weight. I would inspect large dependencies, remove unused libraries and duplicate functionality, and use narrower imports where supported. I would also defer nonessential initialization, such as analytics setup or secondary data fetching, until after the first screen becomes usable, provided that doing so doesn't break application behavior.

Finally, I would benchmark the optimized release build against the original baseline. I would compare cold-start time, time to first usable screen, JavaScript initialization, and memory usage. I would also verify that deferred modules load correctly when the user navigates to those screens.

In summary, Hermes reduces JavaScript startup overhead through bytecode compilation, while techniques such as inline requires, lazy screen loading, and reducing unnecessary initialization help reduce the amount of work performed during startup. I would apply these optimizations based on profiling results and validate them on real devices.

*Q14. Old architecture vs New Architecture (Fabric, TurboModules, JSI). What problem does the async JSON bridge cause? How do JSI, Fabric and TurboModules fix it? What breaks or needs migrating when you turn the New Architecture on in an existing app?*

In React Native, the Old Architecture uses a bridge to communicate between JavaScript and native code. The New Architecture introduces JSI, Fabric, and TurboModules to make that communication more direct and improve rendering and native module integration.

**First, the problem with the old asynchronous JSON bridge.**

In the Old Architecture, JavaScript and native code communicate through a bridge that batches and serializes messages. Data often needs to be converted into a serialized representation before being passed between the two sides, then processed on the receiving side.

This creates serialization overhead, additional memory allocations, and scheduling delays. When an application sends frequent messages, such as continuous sensor updates, animations, or large data payloads, that communication overhead can become a performance bottleneck. The asynchronous nature of the bridge can also make certain interactions more difficult to coordinate.

**Second, JSI — JavaScript Interface.**

JSI provides a C++ interface that allows the JavaScript engine to interact with native C++ objects and functions more directly, without requiring every interaction to go through the old serialized bridge. This can reduce communication overhead and enables synchronous access where the API and threading model permit it.

However, JSI does not mean that every native operation becomes synchronous or automatically runs faster. Threading rules and the cost of the underlying operation still matter.

**Third, TurboModules.**

TurboModules use the New Architecture's native module system. They support typed specifications, code generation, and lazy initialization, so a native module can be initialized when it is first needed rather than loading every module eagerly at startup.

They integrate with JSI instead of relying on the old bridge mechanism. This can improve startup efficiency and native module communication. During migration, existing native modules may need new specifications, generated code, and changes to their Android or iOS implementations.

**Fourth, Fabric.**

Fabric is React Native's new rendering system. It uses a shared C++ renderer and integrates more closely with React's rendering process. It improves coordination between JavaScript and native views and supports concurrent rendering features and more consistent handling of UI updates.

For example, in a complex screen with many React Native components, Fabric can improve how updates are coordinated with the native UI. However, it does not automatically eliminate expensive React renders or fix poorly optimized components.

**Finally, migrating an existing application.**

I would first audit React Native and third-party library compatibility, especially libraries that contain native Android or iOS code.

I would check whether native modules support the New Architecture, migrate custom modules to TurboModules where necessary, and verify custom native views against Fabric requirements. I would also review dependencies that rely on internal React Native APIs or assumptions about the old bridge.

Next, I would enable the New Architecture in a development branch, rebuild both Android and iOS, and test critical flows such as navigation, authentication, camera access, notifications, payments, and background tasks. I would check native build errors, runtime crashes, event handling, and any differences in threading or initialization behavior.

Finally, I would compare performance against the existing release build and roll out only after compatibility and regression testing.

In short, JSI improves how JavaScript interacts with native code, TurboModules modernize native modules and support lazy initialization, and Fabric modernizes rendering. The main migration challenge is ensuring that existing native modules and third-party libraries are compatible with the new execution and rendering systems.

*Q15. Navigation internals. In React Navigation, what happens to screens in a stack, tab and drawer when you navigate (mounted vs unmounted)? How do you handle useFocusEffect vs useEffect, and avoid stale data or memory leaks when returning to a screen?*

In React Navigation, I distinguish between a screen being mounted, focused, blurred, and unmounted. These are different lifecycle states, and understanding them helps prevent unnecessary API calls, stale data, and memory leaks.

**First, stack navigation.**

When I navigate from Screen A to Screen B, Screen A usually remains mounted underneath Screen B. If I go back, Screen B is typically removed from the stack and unmounted, while Screen A becomes focused again.

For example, if I open a product details screen from a product list, the list usually remains mounted when I navigate forward. Therefore, I wouldn't assume that `useEffect` runs again whenever the user returns to the list.

**Second, tab navigation.**

With bottom-tab navigation, screens that have been visited generally remain mounted when switching tabs, depending on the navigator configuration and options such as lazy loading or unmounting inactive screens. Switching tabs typically changes which screen is focused rather than necessarily unmounting the previous screen.

This means that an API request inside `useEffect` with an empty dependency array normally won't run again just because the user returns to an already-mounted tab.

**Third, drawer navigation.**

Drawer screens have similar considerations. A screen may remain mounted after the drawer closes or when another drawer screen becomes active. The exact behavior depends on the navigator configuration and whether the screen has been visited. I would use focus events rather than assume that opening or closing the drawer remounts its screens.

**Next, `useEffect` versus `useFocusEffect`.**

I use `useEffect` for component lifecycle work, such as initializing something when a component mounts or reacting to changes in its dependencies. Its cleanup runs when dependencies change or when the component unmounts.

I use `useFocusEffect` when work should happen whenever the screen becomes focused, such as refreshing a list, fetching updated data, or subscribing to screen-specific events. Its cleanup runs when the screen loses focus or when the effect's dependencies change.

For example, if a user opens a profile screen, navigates to another screen, and then returns, I might use `useFocusEffect` to refresh the profile data because it may have changed while the screen was inactive.

I would wrap the callback in `useCallback` to control when the focus effect is recreated, using the appropriate dependencies.

**Finally, avoiding stale data and memory leaks.**

For API requests, I would cancel in-flight requests using `AbortController` when supported, or use a cancellation mechanism provided by the networking library. If cancellation isn't available, I would prevent outdated responses from overwriting newer state.

For event listeners, timers, and subscriptions, I would always clean them up in the appropriate effect cleanup function. I would also avoid unnecessarily refetching data on every focus if caching or a stale-time strategy already provides sufficiently fresh data.

For example, in a product-list screen, I could fetch data when the screen gains focus, cancel the request when it loses focus, and update the list only when the response belongs to the current request. If the data is already fresh, I could reuse cached results instead of making another request.

My rule is to use `useEffect` for component lifecycle and dependency-driven work, and `useFocusEffect` for work tied to the screen's visibility. I also make cleanup and request cancellation part of the implementation rather than treating them as optional optimizations.

import { useCallback, useState } from 'react';
import { useFocusEffect } from '@react-navigation/native';

function ProductsScreen() {
  const [products, setProducts] = useState([]);

  useFocusEffect(
    useCallback(() => {
      const controller = new AbortController();

      async function loadProducts() {
        try {
          const response = await fetch(
            'https://api.example.com/products',
            { signal: controller.signal }
          );

          if (!response.ok) {
            throw new Error('Failed to fetch products');
          }

          const data = await response.json();
          setProducts(data);
        } catch (error) {
          if (error.name !== 'AbortError') {
            console.error(error);
          }
        }
      }

      loadProducts();

      return () => {
        controller.abort();
      };
    }, [])
  );

  // Render the product list here.
}

*Q16. Animations. Why is the Animated API with useNativeDriver: true smoother than JS-driven animation? What can’t the native driver animate? How does Reanimated worklets/UI-thread execution differ, and when do you pick it over Animated?*

In React Native, I choose an animation approach based on how complex the animation is, which properties need to be animated, and whether it depends on continuous gestures or interactions.

**First, why is `Animated` with `useNativeDriver: true` smoother?**

With a JavaScript-driven animation, JavaScript calculates animation values on each frame and sends updates to the native UI. If the JavaScript thread is busy with API processing, expensive renders, or other calculations, those updates can be delayed, causing dropped frames and janky animations.

With `useNativeDriver: true`, React Native can serialize a supported animation configuration and execute the animation on the native side. Once started, the animation can continue even if the JavaScript thread becomes temporarily busy. This is particularly useful for animations such as fading a view in or translating it across the screen.

For example, I would use the native driver for a simple opacity or transform animation because the native side can update those properties without requiring JavaScript to calculate every frame.

**Second, what can't the native driver animate?**

The traditional Animated native driver supports non-layout properties, primarily `opacity` and transforms such as `translateX`, `translateY`, `scale`, and `rotate`.

It generally cannot animate layout properties such as `height`, `width`, `margin`, `padding`, or `top` using the traditional native driver. These properties can require layout recalculation, which the native driver does not handle through that mechanism. It also doesn't support arbitrary styles or every color animation in the same way across versions.

If I need to animate layout dimensions, I would consider Reanimated or another suitable approach instead of forcing an unsupported property into the native driver.

**Third, how does Reanimated work differently?**

React Native Reanimated uses worklets, which are functions that can execute on the UI runtime rather than requiring every animation update to travel through the JavaScript thread.

I can define shared values and update them during gestures or animation frames. Animated styles derived from those values can be evaluated on the UI runtime, allowing animations to continue smoothly even when the main JavaScript thread is busy.

For example, for a swipeable card, I could update its horizontal position directly from gesture input, apply rotation based on the same shared value, and trigger a decision when the user releases the card. This avoids sending every gesture update back to JavaScript for processing.

Reanimated also supports more advanced interactions, including gesture-driven animations, coordinated sequences, derived values, and layout transitions. However, worklets have execution constraints, and calling back into JavaScript still has a cost.

**Finally, when would I choose each approach?**

I would choose the built-in Animated API with the native driver for straightforward opacity and transform animations, such as a fade-in, a simple modal transition, or an icon scaling effect. It is a good option when the animation is simple and I want to avoid additional dependencies.

I would choose Reanimated for complex gesture-driven interactions, draggable components, swipeable cards, coordinated animations, and interactions that need to respond continuously on the UI thread. It gives me more control over animation logic and reduces dependence on per-frame JavaScript execution.

In either case, I would test on a real mid-range Android device and profile the result. A native-driven animation can still perform poorly if the native UI thread is overloaded, and Reanimated cannot compensate for every rendering or layout bottleneck.

My approach is to use the simplest animation system that meets the requirements, and introduce Reanimated when the interaction genuinely benefits from UI-runtime execution or its more advanced animation capabilities.

*Q17. Native modules and platform differences. You need a feature with no library (e.g. a vendor SDK or Bluetooth printer). How do you write a native module or TurboModule for Android and iOS? How do you handle permissions and platform-specific behaviour (Platform.select, .android.js files)?*

If I need a feature that doesn't have a suitable React Native library, such as integrating a Bluetooth printer or a vendor-specific SDK, I would first check whether the vendor provides native Android and iOS SDKs. If it does, I would create a native integration and expose the required functionality to JavaScript through a React Native native module.

**First, I would define a common JavaScript interface.**

I would design a small API with methods such as `connect()`, `print()`, and `disconnect()`. I would define the expected parameters, return values, and error handling so that the rest of the React Native application doesn't need to know how the underlying platform works.

**Second, I would implement the Android side.**

For Android, I would use Kotlin or Java to integrate the vendor SDK and expose the required methods through a React Native native module. I would configure the required permissions in `AndroidManifest.xml`, handle runtime permission requests where necessary, and ensure Bluetooth operations follow Android's threading and lifecycle requirements.

I would also account for Android-version differences. For example, Bluetooth permissions differ between Android 11 and earlier versions and Android 12 and later versions, where permissions such as `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` may be required depending on the operations being performed.

**Third, I would implement the iOS side.**

For iOS, I would integrate the vendor SDK using Swift or Objective-C and expose equivalent methods through a native module. Depending on the SDK and React Native version, this could involve Objective-C bridging or the New Architecture's TurboModule mechanism.

I would configure the appropriate usage descriptions in `Info.plist`, such as Bluetooth access descriptions where required, and handle permission denial, Bluetooth availability, connection failures, and device disconnection.

**Fourth, I would decide between a legacy native module and a TurboModule.**

For an existing application, I would check the React Native version and whether the vendor SDK or existing native modules support the New Architecture.

A legacy native module can be appropriate when maintaining an existing compatible integration. For a new module targeting the New Architecture, I would define a typed TypeScript or Flow specification, configure code generation, and implement the corresponding Android and iOS native interfaces.

I wouldn't choose TurboModules just because they are newer; I would choose the approach that fits the application's architecture and compatibility requirements.

**Fifth, I would handle platform-specific behaviour cleanly.**

If only a small piece of logic differs, I would use `Platform.select()` or `Platform.OS`. For example, Android and iOS may require different permission flows or SDK configuration.

If the implementations are substantially different, I would create separate files such as `PrinterService.android.ts` and `PrinterService.ios.ts`. React Native can resolve the appropriate file for the target platform, while both files expose the same JavaScript interface.

This keeps platform-specific code isolated instead of scattering platform checks throughout the UI.

**Finally, I would test failure scenarios on real devices.**

I would test permission denial, Bluetooth being disabled, printer disconnection, connection timeouts, repeated print requests, and app backgrounding or termination. I would also ensure native errors are converted into consistent JavaScript errors and that resources are released correctly when disconnecting.

My goal is to expose a clean, predictable JavaScript API while keeping the platform-specific implementation, permissions, and SDK details inside the native layer.

*Q18. Offline-first and sync. Design a flow where a user creates orders offline and syncs later (relevant to POS apps). Which storage would you use (MMKV, SQLite, WatermelonDB, AsyncStorage) and why? How do you handle conflicts, retries, idempotency and queue ordering?*

For an offline-first POS application, I would design the order flow so that creating an order does not depend on network availability. The user should be able to create orders, add items, and continue billing offline, with synchronization happening automatically when connectivity returns.

**First, I would choose the local storage.**

For a POS application, I would prefer SQLite because orders contain related data such as order items, products, payments, and synchronization status. SQLite supports transactions, constraints, and structured queries, which are important for keeping order data consistent.

MMKV is useful for fast key-value storage, such as small preferences or cached configuration, but it isn't my primary choice for relational order data. AsyncStorage is suitable for simple, non-critical key-value data, but I wouldn't use it as the main transactional order database. WatermelonDB is another option when the application has large datasets and complex reactive queries, but it adds an abstraction and setup that I would evaluate against the project's requirements.

**Second, I would save every order locally before attempting synchronization.**

When a user creates an order, I would generate a unique client-side order ID, such as a UUID, and save the order, its items, and an outbox synchronization record in a single SQLite transaction. The order would initially have a status such as `pending_sync`.

This is important because I don't want an order to disappear if the app closes or the device loses connectivity immediately after the user confirms it. The UI can show the locally saved order immediately, without waiting for the server.

**Third, I would implement a durable synchronization queue.**

Each pending operation would have an operation ID, entity ID, operation type, payload, creation sequence, retry count, and status. When the device regains connectivity, a background or foreground sync worker would read pending operations in order and send them to the backend.

I would use exponential backoff with jitter for transient failures, such as network timeouts or server errors, and apply a maximum retry policy. Authentication errors would trigger re-authentication, while validation errors would be marked as failed and surfaced for correction rather than retried indefinitely.

Connectivity detection would trigger synchronization attempts, but I wouldn't treat a network-connected state as proof that the backend is reachable.

**Fourth, I would handle idempotency to prevent duplicate orders.**

A network request can reach the server successfully even if the client times out before receiving the response. The client may retry the request, so the server must not create the same order twice.

I would generate a stable idempotency key for each order and send it with every retry. The backend would enforce uniqueness for that key within the appropriate business or tenant scope and return the original result if the same operation is submitted again.

The idempotency record and order creation should be committed transactionally on the server. This makes retries safe even when the client cannot tell whether the previous attempt succeeded.

**Fifth, I would handle conflicts and queue ordering.**

For immutable events, such as creating an order, I would preserve the original order ID and creation sequence. For operations that depend on previous operations, I would enforce dependencies—for example, an order update should not be processed before the order itself exists.

For editable records, such as customer details, I would use a version number or server revision and detect conflicting updates. Depending on the business rules, I could reject a stale update, merge independent fields, or require the user to resolve the conflict.

I would not blindly apply last-write-wins to financial records. Order totals, discounts, stock deductions, and payments must follow server-side business rules. The backend should validate prices and inventory and process stock changes transactionally.

**Finally, I would reconcile successful synchronization.**

Once the backend confirms an order has been accepted, I would update the local synchronization status and store the server ID or server revision in a local transaction. I would remove or mark the corresponding outbox operation as completed only after the acknowledgement is safely recorded.

If the app crashes during synchronization, the pending operation remains available for retry. Because the operation is idempotent, repeating it will not create a duplicate order.

Overall, my design would use SQLite for durable local transactions, an outbox queue for reliable synchronization, stable IDs and server-side idempotency for safe retries, and explicit conflict rules for financial and inventory data.

*Q19. Secure storage and auth. Where do you store tokens (Keychain/Keystore vs AsyncStorage)? How do you implement refresh-token rotation with concurrent failing requests (a single-flight refresh in an Axios interceptor)? What do you do about certificate pinning and rooted devices?*

For authentication in a React Native application, I would focus on three areas: securely storing credentials, handling token expiration without causing concurrent refresh requests, and protecting communication between the app and the backend.

**First, secure token storage.**

I would store refresh tokens in platform-backed secure storage, such as iOS Keychain and Android Keystore-backed storage through an appropriate React Native library. I would not store sensitive authentication tokens in AsyncStorage because it is not designed as secure credential storage.

For access tokens, I would generally keep them in memory while the app is running and use the refresh token to obtain a new access token when necessary. If the application needs to restore authentication after a restart, I would securely persist the refresh token and retrieve it when the app initializes.

I would also configure appropriate Keychain accessibility and Android storage options, avoid logging tokens, and clear credentials during logout. Secure storage reduces exposure, but it doesn't eliminate every risk on a compromised device.

**Second, I would implement refresh-token rotation with a single-flight refresh mechanism.**

Suppose five API requests are running simultaneously and all receive a 401 response because the access token has expired. If each request independently calls the refresh endpoint, the requests may race, especially when the backend rotates refresh tokens and invalidates the previous token after use.

To prevent that, I would maintain a shared refresh promise. The first failing request starts the refresh operation, and subsequent failing requests wait for the same promise instead of initiating separate refresh calls.

When the refresh succeeds, I would atomically update the access token and rotated refresh token in the appropriate storage, then retry the failed requests with the new access token. If refresh fails because the refresh token is invalid or revoked, I would clear the authentication state and require the user to sign in again.

I would also mark retried requests so they cannot enter an infinite retry loop, and I would exclude the refresh endpoint itself from the normal refresh interceptor.

**Third, certificate pinning.**

HTTPS already provides encryption and server authentication through TLS certificate validation. Certificate pinning adds another layer by checking that the server's certificate or public key matches an expected value.

For an application handling sensitive POS or financial data, I would evaluate pinning based on the threat model. If implemented, I would use a maintained native-compatible solution, define a pin rotation strategy, and maintain backup pins so that a legitimate certificate renewal doesn't unexpectedly break connectivity for every user.

I would test certificate renewal and failure scenarios carefully. Pinning is not a replacement for normal TLS validation, and incorrect implementation can create a serious availability problem.

**Fourth, rooted or jailbroken devices.**

I would treat root or jailbreak detection as a risk signal rather than a perfect security boundary. Detection can be bypassed, and false positives are possible.

For sensitive applications, I would combine platform integrity signals, secure storage, server-side authorization, short-lived access tokens, refresh-token rotation, and appropriate monitoring. Depending on the business risk, I might restrict particularly sensitive actions or require additional verification on a compromised device.

I would never rely solely on a client-side root check to protect an API. The backend must independently validate authentication, authorization, token revocation, and business rules.

Overall, my approach is to minimize credential exposure, coordinate token refreshes through a single shared promise, secure network communication, and enforce critical security decisions on the server rather than trusting the mobile client.

*Q20. Release, OTA and crash handling. How does CodePush or EAS Update work, and what can’t be shipped over the air? How do you handle app signing, ProGuard/R8, Hermes source maps for crash symbolication (Sentry/Crashlytics), and a phased rollout with a rollback plan?*

For a React Native application, I treat a release as a controlled process that includes building, signing, testing, monitoring, and having a rollback strategy.

**First, CodePush and EAS Update.**

CodePush and EAS Update allow us to deliver compatible JavaScript and application assets over the air without requiring users to download a new store release for every change.

For example, if I fix a bug in a React Native screen or update JavaScript business logic, I may be able to publish an OTA update, provided it is compatible with the native binary already installed on the device.

However, OTA updates cannot replace native code that is compiled into the application. Changes such as adding a new native module, modifying Android or iOS permissions, upgrading certain native dependencies, or changing native project configuration generally require a new binary and a store release.

I would also check the current toolchain. CodePush's original App Center service was retired, so I would not assume the legacy service is available for a new project. EAS Update is an option for Expo-based projects, including appropriately configured React Native projects using Expo's tooling. For bare React Native applications, I would evaluate the update solution against the existing build and deployment architecture.

**Second, app signing and release builds.**

On Android, I would generate a signed release build using the appropriate upload or signing key configuration. With Google Play App Signing enabled, Google manages the app-signing key while the developer signs uploads using the upload key.

On iOS, I would configure the correct bundle identifier, distribution signing certificate, provisioning profile, and release configuration. Signing identities and credentials must be protected, and I would never commit private signing keys or passwords to source control.

Before release, I would verify the package identifiers, version and build numbers, environment configuration, and production API endpoints.

**Third, ProGuard and R8.**

On Android, R8 can perform code shrinking, optimization, and obfuscation for supported Java and Kotlin code. This can reduce application size and make reverse engineering more difficult, although obfuscation is not a substitute for security controls.

I would enable the appropriate release configuration and add keep rules for libraries that depend on reflection, JNI, or specific class names. I would test the minified release build because some issues appear only after shrinking or obfuscation.

For crash reporting, I would retain the mapping files generated by R8 for each release so that obfuscated Java and Kotlin stack traces can be deobfuscated correctly.

**Fourth, Hermes source maps and crash symbolication.**

Hermes compiles JavaScript into bytecode in typical release configurations. When a production crash occurs, the reported stack may contain minified JavaScript locations or Hermes bytecode offsets rather than useful source locations.

I would generate the matching source maps for the exact release bundle and upload them to the crash-reporting service, such as Sentry or Firebase Crashlytics where the integration supports the relevant React Native stack traces.

I would associate the artifacts with the correct release, platform, application version, and build identifier. I would also retain native debug symbols when needed for native crash symbolication.

The important point is that source maps and debug symbols must match the exact binary and JavaScript bundle. Uploading artifacts from a different build can produce incorrect or unusable stack traces.

**Finally, phased rollout and rollback.**

I would first test the release internally, then distribute it to a small beta group. For a store release, I would use a staged or phased rollout where supported and monitor crash-free sessions, crash-free users, startup time, API failures, and important business flows.

For OTA updates, I would publish to a limited channel or audience first if the chosen update system supports it, then expand the rollout after validating the results.

I would define rollback criteria before releasing. If crash rates increase or a critical flow fails, I would halt the rollout and revert to a known-good version using the platform's supported rollback mechanism. For an OTA update, I would use the update service's rollback or republish strategy. For a native binary issue, I would stop the staged rollout and prepare a fixed store release; I would not assume every installed binary can be downgraded automatically.

Overall, I would treat release management as a combination of reproducible builds, protected signing credentials, correct symbolication artifacts, staged deployment, monitoring, and a tested rollback plan.

*Q21. State management at scale. Compare Redux Toolkit, Zustand and React Query/TanStack Query. What belongs in server state vs client state? How do you handle cache invalidation, optimistic updates and rollback on a failed mutation in an order-creation flow?*

When choosing a state management solution for a React Native application, I first identify what kind of state I am managing. I don't automatically put everything into Redux or Zustand because server data and local UI state have different requirements.

**First, Redux Toolkit.**

Redux Toolkit is useful for managing shared client-side state across multiple screens. It provides a predictable state structure, reducers, actions, and middleware, while simplifying traditional Redux boilerplate.

For example, I might use it for a multi-step checkout workflow, application-wide preferences, or a complex client-side workflow where multiple components need coordinated updates. However, I wouldn't use Redux as my default solution for all API responses because that means I would also need to manage loading states, caching, request deduplication, and cache invalidation myself.

**Second, Zustand.**

Zustand is a lightweight state management library with a simple store and selector-based subscriptions. I would consider it for state that needs to be shared across screens without the additional structure of Redux Toolkit.

For example, in a restaurant POS application, I could use Zustand to maintain the current cart, selected table, or active order draft. I would use selectors so that a component subscribes only to the part of the store it needs.

However, Zustand doesn't automatically provide the server-state caching and invalidation features that a query library offers.

**Third, TanStack Query, also known as React Query.**

I use TanStack Query for server state: data fetched from APIs that can become stale or change on the backend. It provides query caching, request deduplication, background refetching, retries, and mutation lifecycle callbacks.

For example, product lists, menu items, order history, and customer records are server state. TanStack Query helps manage their lifecycle without manually building a separate caching system in Redux or Zustand.

It is important to understand that TanStack Query doesn't replace all client state management. It solves a different problem.

**Next, I separate server state from client state.**

In a restaurant POS application, menu items fetched from the backend, server-confirmed orders, and order history belong to server state. The selected table, an open modal, temporary form input, and local UI preferences belong to client state.

A cart or draft order needs a more deliberate decision. If it is temporary UI data, I might keep it in local component state or Zustand. If it must survive app restarts or work offline, I would persist it in a local database such as SQLite and synchronize it with the backend later.

I would avoid maintaining multiple independent copies of the same server data in Redux, Zustand, and TanStack Query because they can become inconsistent.

**Finally, I would handle order creation using a mutation and cache invalidation.**

When the user submits an order, I would call a TanStack Query mutation. If the order should appear immediately, I could apply an optimistic update to the relevant cached order list and retain the previous cache value for rollback.

If the request succeeds, I would replace the optimistic entry with the authoritative server response and invalidate related queries, such as the order list or table status, so that the next fetch retrieves fresh data.

If the request fails, I would restore the previous cached value and show an appropriate error message. I would distinguish a definite failure from an ambiguous network timeout because the server may have created the order even if the client never received the response.

For an offline-first POS application, I would not rely solely on a temporary optimistic cache entry. I would persist the order locally and use a durable synchronization queue with an idempotency key so retries cannot create duplicate orders.

My overall approach is to use TanStack Query for remote data, Zustand or Redux Toolkit for shared client state depending on complexity, and a local database when data must survive restarts or support offline transactions.


import { useMutation, useQueryClient } from '@tanstack/react-query';

type Order = {
  id: string;
  status: string;
  items: { productId: string; quantity: number }[];
};

function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (input: {
      idempotencyKey: string;
      items: { productId: string; quantity: number }[];
    }): Promise<Order> => {
      const response = await fetch(
        'https://api.example.com/orders',
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Idempotency-Key': input.idempotencyKey,
          },
          body: JSON.stringify({ items: input.items }),
        }
      );

      if (!response.ok) {
        throw new Error('Order request failed');
      }

      return response.json();
    },

    onMutate: async (input) => {
      await queryClient.cancelQueries({
        queryKey: ['orders'],
      });

      const previousOrders =
        queryClient.getQueryData<Order[]>(['orders']);

      const optimisticOrder: Order = {
        id: `pending-${input.idempotencyKey}`,
        status: 'pending_sync',
        items: input.items,
      };

      queryClient.setQueryData<Order[]>(
        ['orders'],
        (old = []) => [...old, optimisticOrder]
      );

      return { previousOrders };
    },

    onError: (_error, _input, context) => {
      if (context?.previousOrders !== undefined) {
        queryClient.setQueryData(
          ['orders'],
          context.previousOrders
        );
      }
    },

    onSuccess: (serverOrder, input) => {
      queryClient.setQueryData<Order[]>(
        ['orders'],
        (old = []) => [
          ...old.filter(
            (order) =>
              order.id !== `pending-${input.idempotencyKey}`
          ),
          serverOrder,
        ]
      );
    },

    onSettled: () => {
      queryClient.invalidateQueries({
        queryKey: ['orders'],
      });

      queryClient.invalidateQueries({
        queryKey: ['tables'],
      });
    },
  });
}

*Q22. Metro bundler and the build pipeline. How does Metro resolve modules, transform code (Babel) and produce a bundle? What are metro.config.js customizations used for (monorepo setups, SVG transformers, asset extensions)? How do you debug “Unable to resolve module” errors and stale-cache problems?*

**Metro Bundler and the React Native Build Pipeline**

Metro is the JavaScript bundler used by React Native. Its main responsibilities are module resolution, code transformation, dependency graph construction, and bundling.

When I start the application using `npx react-native start`, Metro starts a development server. It begins with the application entry point, usually `index.js`, and recursively discovers the modules imported by the application.

**First, module resolution:** Metro resolves imports such as relative files, npm packages, platform-specific files like `Button.android.tsx` and `Button.ios.tsx`, and supported assets. It follows the project's module resolution rules to locate the actual file.

**Second, code transformation:** Metro passes JavaScript and TypeScript/JSX source through Babel and the configured transformers. Babel transforms JSX and modern JavaScript syntax into code that the target JavaScript runtime can execute. TypeScript type annotations are removed during transformation; TypeScript type-checking is a separate responsibility.

**Third, dependency graph and bundling:** Metro builds a dependency graph from the entry point and its imports, then combines the required modules into a JavaScript bundle. In development, it can serve modules incrementally and support Fast Refresh. For production, React Native's build process generates a bundle and assets that are packaged into the Android or iOS application. Hermes bytecode compilation may also occur as part of the release build when Hermes is enabled.

The `metro.config.js` file customizes this process. For example, in a monorepo, I may configure workspace package resolution and watch folders. For SVG support, I can configure `react-native-svg-transformer` so SVG files can be imported as React components. I can also customize `resolver.assetExts` and `resolver.sourceExts` when adding supported asset or source extensions.

When debugging an `Unable to resolve module` error, I first verify the import path, filename casing, file extension, package installation, and whether the module is accessible from Metro's project and watch folders. I then inspect `metro.config.js` for incorrect resolver settings. If the configuration and dependencies are correct but Metro appears to be serving stale results, I restart it with `npx react-native start --reset-cache`. If necessary, I reinstall dependencies or rebuild the native application, depending on the actual cause.

My main principle is to identify whether the issue is caused by an incorrect import, dependency resolution, transformer configuration, stale Metro cache, or the native build pipeline rather than clearing every cache immediately.

*Q23. Keyboard, safe areas and layout bugs. How do you handle KeyboardAvoidingView differences between iOS and Android, notches and gesture bars (react-native-safe-area-context), and forms inside scroll views? Why does flex behave differently from web CSS, and how does Yoga compute layout?*

In React Native, I handle keyboard and layout issues by considering the differences between iOS and Android, using safe-area insets correctly, and making sure forms remain accessible when the keyboard opens.

First, for keyboard handling, I use KeyboardAvoidingView. On iOS, I generally use behavior="padding" because it adds padding to prevent the keyboard from covering the content. On Android, I usually rely on the operating system's adjustResize behavior, depending on the screen configuration, because Android handles keyboard resizing differently. I test both platforms rather than assuming the same configuration will work everywhere. I also adjust the keyboardVerticalOffset when a navigation header affects the available space.

Second, for notches and system bars, I use react-native-safe-area-context. I wrap the application or relevant screen in SafeAreaProvider and use SafeAreaView or useSafeAreaInsets() to handle areas such as the iPhone notch, status bar, and bottom gesture bar. This prevents content, buttons, and input fields from appearing underneath system UI. I also avoid applying safe-area padding twice, which can create unnecessary spacing.

Third, for forms inside scroll views, I use ScrollView when the form needs to scroll, especially on smaller devices. I combine it with keyboard handling and use keyboardShouldPersistTaps="handled" so users can tap buttons while the keyboard is open. For longer forms, I make sure the focused input can scroll into view and that the submit button remains accessible. I also test different screen sizes and keyboard configurations because a layout that works on one device may fail on another.

Now, regarding flex and Yoga, React Native does not use the browser's CSS layout engine. It uses Yoga, a cross-platform layout engine that calculates the size and position of components based on properties such as flex, flexDirection, justifyContent, alignItems, padding, and margins.

For example, React Native uses flexDirection: 'column' by default, whereas web CSS uses row. Also, flex: 1 generally tells a component to occupy the available space within its parent, provided the parent has a meaningful size constraint. If the parent does not have a suitable layout constraint, the component may not expand as expected. That's why I check the parent-child layout hierarchy when debugging unexpected spacing or sizing issues.

Finally, my debugging approach is to reproduce the issue on both platforms, inspect the component hierarchy and parent dimensions, verify safe-area and keyboard settings, and check for conflicting margins, padding, or flex values. I prefer fixing the underlying layout issue rather than adding arbitrary offsets that might break on other devices.

That is how I approach keyboard handling and responsive layouts in React Native while keeping the UI consistent across iOS and Android.

*Q24. Images and media performance. Your app loads many remote images (book covers, menu photos) and some fail or render blurry. How do you handle caching (react-native-fast-image/expo-image), resizing at the server, placeholders, fallback on error, and memory pressure on low-end Android?*

When an app loads many remote images, such as book covers or menu photos, I focus on caching, image optimization, loading and error states, and memory management, especially on low-end Android devices.

**First, for image caching,** I use libraries like `react-native-fast-image` in React Native CLI projects or `expo-image` in Expo projects. These libraries support image caching and efficient loading. I configure caching based on the use case so frequently accessed images do not need to be downloaded repeatedly. For example, book covers can usually be cached for longer, while frequently updated menu photos may require cache invalidation or URL versioning.

**Second, I optimize image sizes on the server.** I avoid downloading full-resolution images when the UI only displays a small thumbnail. For example, if a book cover is displayed at around 120 × 180 points, I request an appropriately sized image instead of downloading the original high-resolution file. I use server-side resizing, compression, and formats like WebP where supported. This reduces network usage, loading time, and memory consumption while maintaining image quality.

**Third, I handle loading and failure states.** I display a placeholder or lightweight skeleton while an image is loading. If the request fails because of a broken URL, network issue, or missing image, I use an error handler to display a fallback image instead of leaving a blank space. I also preserve the correct aspect ratio to prevent images from appearing stretched or distorted.

**Fourth, I manage memory on low-end Android devices.** Large images can consume significant memory after decoding, even when their compressed file sizes are small. To prevent memory issues, I request appropriately sized images, use `FlatList` virtualization for large catalogues, avoid storing image binaries or Base64 strings in React state, and avoid keeping unnecessary image-heavy screens mounted. These practices help reduce memory pressure and the risk of out-of-memory crashes.

**Finally, I test and monitor performance.** I check scrolling smoothness, network requests, cache behavior, image loading failures, and memory usage on lower-end devices. If images still appear blurry, I verify the requested resolution, source image quality, and display dimensions.

Overall, my approach is to download only the image resolution the UI needs, cache images appropriately, provide reliable placeholders and fallbacks, and minimize unnecessary memory usage. This improves image loading, scrolling performance, and reliability across different devices and network conditions.

*Q25. Push notifications end to end. Explain the flow with FCM and APNs: token registration, token refresh, foreground vs background vs killed-state handling, notification channels on Android, and deep-linking to a specific screen from a tapped notification.*

In React Native, I handle push notifications using Firebase Cloud Messaging (FCM) for message delivery on Android and FCM integrated with APNs (Apple Push Notification service) for iOS. The complete flow involves device registration, sending the token to the backend, handling notifications in different app states, and navigating to the appropriate screen when a user taps a notification.

**First, for token registration,** when the application starts, I request notification permission where required and register the device for push notifications. Using Firebase Messaging, I retrieve the FCM registration token and send it to my backend along with the authenticated user's ID and device information. The backend stores this token so it can target the correct device when sending notifications. On iOS, I also ensure that APNs credentials are configured correctly in Firebase, because FCM relies on APNs to deliver notifications to Apple devices.

**Second, I handle token refresh.** FCM registration tokens can change, for example, after reinstalling the application or when Firebase refreshes the token. I listen for token refresh events using `onTokenRefresh` and update the token in my backend. I also register the current token when the user logs in or the app initializes. On logout, I remove or disassociate the token from the user account so notifications are not sent to the wrong user. The backend should also handle invalid or expired tokens by removing them when FCM reports them as unusable.

**Third, I handle notifications according to the app state.** In React Native, there are three main cases:

- **Foreground:** When the app is open, I use `onMessage` to receive the message. Depending on the requirement, I show an in-app notification or display a local notification, because foreground notifications are not automatically presented in the same way on both platforms.
- **Background:** When the app is running in the background, the operating system can display a notification payload. I use `onNotificationOpenedApp` to handle the user tapping it and opening the app.
- **Killed state:** When the app is not running, I use `getInitialNotification` to retrieve the notification that launched the app after the user taps it. I process this after the application and navigation system are initialized.

I also distinguish between notification messages and data messages because their handling differs across platforms and app states. Background data-message processing has platform restrictions, particularly on iOS, so I do not rely on it for guaranteed background execution.

**Fourth, I configure Android notification channels.** On Android 8.0 and later, notifications must use notification channels. I create channels for categories such as orders, bookings, and general updates, with appropriate importance levels, sounds, and vibration settings. The notification must reference the correct channel ID. Users can control channel preferences through Android settings, so I avoid creating unnecessary channels or assuming the app can override a user's disabled channel.

**Fifth, I implement deep linking when a notification is tapped.** For example, if a user receives a notification about a particular booking, the payload can contain a notification type and a booking ID. When the user taps it, I extract that data and navigate to the booking details screen using React Navigation. I handle both background and killed-state notification taps, and I wait until navigation is ready before navigating. I also validate the payload and fetch the latest booking details from the backend instead of blindly trusting notification data.

**Finally, I test the entire flow on real devices.** I verify permissions, token registration and refresh, foreground and background behavior, killed-state handling, Android channels, and navigation from notification taps. I also test invalid tokens and network failures to make sure the backend and application handle errors correctly.

Overall, I treat push notifications as an end-to-end system involving the mobile app, Firebase, APNs, and the backend. The key is to manage tokens reliably, handle each application state correctly, and ensure that tapping a notification takes the user to the intended screen without navigation or authentication issues.

*Q26. Deep linking and universal/app links. How do you configure URL schemes vs Universal Links (iOS) and App Links (Android)? How does React Navigation’s linking config map URLs to screens, and how do you handle a link that arrives while the user is logged out?*

In React Native, I handle deep linking by configuring URL schemes for direct app access and Universal Links on iOS or Android App Links for verified HTTPS URLs. I use React Navigation's `linking` configuration to map incoming URLs to specific screens and handle authentication before navigating to protected screens.

**First, URL schemes vs Universal Links and App Links.** A custom URL scheme, such as `myapp://booking/123`, allows the operating system to open my application directly. However, custom schemes are not domain-verified, and another application could potentially register the same scheme. For production applications, I prefer HTTPS links, such as `https://example.com/booking/123`, because they provide a better web fallback and verified app-opening behavior.

On iOS, I configure Associated Domains in Xcode with an entry such as `applinks:example.com` and host an `apple-app-site-association` file on the domain. On Android, I configure an intent filter with `android:autoVerify="true"` for the relevant HTTPS URLs and host a valid `assetlinks.json` file containing the application's package name and signing certificate fingerprint. These configurations allow the operating system to verify that my app is authorized to handle links for that domain.

**Second, I configure React Navigation's linking system.** I define the supported URL prefixes and map URL paths to screen names through the `linking` configuration. For example, a URL like `https://example.com/booking/123` can map to a `BookingDetails` screen with `bookingId` set to `123`.

```javascript
const linking = {
  prefixes: [
    'myapp://',
    'https://example.com',
  ],
  config: {
    screens: {
      Home: 'home',
      BookingDetails: 'booking/:bookingId',
      Profile: 'profile',
    },
  },
};
```

I pass this configuration to `NavigationContainer` using its `linking` prop. React Navigation then handles incoming links and converts the URL path and parameters into navigation actions. I also configure nested navigators correctly when the target screen is inside a tab or stack navigator.

**Third, I handle links received in different application states.** When the app is already open, the linking system processes incoming URLs and navigates to the appropriate screen. When the app is in the background or completely closed, React Navigation can process the initial URL when the application launches, provided the native linking configuration is correct. I test both initial URLs and subsequent URL events to avoid duplicate navigation.

**Fourth, I handle deep links when the user is logged out.** For example, suppose a user opens a booking link but has not authenticated. I do not immediately open the protected booking screen. Instead, I store the intended destination and its parameters, redirect the user to the login screen, and restore the original destination after successful authentication.

My flow is:

1. Receive the incoming URL and identify the target screen.
2. Check whether the user is authenticated.
3. If authenticated, navigate to the target screen.
4. If logged out, preserve the intended destination and redirect to login.
5. After successful login, navigate to the original destination and fetch the required data from the backend.

I also handle expired sessions, invalid links, missing resources, and unauthorized access. I never rely on the URL alone to authorize access to protected data; the backend must verify the user's permissions.

**Finally, I test the complete flow on real devices.** I verify domain association, app installation and fallback behavior, navigation parameters, nested navigators, logged-out flows, and links received in foreground, background, and killed states.

Overall, my approach is to use verified HTTPS links for production, configure native domain association correctly, let React Navigation map URLs to screens, and preserve the intended destination when authentication is required. This makes deep linking reliable, secure, and consistent across iOS and Android.

*Q27. JavaScript thread vs UI thread. What work blocks the JS thread and freezes interactions (large JSON parsing, heavy loops, sync storage reads)? How do you move work off it with InteractionManager, requestAnimationFrame, web workers/JSI libraries or native modules? How do you detect dropped frames?*

In React Native, the JavaScript thread handles most application logic, while the native UI thread handles native view operations, layout-related work, and rendering coordination. If the JavaScript thread is blocked for too long, interactions such as button presses, JavaScript-driven animations, and navigation responses can become delayed, even when the native UI thread is still responsive.

**First, I identify what blocks the JavaScript thread.** Common examples include parsing large JSON responses, running heavy loops, performing complex calculations, filtering thousands of records synchronously, and making synchronous storage or native calls. For example, if I receive thousands of records and process the entire dataset inside a single synchronous loop, JavaScript cannot respond to other events until that loop finishes. This can make the screen appear frozen.

To avoid this, I reduce unnecessary computation, paginate large datasets, optimize algorithms, and avoid synchronous operations on the main JavaScript execution path. For local storage, I prefer asynchronous APIs such as AsyncStorage rather than synchronous storage reads that block JavaScript.

**Second, I schedule non-critical work appropriately.** I can use `requestAnimationFrame` to schedule work around animation frames, but I avoid placing a large computation inside it because the computation can still block JavaScript. For expensive work that does not need to happen immediately, I can defer it until after navigation transitions or initial interactions.

Historically, `InteractionManager.runAfterInteractions()` was used for this purpose. However, it is deprecated in current React Native documentation, so I would consider alternatives such as scheduling work in smaller chunks or using an appropriate task scheduler supported by the project's React Native version. Deferring work changes when it runs; it does not make CPU-intensive work execute on another thread.

**Third, I move genuinely expensive computation off the JavaScript thread when necessary.** Depending on the project, I can use a background worker solution where supported, a native module, or a JSI-based library that performs expensive processing in native code or on a separate native thread. For example, image processing, cryptographic operations, or complex data transformations may be better handled by native implementations.

I choose the solution based on the workload. A native module does not automatically run on a background thread; its implementation must explicitly schedule the expensive work away from the UI thread. Similarly, JSI provides direct JavaScript-to-native integration, but it does not automatically make synchronous computations non-blocking.

**Fourth, I optimize React Native rendering and large lists.** I use `FlatList` instead of rendering thousands of items inside a `ScrollView`, optimize row components, use stable keys, and avoid unnecessary re-renders. I also use memoization where it provides measurable benefits. This reduces the amount of work JavaScript must perform when users scroll or interact with the application.

**Finally, I detect dropped frames and performance bottlenecks.** I reproduce the problem in a release build because development mode can introduce additional overhead. I use React Native DevTools and React Profiler to inspect JavaScript execution and component re-renders, and I use Android Studio's system profiling tools or Xcode Instruments to investigate native UI performance.

I also monitor frame rates and responsiveness during scrolling, animations, and navigation. If the JavaScript thread is overloaded, JavaScript-driven interactions and animations may stutter. If the native UI thread or rendering pipeline is overloaded, native rendering can also drop frames. I distinguish between these cases rather than assuming every performance issue originates in JavaScript.

Overall, my approach is to identify the blocking operation, reduce unnecessary work, schedule non-critical tasks appropriately, and move CPU-intensive operations off the JavaScript thread when needed. Then I verify the improvement through profiling and real-device testing rather than relying on assumptions.

*Q28. WebSockets in React Native. Design a reliable real-time connection, for example live order updates. How do you handle reconnect with exponential backoff, the app going to background (iOS suspends sockets), heartbeats/ping-pong, missed messages after reconnecting, and cleanup on unmount?*

For a real-time feature like live order updates, I use WebSockets to maintain a persistent, bidirectional connection between the React Native application and the backend. The main challenge is not just establishing the connection; it is keeping the connection reliable across network failures, background transitions, and missed messages.

**First, I design the connection lifecycle.** When the user opens the order screen, the app establishes an authenticated WebSocket connection and subscribes to updates for the relevant order or restaurant. When the server sends an event, such as an order changing from `Preparing` to `Ready`, the client validates the event and updates the application state. I keep connection management separate from individual screen components so that multiple screens do not create duplicate connections unnecessarily.

**Second, I implement automatic reconnection with exponential backoff.** Mobile networks can disconnect temporarily when users switch between Wi-Fi and mobile data or enter areas with poor connectivity. Instead of reconnecting continuously, I use exponential backoff with jitter. For example, the retry delay can start at one second and increase to two, four, and eight seconds, up to a configured maximum. Random jitter prevents many clients from reconnecting simultaneously. I also reset the retry counter after a stable connection is re-established and stop retrying when the user logs out.

**Third, I handle background and foreground transitions.** On iOS, the operating system can suspend an application in the background, so I never assume a WebSocket will remain active. I listen to React Native's `AppState` changes and use an appropriate lifecycle strategy. When the app returns to the foreground, I check the connection and reconnect if necessary. Depending on the product requirements, I may close the socket when the app becomes inactive and restore it when the user returns. If the user needs alerts while the app is suspended, I use push notifications rather than relying on a WebSocket to keep running in the background.

**Fourth, I detect stale connections using heartbeats.** A socket can appear connected even when the underlying network is no longer usable. I use ping/pong heartbeats, where supported by the WebSocket implementation, to verify that the connection is responsive. If the server does not respond within a configured timeout, I consider the connection unhealthy, close it, and trigger reconnection. If the client library does not expose protocol-level ping/pong, I can implement an application-level heartbeat message and response.

**Fifth, I recover missed messages after reconnecting.** Reconnection alone does not guarantee that the client receives every event sent while it was offline. For reliable updates, I use event IDs or sequence numbers and maintain the last successfully processed event ID. After reconnecting, the client requests missed events from the backend using that cursor, or fetches the latest order state through an API. The server replays available events, and the client processes them in the correct order while ignoring duplicates. For critical order updates, I treat the backend as the source of truth rather than relying exclusively on in-memory client state.

**Sixth, I clean up connections properly.** When a component that owns a connection unmounts, I remove its event listeners, clear heartbeat and reconnection timers, unsubscribe from relevant events, and close the socket if that component owns it. If the connection is shared across the application, I use a central service or context to manage its lifecycle and avoid closing a connection that other screens still need. I also prevent stale callbacks from updating state after a screen has unmounted.

**Finally, I test failure scenarios.** I test Wi-Fi disconnection, switching networks, app backgrounding, token expiration, server restarts, duplicate events, and reconnecting after several minutes offline. I verify that the order state eventually matches the backend and that reconnection does not create duplicate subscriptions or duplicate updates.

Overall, my approach is to combine exponential backoff with jitter, lifecycle-aware reconnection, heartbeats, event replay or state resynchronization, and proper cleanup. This makes real-time updates resilient to the network and application lifecycle problems that are common in mobile environments.

*Q29. Testing strategy. What do you test with Jest and React Native Testing Library vs E2E (Detox/Maestro)? How do you mock native modules, navigation and async storage? What makes a test brittle (implementation details vs user behaviour), and how do you test a custom hook with async effects?*

In React Native, I use a combination of unit tests, component tests, and end-to-end tests. My goal is to verify business logic, user interactions, and complete application flows without making the test suite unnecessarily slow or difficult to maintain.

**First, for Jest and React Native Testing Library,** I use Jest as the test runner and assertion framework, while React Native Testing Library helps me render components and test them from the user's perspective. I use these tools to test component rendering, form validation, button interactions, API success and failure states, conditional UI, and business logic.

For example, in a login screen, I test whether validation errors appear for invalid input, whether the loading indicator appears during submission, and whether the correct error message is displayed when the API request fails. I focus on observable behaviour rather than checking the component's internal state or implementation.

**Second, I use E2E testing for complete user journeys.** Tools like Detox and Maestro allow me to test the application running on a simulator or real device. I use E2E tests for critical flows such as login, navigation between screens, creating an order, completing a booking, and handling important permission or authentication scenarios.

Jest and React Native Testing Library are generally faster and easier to run frequently, whereas E2E tests validate integration between the UI, navigation, native platform behaviour, and backend or test services. I do not try to cover every possible condition with E2E tests because they take longer to execute and can be more sensitive to environmental issues.

**Third, I mock native modules and external dependencies.** React Native applications depend on native APIs that may not be available in Jest's JavaScript testing environment. For example, I mock camera access, notifications, device information, and other native integrations when testing components that depend on them.

For AsyncStorage, I use its Jest mock implementation:

```javascript
jest.mock(
  '@react-native-async-storage/async-storage',
  () =>
    require(
      '@react-native-async-storage/async-storage/jest/async-storage-mock'
    )
);
```

For API requests, I mock the service layer or network responses so tests remain deterministic and do not depend on a live backend. For navigation, I either render the component within a suitable navigation test setup or mock navigation functions when I only need to verify that a user action triggers the expected navigation behaviour.

I avoid mocking everything indiscriminately because excessive mocking can hide integration problems.

**Fourth, I avoid brittle tests.** A test becomes brittle when it depends on implementation details instead of actual user behaviour. For example, checking a component's internal state, relying on a specific child-component hierarchy, or selecting elements through deeply nested implementation-specific test IDs can cause tests to fail after harmless refactoring.

Instead, I prefer accessible queries such as `getByRole`, `getByLabelText`, and `getByText`, where appropriate. I test what the user sees and does, such as entering credentials and pressing the Login button. I use `testID` when necessary, particularly for elements that are difficult to identify accessibly, but I do not make it the default for every assertion.

**Fifth, I test custom hooks that perform asynchronous effects.** For a hook that fetches order details when an order ID changes, I use `renderHook` from React Native Testing Library, mock the API dependency, and wait for the asynchronous result using `waitFor` or the appropriate asynchronous utilities.

I test the initial loading state, successful response, error state, and behaviour when the input changes. I also verify that stale requests do not overwrite newer results and that effects clean up correctly when the hook unmounts.

For example:

```javascript
import { renderHook, waitFor } from
  '@testing-library/react-native';

import { useOrder } from './useOrder';
import { getOrder } from './orderService';

jest.mock('./orderService', () => ({
  getOrder: jest.fn(),
}));

test('loads order details successfully', async () => {
  getOrder.mockResolvedValue({
    id: '123',
    status: 'Preparing',
  });

  const { result } = renderHook(() => useOrder('123'));

  expect(result.current.loading).toBe(true);

  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });

  expect(result.current.order).toEqual({
    id: '123',
    status: 'Preparing',
  });
});
```

In a real project, I would also test rejected requests, changing order IDs, and cleanup or cancellation behaviour where applicable. I use asynchronous utilities instead of arbitrary timeouts to make tests more reliable.

**Finally, I integrate testing into the development workflow.** I run unit and component tests regularly, include critical E2E flows in CI where practical, and prioritize regression tests for bugs that have occurred in production. I also test on both iOS and Android because a passing JavaScript test suite does not guarantee that native behaviour works correctly on both platforms.

Overall, my strategy is to use Jest and React Native Testing Library for fast, focused behavioural tests, and Detox or Maestro for critical end-to-end journeys. I keep tests deterministic, mock external dependencies at appropriate boundaries, and prioritize user-visible behaviour over implementation details.

*Q30. Memory leaks and app stability. List the common leak sources (uncleared timers, event listeners, subscriptions, closures in useEffect, state updates after unmount). How do you find them with Flipper/Xcode Instruments/Android Profiler, and how do you handle an Android app being killed in the background and restored with lost state?*

In React Native, memory leaks occur when objects, event listeners, timers, or subscriptions remain in memory even though the application no longer needs them. Over time, this can increase memory usage, slow down the app, cause UI freezes, or lead to crashes, especially on low-end Android devices.

**First, I identify the common sources of memory leaks.** One common issue is forgetting to clear timers created with `setInterval` or `setTimeout`. For example, if a screen starts polling for live order updates but does not clear the interval when it unmounts, the timer may continue running unnecessarily.

Other common sources include event listeners that are never removed, WebSocket subscriptions that remain active, native event subscriptions that are not cleaned up, and closures that retain references to large objects. Another issue is starting asynchronous operations in `useEffect` and allowing their results to update state after the component has unmounted. Although a state update after unmount does not automatically prove a memory leak, it can indicate incorrect lifecycle handling.

**Second, I clean up side effects properly.** Whenever I create a timer, listener, or subscription inside `useEffect`, I return a cleanup function. For example:

```javascript
useEffect(() => {
  const intervalId = setInterval(() => {
    fetchOrderStatus();
  }, 5000);

  const subscription = AppState.addEventListener(
    'change',
    handleAppStateChange
  );

  return () => {
    clearInterval(intervalId);
    subscription.remove();
  };
}, []);
```

For asynchronous API requests, I use `AbortController` when the API client supports cancellation, or another cancellation mechanism to prevent obsolete requests from updating the UI. I also ensure that WebSocket connections and event subscriptions are released according to their ownership and lifecycle.

**Third, I investigate memory issues using profiling tools.** I use Flipper when the project's React Native version and installed plugins support the required debugging features. I can inspect available logs, network requests, and other diagnostic information, although Flipper is not a universal memory-leak detector for every modern React Native setup.

On iOS, I use Xcode Instruments, particularly Allocations and Leaks, to inspect memory growth, object lifetimes, and potential leaks. On Android, I use Android Studio's Memory Profiler to monitor heap usage, garbage collection, allocations, and memory growth over time. I reproduce the issue by repeatedly opening and closing the same screen and checking whether memory usage returns to a stable baseline.

I also test release builds because development mode and debugging tools can introduce additional memory and performance overhead.

**Fourth, I handle Android background process termination and state restoration.** Android can kill an application's process while it is in the background to reclaim memory. I never assume that React state, component instances, or in-memory variables will survive this event.

For important state, I distinguish between persistent data and temporary UI state. I persist information such as authentication credentials using an appropriate secure-storage solution, and I can store recoverable application data or navigation information using AsyncStorage or another suitable persistence mechanism. Sensitive tokens should not be stored in plain, unencrypted storage.

When the app starts again, I restore the required state, validate the user's session, and fetch the latest data from the backend. For example, if the user was viewing order `123` before Android killed the process, I can restore the order identifier and reopen that screen after initialization, then retrieve the current order status from the server.

I avoid persisting everything blindly. Temporary states such as an animation's current frame do not need restoration, while unsaved form input may need to be preserved depending on the product requirements.

**Fifth, I make restoration resilient.** I account for missing or corrupted persisted data, expired authentication sessions, deleted records, and failed network requests. I show an appropriate loading state during restoration and fall back to a safe screen if the original destination can no longer be opened. I also distinguish normal backgrounding from a fresh process launch rather than assuming every transition triggers a complete application restart.

**Finally, I validate app stability under realistic conditions.** I repeatedly navigate between screens, start and stop subscriptions, background and reopen the app, and test Android process recreation. I monitor memory usage and verify that timers, listeners, and network connections do not accumulate across repeated screen visits.

Overall, my approach is to clean up every side effect, profile memory growth rather than guessing, and persist only the state necessary to recover after process termination. This improves long-term stability and makes the application more reliable on memory-constrained devices.