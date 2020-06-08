---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: ScandiPWA plugins
description: Definitive guide for ScandiPWA plugin functionality.

# Micro navigation
micro_nav:
  enabled: true
  url: '/docs/development'
  title: Development

# Page navigation
page_nav:
    prev:
        content: Extension mechanism
        url: '/docs/development/extension'
    next:
        content: Debugging
        url: '/docs/development/debugging'

---

ScandiPWA v3 (currently in beta) supports frontend plugins, which allow reusing one piece of code throughout a lot of completely different projects. This guide is devoted to explaining all the functionality related to plugins.


## Watch an explanation video

### Implementing an extension from scratch
<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/9f6rpIrlNMk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Implementing an extension from customisation
<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/N2TJJbSDTbM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Extension's file structure

ScandiPWA extension is M2 composer package with one extra directory - `scandipwa`. Inside of it ScandiPWA frontend-related functionality is stored. Any other M2 related folders with backend-related functionality can be stored in such package. You see `etc` and `Model` in this example, but these are not the only possible ones.

All directories inside of `scandipwa` are optional, nevertheless following the provided structure is mandatory. You see `app` and `sw` subdirectories in it, these folders' structure is the same as you see in `vendor/scandipwa/source/src/(app|sw)` and they have the same meaning: inside of them are all parts that your extension requires: extra components, queries, routes etc.

`Plugin` directory contains plugin definitions: which functionality parts are you plugging into and how you want to change their behaviour. More about that below.

```bash
📦my-awesome-extension
 ┣ 📂src
 ┃ ┣ 📂etc
 ┃ ┃ ┗ # ...
 ┃ ┣ 📂Model
 ┃ ┃ ┗ # ...
 ┃ ┗ 📂scandipwa  # Frontend-related functionality
 ┃   ┣ 📂 app     # Application logic
 ┃   ┃ ┣ 📂component
 ┃   ┃ ┣ 📂query
 ┃   ┃ ┣ 📂route
 ┃   ┃ ┣ 📂store
 ┃   ┃ ┣ 📂util
 ┃   ┃ ┗ 📂plugin # Plugging logic declarations
 ┃   ┃   ┗ 📜<SourceComponentName>.plugin.js
 ┃   ┗ 📂 sw      # Service Worker
 ┃     ┣ 📂handler
 ┃     ┣ 📂util
 ┃     ┗ 📂plugin # Plugging logic declarations
 ┃       ┗ 📜<SourceServiceWorkerFileName>.plugin.js
 ┗ 📜composer.json
```

## A step-by-step algorithm of creating a simple extension

This document is an addition to the video tutorial above. You are welcome to watch it in order to learn from an example. Start with understanding how you wish to change ScandiPWA logic. Find the places which need modification.

1. Create a `localmodules` directory in the magento root of your application. Then create a composer package there. Inside of it, in `src/scandipwa/plugin`, create `ComponentName.plugin.js` file, where `ComponentName` is name of the component you'd like to modify.

2. In there, you should create a class `ComponentNamePlugin`, its members are meant to wrap around the `SampleClass` members and be ran instead of the original members. It is essential that wrapper members must be arrow functions, otherwise your plugin's context will not be available from inside of them.

>**Note: It is recommended to stick to the naming convention regarding the arguments of these functions.**

Each member which wraps around a **_function_** has the following arguments.
- `args` is an array of original arguments that are passed to the function this one is wrapped around.

- `callback` is function that calls the next plugin if there is one, or the original method, when all plugins defined for this method are applied.

- `instance` references the object you are plugging into.

Each member which wraps around a **_property_**  has the following arguments:

- `originalMember` is the member you are plugging into.

- `instance` references the object you are plugging into.

```javascript
    // This wraps around the member function, logs the arguments and adds one to the first argument
    // It is essential that wrapper function is an arrow function!
    // Otherwise other member functions will not be available from it.
    aroundFunction = (args, callback, instance) => {
        console.log(args); // [ ...arguments ]
        args[0] += 1;

        // The context must ALWAYS be passed to the callback, using 'apply' for that
        // Note that the original member (or the next plugin) will be called with changed arguments
        callback.apply(instance, args);
    }

    // This wraps around a property
    property = (originalMember, instance) => {
        return {
            ...originalMember,
            // And adds this new value to it
            someAddedValue: 'new value!'
        }
    }
```

3. Create the configuration in the bottom of `.plugin.js` file. This object must be a default export. The following things are defined in this configuration:

- Which namespace to modify

- How what exactly in the namespace would you like to modify

  - _'member-function'_ plugins intercept calls to instance members. These plugins are used to change the behavior of member functions, which are called on instance.

  - _'member-property'_ is an approach to change **properties**, which are not available on prototypes, e.g. state in a way it's defined throughout the ScandiPWA (`state = { ... };`).

  - _'static-member'_ plugins enable changing classes' **static members**.

  - _'function'_ is an approach to change **functions** which are not class members, e.g. `mapStateToProps` or `mapDispatchToProps`.

- Name of the member to modify

- A position, in which this plugin will be called. There may be multiple plugins for a single member if there are several extensions active in the application. The closer the position to 0 - the sooner it is called. The higher a position - the later. Non-unique.

> **Note: you can create class members that do not exist in the original classes and they will be called as you'd expect writing them directly in the class, e.g. `componentDidUpdate`. Use _member-function_ plugins to do that!**

Configuration must follow this format:

```javascript
const config = {
    namespace: {
        'member-function': {
            '<name>': [
                {
                    position: A, // number
                    implementation: B // function
                }
            ]
        },
        'member-property': {
            '<name>': [
                {
                    position: C,
                    implementation: D
                }
            ]
        },
        'static-member': {
            '<name>': [
                {
                    position: E,
                    implementation: F
                }
            ]
        },
        'function': {
            // Reduced structure for functions
            // because function is the only member of its namespace
            {
                position: G,
                implementation: H
            }
        }
    }
}
```

4. Activate your plugin. In the FE root of your theme, there is a file called `scandipwa.json`. It is responsible for theme's configuration. Active plugins should be defined there.

Contents:

- `<extension name>`: should be picked by you, it is not related to any functionality, just denotes which plugin files are meant for which extension. Put anything you like here.

- `<path>`: relative path from Magento root to `.plugin.js` file. Not that it's mandatory to have these paths in an array, even if only one is required for the extension.

The format for the 'extensions' block of this file is the following:
```javascript
{
    // ...
    "extensions": {
        "<extension name>": [
            "<path1>",
            "<path2>"
        ]
    }
    // ...
}
```

## Overriding extensions

### Watch an explanation video

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/vG5eZcvCq48" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Approach 1: Fallback Plugin

1. Creating a new file

Imagine you have to override the following file:

```bash
.../awesome-extension-provider/paypal-graphql/src/scandipwa/app/component/PayPal/PayPal.component.js
```

In order to acomplish that all you need to do is to create a file with the following path:

```bash
.../pwa/src/plugin/awesome-extension-provider/paypal-graphql/app/component/PayPal/PayPal.component.js
```

The pattern is: for file with the following path

```bash
.../<vendor>/<extension>/src/scandipwa/<PATH>
```

You need to create a file with the following path, that is going to be taken in place of original file. Notice that to reduce the path you don't need to reference src/scandipwa folders each and every time. They'd always be there, that's why we removed them from the modified path to simplify file structure.

```bash
.../pwa/src/plugin/<vendor>/<extension>/<PATH>
```

2. Retrieving the original functionality

You can import anything exported from the original plugin file to extend it, just as when extending source theme files. To do that you can reference the original file by relative path, but that is a bit too long. To make it simpler, a different approach has been implemented. Now you can import original functionality using a generated alias, that depends on directories' names and consists of vendor name and extension name, written in PascalCase and separated by underscore.

So if you need to import something from this file:

```bash
.../awesome-extension-provider/paypal-graphql/src/scandipwa/app/component/PayPal/index.js
```

You can do it by referencing the original file by following alias, anywhere throughout the application:

```javascript
import PayPal from 'AwesomeExtensionProvider_PaypalGraphql/app/component/PayPal';
// instead of
import PayPal from '../../../<some more iterations>/awesome-extension-provider/paypal-graphql/src/scandipwa/app/component/PayPal';
```

Notice how `src/scandipwa` part disappeared in here just as in the first approach.

> **Note: remember that these aliases are case-sensitive. PayPalGraphQL instead of PaypalGraphql will throw errors**

3. Extending

See more on extending functionality with such approach (using ScandiPWA Fallback plugin) in the [extension guide](/docs/development/extension/). All extensions-related specifics have been described in this article, follow them when using approaches described in video and text guides following the link.


### Approach 2: Plugins!

ScandiPWA allows **plugging into plugins**. All classes that your plugin requires should be assigned namespaces by wrapping them into the `middleware` function. The only exception is that plugin class in `.plugin.js` file cannot be plugged into due to configuration builder's limitations. It still can be overriden as described above though.