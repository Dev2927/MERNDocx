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