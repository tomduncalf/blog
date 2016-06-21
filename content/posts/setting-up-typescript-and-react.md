+++
date = "2016-06-21T19:46:14+01:00"
draft = false
title = "Setting up a new Typescript 1.9 and React project"
+++

# Introduction 

This post is a brain dump of the steps required to set up a Typescript and React project with some explanatory notes – I intend to write about working with Typescript and React in a real world project in more detail soon. Some knowledge of React and of the basics of Typescript is assumed.

These instructions will guide you through setting up a new project with Typescript, React, Webpack and Babel – neither Webpack nor Babel are required to work with Typescript, as Typescript can transpile ES6 to ES5 and do some degree of bundling itself; but using them enables Hot Module Reloading, and also allows you to run other Babel and Webpack plugins on the compiled output if desired.

The resulting project template is available on Github at https://github.com/tomduncalf/ts-react-template.

# Editor setup

## Atom

Atom users will need to install the `atom-typescript` plugin, which is excellent, but currently doesn't support some of the features in Typescript 1.9, and seems to be [lacking an active project leader](https://github.com/TypeStrong/atom-typescript/pull/849). 

## Sublime

Sublime users will need the official Microsoft `Typescript` package (https://github.com/Microsoft/TypeScript-Sublime-Plugin), which is also very good.

## Visual Studio Code

Visual Studio Code users don't need to install anything – it's already set up for Typescript.

## Recommendation

All of these editor/plugin combinations work well, and will automatically detect Typescript projects and offer the appropriate autocompletion and error highlighting.

Visual Studio Code has the best Typescript integration (as you might expect from Microsoft!) and is noticeably faster than Atom. In the past, I was put off using VS Code due to the lack of tabs, but as these are now available in the latest [Insiders build](http://code.visualstudio.com/Download#insiders) by [enabling a flag](https://github.com/Microsoft/vscode-docs/blob/vnext/release-notes/June_2016.md#tabs), it's now my recommended editor. 

The Atom and Sublime plugins offer broadly similar functionality, and I have always used Atom in the past, but I can't fully recommend it at the minute due to the uncertain status of the `atom-typescript` plugin and its lack of support for Typescript 1.9 features (although this is fair enough, as it is still a beta). The Sublime plugin is an official Microsoft product, so shouldn't suffer from the same issues, and is a great choice if you are already a Sublime user.

*Note:* If you are using Visual Studio Code, see step 1 for additional setup required to make it use the `typescript@next` compiler.

## Linting

If you want to lint your code, you'll want to install a `tslint` plugin for your editor and setup `tslint` – see the project page at https://github.com/palantir/tslint/ for more information.

# Basic project setup

1. **Install Typescript globally** - this is optional, but it's handy to have the Typescript compiler `tsc` available globally at the command line. We will use `typescript@next` (version 1.9.x) even though it is still in beta, as it supports installing type declarations from `npm` rather than having to use the `typings` tool, which is [the future of working with type declaration files](https://blogs.msdn.microsoft.com/typescript/2016/06/15/the-future-of-declaration-files/) and makes life much easier.

    ```bash
    npm install -g typescript@next
    ```

    ## Note for VS Code users

    If you are using VS Code, you need to tell it to use the Typescript compiler that you installed with `npm`, otherwise you'll get syntax errors as the internal VS Code Typescript compiler is v1.8 and so doesn't support type declarations installed from `npm`. 

    To do so, you'll need to know where `npm` has installed your global packages to – you can check this with:

    ```bash
    npm list -g | head -n1
    ```

    Then open up your user settings in VS Code (`Cmd` + `,` on a Mac) and add a line to your user settings, pointing the `typescript.tsdk` option to the `node_modules/typescript/lib` directory in this location – for example, if the global packages are installed to `/Users/td/.nvm/versions/node/v5.5.0/lib/`, then add the following and restart Code:

    ```
    {
      "typescript.tsdk": "/Users/td/.nvm/versions/node/v5.5.0/lib/node_modules/typescript/lib"
    }
    ```

2. **Create a new project**:

    ```bash
    mkdir my-project && cd my-project
    git init
    npm init -y
    ```

3. **Initialise the project for Typescript** - install Typescript as a dev dependency and create a skeleton `tsconfig.json`:

    ```bash
    npm i -D typescript@next
    tsc --init
    ```

# Setting up tsconfig.json

1. `tsconfig.json` controls the behaviour of the Typescript compiler (`tsc`). There are a few settings which are useful for React and Babel projects which differ from the defaults. **Open `tsconfig.json` and change it to the following**:

    ```
    {
      "compilerOptions": {
        "module": "es6",
        "target": "es6",
        "moduleResolution": "node",
        "baseUrl": "src",
        "allowSyntheticDefaultImports": true,
        "noImplicitAny": false,
        "sourceMap": true,
        "outDir": "ts-build",
        "jsx": "preserve"
      },
      "exclude": [
        "node_modules"
      ]
    }
    ```

    An explanation of what this all means:

    -  `"module": "es6"` tells `tsc` to output code which uses the ES6 module spec (i.e. `import` statements). It's also possible to set this to e.g. `commonjs`, in which case `tsc` will convert your code to that module spec, but as we will be putting our compiled JS code through Webpack, we can keep it as ES6 and let Webpack handle the module bundling.

    -  `"target": "es6"` tells `tsc` to output ES6 rather than ES5 Javascript code. We want this as we will be running the compiled JS through Babel, and keeping ES6 code intact after compilation can be useful for some Babel plugins (e.g. [`transform-react-stateless-component-name`](https://www.npmjs.com/package/babel-plugin-transform-react-stateless-component-name), which automatically names stateless components, will only pick up on arrow functions). 
      
        This also allows us to use `async`/`await`, which is understood by Typescript but not currently able to be transpiled – instead, we can use Babel to handle the transpilation of `async` code. If we weren't using Babel, this setting could be omitted or set to `es5`, to make `tsc` do the transpilation of ES6 to ES5 itself.

    -  `"moduleResolution": "node"` tells `tsc` to use the Node module resolution strategy. This allows Typescript to load type declarations supplied alongside `npm` packages (e.g. MobX includes its own type declarations in the main `mobx` package), and with the `baseUrl` option, allows us to use absolute-style imports for local modules.

    -  `"baseUrl": "src"` tells `tsc` to look in `src` for any modules we import that aren't found in `node_modules`. This allows us to write absolute-style imports for local modules, e.g. `import Whatever from 'components/Whatever'` rather than `import Whatever from '../components/Whatever'`, which is great for one's sanity.

        **Note that this settings does not work with the current version of `atom-typescript`** (see https://github.com/TypeStrong/atom-typescript/pull/849) – remove this line from your `tsconfig.json` if you want to use Atom. You can get an approximation of the absolute-style import by using `"moduleResolution": "classic"` (which will walk up the directory tree until a match is found, so not the same behaviour, but similar end result in many cases), but this breaks the ability to automatically import type declarations supplied with `npm` packages.

    -  `"allowSyntheticDefaultImports": true` allows us to use ES6 `import` syntax for `npm` modules which don't have a default export.

    -  `"noImplicitAny": false` tells `tsc` not to warn us if any variables are inferred as having a type of `any`. It's actually probably good practice to set this to `true`, but it does mean you'll potentially have to be more liberal with type annotations.

    - `"sourceMap": true` tells `tsc` to output a source map, which enables easier debugging from the browser as it can tell you where in the original `.ts` source file an error occurred, rather than just in the compiled `.js`.

    - `"outDir": "ts-build"` tells `tsc` to output the compiled `.js` files to a directory called `ts-build` (which can be `.gitignore`d). The default is to output them alongside the original `.ts` source files, but this gets messy. It should be noted that most of the time, we won't be outputting the compiled `.js` to disk, as the Webpack loader will do the compilation in memory, but it is sometimes useful to be able to invoke `tsc` manually and inspect the compiled output.

    - `"jsx": "preserve"` tells `tsc` to leave JSX code as it is, meaning that something else (in this case, Babel) is responsible for compiling it down to `React.createElement` function calls. It is possible to set this to `"react"` instead, which will cause `tsc` to output `React.createElement` calls directly, but it can be useful to have the raw JSX available to Babel, e.g. for plugins to process.

    - We `exclude` `node_modules` as we don't want `tsc` to try and compile anything it finds in there – it's alternatively possible to explicitly `include` files for compilation (or use the non-standard `filesGlob` option, which allows you to specify wildcard patterns, supported by the Atom plugin and https://github.com/TypeStrong/tsconfig)

# Adding React to the project

1. **Install React to `node_modules`**:

    ```bash
    npm i -S react
    ```

2. To demonstrate what we are about to do, **`mkdir src` and create a file `index.tsx` in there** containing a basic (stateless) React component:

    ```javascript
    import * as React from 'react';

    export default () => <div>Hello world</div>;
    ```

    If your editor is set up correctly, you should already see that it has highlighted an issue with the `import * as React from 'react';` line, but to demonstrate further, run `tsc` from your project root and you should get the following output:

    ```bash
    src/index.tsx(1,24): error TS2307: Cannot find module 'react'.
    ```

    The issue here is that the Typescript type declarations for React aren't installed, and so Typescript says it can't find the module, as type declarations are required for any modules that are `import`ed (the error message doesn't make this especially clear!).

    Something interesting to note, however, is that `tsc` has created a `ts-build` directory and written `index.jsx` there, with sensible contents – in general, the Typescript compiler will try and emit code even if there are errors, as long as they aren't fatal (although this can be disabled in `tsconfig.json` with the `noEmitOnError` option).

3. **Before installing type declarations for React, we first need to check if they exist** – the majority of type declaration files are created by the community rather than the library authors, but most popular libraries are covered. 

    The type declaration system has been through several iterations of tooling (first `tsd`, then `typings`) but is now moving to be purely `npm` based. The original definitions live in the massive [DefinitelyTyped Github repo](https://github.com/DefinitelyTyped/DefinitelyTyped), and are automatically synced to `npm`.

    It seems that the only way to check if a package has type declarations in `npm` is to search at http://microsoft.github.io/TypeSearch/ – type in `react`, and indeed it does have a definition, hosted on `npm` at https://www.npmjs.com/package/@types/react. 

    Previous iterations of the typing system had the ability to search from the command line, which is preferable in most cases, so hopefully someone will fill this gap for `npm`-based type declarations – for now, you could install `typings` (https://github.com/typings/typings) and use that to search DefinitelyTyped, or just use Google/GitHub/npm search.

4. Now we know the React type declarations exist at `@types/react`, **we can install them with `npm`**:

    ```bash
    npm i -D @types/react
    ```

    And now both `tsc` and your editor should be happy with the React import. At this point, you should also be able to get a sense for the autocompletion Typescript enables, by typing `React.` somewhere in your `index.tsx` file and seeing the autocomplete dropdown with the correct options.

5. **We also need the `react-dom` package**, which can be installed in the same way:

    ```bash
    npm i -S react-dom
    npm i -D @types/react-dom
    ```

    To my mind, it makes sense for type declarations to be in `devDependencies` (so `npm -D` rather than `npm -S`). Also note that the structure of the type declaration package name is always `@types/<npm_package_name>`, so you could take advantage of this naming scheme and try installing types for a given package based on its name rather than using the search in most cases.

    One other thing worth noting is that the current version of the React typings is `0.14` whereas we are using React `0.15` – this is one downside of type declarations being maintained by the community, but in most cases it doesn't present too much of a problem (as long as the API hasn't changed much). It is of course possible to put in a PR to DefinitelyTyped to fix the type defs if they are out of sync (or to augment them with local type declaration "overrides", or in extreme cases, to import the module without any typings – more on that later).

# Setting up Webpack and Babel

As mentioned above, we're using Babel to transpile to ES6 output from Typescript so we can take advantage of the Babel plugin ecosystem, and Webpack as our module bundler; so we need to setup Webpack to invoke the Typescript loader and then pass the output to Babel. I won't go into too much detail on the Webpack setup as it's a huge topic in itself!

1. **Install Webpack itself**, and the handy notifier plugin which will notify you of the build status with your system notifier (especially useful with Typescript as the code will be compiled every time you save a file, so this surfaces compile errors much quicker than watching the terminal):

    ```bash
    npm i -D webpack webpack-notifier
    ```

2. **Install the Webpack Typescript loader**, so Webpack can handle compiling Typescript as part of the bundling process:

    ```bash
    npm i -D ts-loader
    ```

3. **Install Babel itself, the Babel Webpack loader, and a few presets** so it can understand the ES6 and JSX code that the Typescript compiler will output (I'm not sure if `stage-0` is strictly necessary, `stage-3` might be enough for `async`/`await` support):

    ```bash
    npm i -D babel-core babel-loader babel-preset-es2015 babel-preset-react babel-preset-stage-0
    ```

4. **Create a `.babelrc` file** in the project root to tell Babel to use the presets we just installed:

    ```
    {
      "presets": ["es2015", "react", "stage-0"]
    }
    ```

5. **Create a Webpack config** in `config/webpack.config.js` to tell Webpack how to build and bundle the project. I've added comments explaining what is going on inline:

    ```javascript
    var webpack = require('webpack');
    var path = require('path');
    var WebpackNotifierPlugin = require('webpack-notifier');

    module.exports = {
      devtool: 'eval',
      // This will be our app's entry point (webpack will look for it in the 'src' directory due to the modulesDirectory setting below). Feel free to change as desired.
      entry: [
        'index.tsx'
      ],
      // Output the bundled JS to dist/app.js
      output: {
        filename: 'app.js',
        path: path.resolve('dist')
      },
      resolve: {
        // Look for modules in .ts(x) files first, then .js(x)
        extensions: ['', '.ts', '.tsx', '.js', '.jsx'],
        // Add 'src' to our modulesDirectories, as all our app code will live in there, so Webpack should look in there for modules
        modulesDirectories: ['src', 'node_modules'],
      },
      module: {
        loaders: [
          // .ts(x) files should first pass through the Typescript loader, and then through babel
          { test: /\.tsx?$/, loaders: ['babel', 'ts-loader'] }
        ]
      },
      plugins: [
        // Set up the notifier plugin - you can remove this (or set alwaysNotify false) if desired
        new WebpackNotifierPlugin({ alwaysNotify: true }),
      ]
    };
    ```

6. **Add a script to invoke Webpack** when you run `npm run build` to your `package.json`:

    ```
    "scripts": {
      "build": "webpack --config config/webpack.config.js"
    },
    ```

7. You should now be able to **build the project** with Webpack:

    ```
    npm run build
    ```

    and see that it has created an output file at `dist/app.js` (you probably want to `gitignore` the `dist` directory).

# Adding hot module reloading

Nearly there! All that remains is to add hot module reloading to the project. In reality, this is completely optional, but I find it invaluable for working on a React project, so would consider it part of any project's setup. It's also not really Typescript-specific, but there aren't many good guides out there for setting up the latest version of HMR, and setting it up gives us a chance to see a couple more tips on working with Typescript.

1. **Create an `index.html` file** which will load the bundled JS and give React somewhere to mount the application to. Create the file in the project root with the following contents:

    ```
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>My Typescript App</title>
    </head>
    <body>
      <div id="app"></div>
      <script src="dist/app.js"></script>
    </body>
    </html>
    ```

2. Next, **install the `react-hot-loader` `npm` package**. We are using the latest 3.0.0 beta version, as it supports stateless functional components.

    ```
    npm i -D react-hot-loader@3.0.0-beta.2
    ```

3. **Add the `react-hot-loader` plugin to `.babelrc`**, so it now contains:

    ```
    {
      "presets": ["es2015", "react", "stage-0"],
      "plugins": ["react-hot-loader/babel"]
    }
    ```

4. **Install the `webpack-dev-server` `npm` package**, to allow us to setup a dev server to work with the hot module reloading:

    ```
    npm i -D webpack-dev-server
    ```

5. **Create the dev server** in a new file in your project root, `server.js`, with the following contents (based on https://github.com/gaearon/react-hot-boilerplate/blob/next/server.js):

    ```
    var webpack = require('webpack');
    var WebpackDevServer = require('webpack-dev-server');
    var config = require('./config/webpack.config');

    new WebpackDevServer(webpack(config), {
      publicPath: config.output.publicPath,
      hot: true,
      historyApiFallback: true
    }).listen(3000, 'localhost', function (err, result) {
      if (err) {
        return console.log(err);
      }

      console.log('Listening at http://localhost:3000/');
    });
    ```

6. **Modify `config/webpack.config.js` to enable Webpack support for hot module reloading**, so it now contains the following (explanatory comments inline):

    ```
    var webpack = require('webpack');
    var path = require('path');
    var WebpackNotifierPlugin = require('webpack-notifier');

    module.exports = {
      devtool: 'eval',
      entry: [
        // Add the react hot loader entry point - in reality, you only want this in your dev Webpack config
        'react-hot-loader/patch',
        'webpack-dev-server/client?http://localhost:3000',
        'webpack/hot/only-dev-server',
        'index.tsx'
      ],
      output: {
        filename: 'app.js',
        publicPath: '/dist',
        path: path.resolve('dist')
      },
      resolve: {
        extensions: ['', '.ts', '.tsx', '.js', '.jsx'],
        modulesDirectories: ['src', 'node_modules'],
      },
      module: {
        loaders: [
          { test: /\.tsx?$/, loaders: ['babel', 'ts-loader'] }
        ]
      },
      plugins: [
        // Add the Webpack HMR plugin so it will notify the browser when the app code changes
        new webpack.HotModuleReplacementPlugin(),
        new WebpackNotifierPlugin({ alwaysNotify: true }),
      ]
    };
    ```

7. **Add a script to start the dev server** with `npm start` to your `package.json`:

    ```
    "scripts": {
      "build": "webpack --config config/webpack.config.js",
      "start": "node server.js"
    },
    ```

8. Finally, we need to **create a skeleton app set up in a suitable way for hot module reloading**. 

    Our entry point needs to be set up so that it can handle requests to hot reload the app. Open up `src/index.tsx` and change it to the following (explanatory comments inline):

    ```
    // Import React and React DOM
    import * as React from 'react';
    import { render } from 'react-dom';
    // Import the Hot Module Reloading App Container – more on why we use 'require' below
    const { AppContainer } = require('react-hot-loader');

    // Import our App container (which we will create in the next step)
    import App from 'containers/App';

    // Tell Typescript that there is a global variable called module - see below
    declare var module: { hot: any };

    // Get the root element from the HTML
    const rootEl = document.getElementById('app');

    // And render our App into it, inside the HMR App ontainer which handles the hot reloading
    render(
      <AppContainer>
        <App />
      </AppContainer>,
      rootEl
    );

    // Handle hot reloading requests from Webpack
    if (module.hot) {
      module.hot.accept('./containers/App', () => {
        // If we receive a HMR request for our App container, then reload it using require (we can't do this dynamically with import)
        const NextApp = require('./containers/App').default;

        // And render it into the root element again
        render(
          <AppContainer>
             <NextApp />
          </AppContainer>,
          rootEl
        );
      })
    }
    ```

    Effectively, this renders our app inside a special container (the `react-hot-loader` `AppContainer`), and then waits for Webpack to notify it of any changes to any of the app's files (which will trigger HMR requests, which end up bubbling up to the top level parent module, `containers/App`). When a change is detected, the whole `App` container is reloaded and the existing instance of it in the DOM is replaced with the new, modified one. 

    The trick here is that the `react-hot-loader` `AppContainer` takes care of persisting the state of components (whether local state or in Redux or similar) when it is reloaded, so in most cases, we don't lose where we were in the app.

    A quick note on a couple of things:

    ## Importing modules without type declarations

    ```
    const { AppContainer } = require('react-hot-loader');
    ```

    Here, we are using `require` rather than `import`. This is a useful trick to know, as it allows you to import an `npm` module without requiring any type declarations. 

    In this case, there are currently no type declarations for `react-hot-loader`, but if we use `require`, it is treated as being of `any` type. This can be handy for working with modules that don't have type declarations, particularly if you are just trying out modules to see if they are suitable and don't want worry about typing them.

    If you use `const Whatever = require('whatever');` rather than `import Whatever from 'whatever';`, Typescript won't complain about `Cannot find module` — although it also won't offer you any type safety when working with this module!

    See the next step for an important note on this – by default, Typescript doesn't know what `require` means (as it's not a built-in Javascript construct).

    ## Declaring global variables

    ```
    declare var module: { hot: any };
    ```

    `declare` is how we can tell Typescript about global variables that it doesn't already know about. Here, we are telling it that a global variable called `module` will exist, and it's type will be an object, which has a key called `hot`, who's value is of type `any` (which means it will not be type-checked – a bit of a cop out, and generally `any` types should be avoided, but this is okay for our purposes here).

9. If you run `tsc` at this point, you'll get an error:

    ```
    src/index.tsx(5,26): error TS2304: Cannot find name 'require'.
    ```

    This is because Typescript doesn't know what `require` means. There are a few ways round this, for example installing the `node` or `requirejs` type declarations, but the one recommended at https://github.com/TypeStrong/ts-loader#loading-other-resources-and-code-splitting which plays nicely with CSS modules is to create your own declaration for `require` — this also gives me the opportunity to demonstrate how we can create our own local type declarations for libraries!

    It's up to you where you would like to keep your local type declarations, but I would suggest a top-level directory named `type-declarations`. If you are using the default `exclude` pattern for `tsconfig.json`, any `*.ts` files in your entire project except those in `node_modules` will automatically be included in the compilation, so we don't need to explicitly tell Typescript where we have put the declarations.

    A type declaration file has the extension `.d.ts`, and can declare types either globally or inside a named module scope (which is how most third party library definitions are written, so the types only come into scope when the module is imported). Details of how type declarations work is beyond the scope of this guide, but for now **create a file named `require.d.ts` in a directory called `type-declarations` in the project root** with the following contents:

    ```
    declare var require: {
        (path: string): any;
        <T>(path: string): T;
        (paths: string[], callback: (...modules: any[]) => void): void;
        ensure: (paths: string[], callback: (require: <T>(path: string) => T) => void) => void;
    };
    ```

    Run `tsc` again, and the warning should be gone (with just one about `containers/App` being missing remaining) – we've told Typescript that calling `require` with a single string argument will return a variable of type `any`.

    This technique of creating local type declarations can also be useful if a libraries' declarations are out of date or inaccurate – you can add exported variables or additional properties to existing interfaces like so (although really it's best to submit these changes back to DefinitelyTyped if time allows):

    ```
    declare module "redux-form" {
        export var change: any

        export interface ReduxFormConfig {
            alwaysAsyncValidate?: boolean
        }
    }
    ```

10. The final step: we need to create our actual `App` container component. We'll just create a placeholder for now, but in reality this would be the root component of your app (e.g. where your `<Router>` lives).

    Create a new directory, `src/containers`, and create a new file in there, `App.tsx`, with the following contents:

    ```
    import * as React from 'react'

    export default () => <div>Hello world</div>
    ```

    Now you should be able to start your dev server:

    ```
    npm start
    ```

    and go to http://127.0.0.1:3000 to see the "Hello World" message! Update the `App.tsx` file to say something else, and you should see the app hot reload.

# Next steps

That's basically it in terms of project setup. As mentioned at the start, the complete template is available at https://github.com/tomduncalf/ts-react-template – I think it's helpful to go through each step and understand why it is required the first time, but in future, you can use the template you've created as a starting point for new projects.

In terms of next steps, it's really just a case of building your app as usual, but taking advantage of Typescript's type checking and editor integration. The only real pain point is likely to be working with type declarations for third party libraries, but using `npm` for the typings has made this easier, and there is always the option to import the library using `require` and bypassing the need for type declarations initially.

I intend to write more about working with Typescript and React soon, but hopefully this is a useful start – any questions, comments or feedback is very welcome either via the comments, or via Twitter or email.
