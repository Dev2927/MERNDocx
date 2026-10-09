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

🚨 1. Performance Issues
Communication between JavaScript and Native is slow because messages are passed asynchronously through the Bridge.
Scrolling may lag if UI updates arrive too late.

🚨 2. No Shared Memory
JavaScript and Native don’t share memory.
Instead of referencing a data object, React Native copies it in JSON format, which slows things down.

🚨 3. Heavy Native Module Loading
React Native loads all native modules at startup, even if some aren’t needed, which increases load time.

🚨 4. Harder to Contribute
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

Step 1: App Launches & UI Manager Starts
When you tap the app icon, the operating system launches the app and starts React Native.
The UI Manager initializes all native components like text fields, buttons, images, etc.

Step 2: JavaScript Bundle Loads
React Native loads MainBundle.js, which contains all JavaScript code for the app.
If you check the terminal while the app is starting, you’ll see logs showing “Loading MainBundle.js”.

Step 3: Views are Created via the Bridge (Old Architecture) or Fabric (New Architecture)
Old Architecture: JavaScript sends JSON messages to the Bridge to create UI elements.
New Architecture: Fabric directly interacts with native code, making UI rendering faster.

Step 4: Layout Calculation (Using Yoga Engine)
React Native doesn’t use standard CSS like the web.
Instead, it uses Yoga Engine to calculate layouts and positions of elements.
The Shadow Thread handles this layout processing.

Step 5: UI Appears on Screen
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
