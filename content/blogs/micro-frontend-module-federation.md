---
title: "Micro Frontend Module Federation"
date: 2022-12-05T16:50:28+08:00
author: "Feng Xue"
draft: true
---

Nowadays, our fleet-web is a monolith frontend project which includes everything. It works currently but have some issues with other teams, like team delivery and picap. Team Delivery is working on the tab `Delivery` in the fleet-web page, it uses the same framework (React) and similar technologies with fleet-web, but it does not have any intersection with fleet-web in business. It has its own development libraries and release plan. But since its codes are still in the same src folder with fleet-web, which causes team delivery still stays inside of the fleet-web's release lifecycles. Therefore, we need to separate it from fleet-web with Microfrontend technology.

## What is MicroFrontend

Microfrontend is a popular architechtural solution that helps large, complex monolith applications to be built as a collection of independently-developed and deployed microservices. This can improve the scalability, maintainability and flexibility of the application, making it easier to collaborate with a team of developers.

## What is Module federation

One way to implement microfrontend in the application is module federation, a plugin of the webpack 5. It connects different applications during build time and share modules with each other during runtime. This allows each module to define and manage its own components, while still being able to use common libraries by other modules.

Actually there are other popular frameworks doing the microfrontend, like single-spa, bit, qiankun. The reason I choose module federation is:
1. All the other frameworks are doing two things, organize the microfrontend services and connect these services together through importing. Since we already have our own application well-organized in a monorepo way, each module is dynamically imported with react new feature `React.lazy`. So what we need here actually is just the way of importing modules dynamically.
2. Module federation is just a plugin of webpack 5, we do not need to install another library to enable microfrontend. It also saves us time to maintain the extra library


## Migration with Module federation with Nx

Since we already installed [Nx](https://nx.dev/) in our repo to manage the monorepo, it's good to use module federation with Nx as well to include the new project in the management of Nx. Check [here](https://nx.dev/recipes/module-federation#module-federation-and-micro-frontends). Nx creates its own module federation module to simplify the procedure, if the whole project has not been created, it's good to use it.

```js
const withModuleFederation = require('@nrwl/react/module-federation');
const moduleFederationConfig = require('./module-federation.config');

module.exports = withModuleFederation({
  ...moduleFederationConfig,
});
```

### Configure host with original configuration

But our fleet-web project's webpack has already been well configured, it comes some conflict if we use the methods from Nx. So instead of that, let's use the original module federation configuration.

```js
const { ModuleFederationPlugin } = require('webpack').container
const deps = require('../../package.json').dependencies

...

module.exports = {
  entry: './src/bootstrap.tsx', // used to be entry: './src/index.tsx',
  //...
  plugins: [
    new ModuleFederationPlugin({
      name: 'fleet-web',
      shared: {
        react: {
          singleton: true,
          requiredVersion: deps['react'],
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
        },
        'react-redux': {
          singleton: true,
          requiredVersion: deps['react-redux'],
        },
        '@reduxjs/toolkit': {
          singleton: true,
          requiredVersion: deps['@reduxjs/toolkit'],
        },
      },
    }),
    ...
   ]};
```

Here we only define the name in module federation plugin because we will use dynamic importing later. And we update the entry from `./src/index.tsx` to `'./src/bootstrap.tsx'`. It's because module federation needs an entrance to import the modules.

Then let's create a new file `bootstrap.tsx` under folder `src` and fill the content with:

```js
import('./index')
```

### Configure remote

So the host part is finished, let's see how to create a remote project with Nx. In the root folder, run the following command. Here I named the project as `delivery-test`, feel free to change it if you want other name.

```bash
pnpm exec nx generate @nrwl/react:remote delivery-test
```

It would create the remote project under the fleet-web project with its own configuration. For the webpack, we would configure like below:

```js
require('dotenv').config()
const { join, resolve } = require('path')
const webpack = require('webpack')
const HTMLWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')
const packageConfig = require('../../package.json')
const { version } = packageConfig
const ReactRefreshPlugin = require('@pmmmwh/react-refresh-webpack-plugin')
const isDev = process.env.NODE_ENV === 'development'
const isProductionBuild = process.env.NODE_ENV === 'production'
const isDevDockerEnv = isDev && process.env.DEVELOPMENT_ENV === 'docker'

const { ModuleFederationPlugin } = require('webpack').container
const deps = packageConfig.dependencies

module.exports = {
  mode: isDev ? 'development' : 'production',
  entry: './src/main.ts',
  experiments: (() => {
    if (isDevDockerEnv) {
      return undefined
    }
    return {
      lazyCompilation: isDev
        ? {
            // Please see https://github.com/webpack/webpack/issues/12564#issuecomment-771877525 to understand why this is useful
            entries: false,
          }
        : false,
      cacheUnaffected: isDev,
      topLevelAwait: true,
      // outputModule: true,    Remember to not set outputModule as true
    }
  })(),
  output: {
    ...(isProductionBuild && {
      path: join(process.cwd(), 'dist/apps/delivery-test'),
      filename: 'static/js/[name].[contenthash].js',
      chunkFilename: 'static/js/[name].[contenthash].chunk.js',
    }),
    publicPath: 'auto',
  },
  context: __dirname,
  devtool: isDev ? 'eval-cheap-module-source-map' : 'nosources-source-map',
  devServer: isDev
    ? {
        historyApiFallback: true,
        host: '0.0.0.0',
        port: 9002,
        hot: true,
        liveReload: false,
        static: __dirname,
        devMiddleware: {
          publicPath: '/',
        },
        client: { overlay: false },
      }
    : undefined,
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
  },
  // ...
  plugins: [
    new ModuleFederationPlugin({
      name: 'delivery-test',
      filename: 'remoteEntry.js',
      library: { type: 'global', name: 'delivery_test' },
      exposes: {
        './Module': './src/remote-entry.ts',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: deps['react'],
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
        },
        'react-redux': {
          singleton: true,
          requiredVersion: deps['react-redux'],
        },
        '@reduxjs/toolkit': {
          singleton: true,
          requiredVersion: deps['@reduxjs/toolkit'],
        },
      },
    }),
    ...(isDev
      ? [new ReactRefreshPlugin({ overlay: false })]
      : [
          new MiniCssExtractPlugin({
            filename: 'static/css/[name].[contenthash].css',
            chunkFilename: 'static/css/[name].[contenthash].chunk.css',
          }),
        ]),
    new HTMLWebpackPlugin({
      template: './src/index.html',
      version,
      env: isProductionBuild ? 'production' : 'development',
    }),
  ],
}
```

In the module federation configuration, we define a global variable `delivery_test` to pass the parameters through dynamical importing. Be carefull, the variable's name does not accept `-` symbol.

For the Nx commands, we can also update the commands in `project.json`:

```js
"build": {
  "executor": "nx:run-commands",
  "options": {
    "command": "pnpm cross-env NODE_OPTIONS=--max-old-space-size=6144 NODE_ENV=production STAGING=true webpack --config ./apps/delivery-test/webpack.config.js"
  }
},
"build-serve": {
  "executor": "nx:run-commands",
  "options": {
    "command": "pnpm cross-env NODE_OPTIONS=--max-old-space-size=6144 NODE_ENV=production STAGING=true webpack serve --config ./apps/delivery-test/webpack.config.js --port 9002"
  }
},
"serve": {
  "executor": "nx:run-commands",
  "options": {
    "command": "pnpm cross-env NODE_OPTIONS=--max-old-space-size=6144 NODE_ENV=development webpack serve --config ./apps/delivery-test/webpack.config.js",
    "envFile": "apps/fleet-web/.env"
  }
},
```

### Module federation dynamic library

So the setup and configuration are completed. So how can we use the component from remote project in the host fleet-web? In the module federation configuration of host fleet-web, we do not have any text about the remote project `delivery-test`, it's because we gonna import the remote dynamically, which means the remote project can be imported during runtime without specifying at build time. For the principle, you can check [this one](https://h3manth.com/posts/dynamic-remotes-webpack-module-federation/) and also [4 ways to use dynamic remotes](https://oskari.io/blog/dynamic-remotes-module-federation/).


```js
function loadComponent(scope: string, module: string) {
  return async () => {
    const libName = scope.replace(/\//g, '_').replace(/-/g, '_')
    // Initializes the share scope. This fills it with known provided modules from this build and all remotes
    await __webpack_init_sharing__('default')
    const container = window[libName] // or get the container somewhere else
    console.log('load component', libName, container)
    // Initialize the container, it may provide shared modules
    await container.init(__webpack_share_scopes__.default)
    const factory = await window[libName].get(module)
    const Module = factory()
    return Module
  }
}

const urlCache = new Set()
const useDynamicScript = (url: string) => {
  const [ready, setReady] = React.useState(false)
  const [errorLoading, setErrorLoading] = React.useState(false)
  useEffect(() => {
    if (!url) return
    if (urlCache.has(url)) {
      setReady(true)
      setErrorLoading(false)
      return
    }
    setReady(false)
    setErrorLoading(false)
    const element = document.createElement('script')
    element.src = url
    element.type = 'text/javascript'
    element.async = true
    element.onload = () => {
      urlCache.add(url)
      setReady(true)
    }
    element.onerror = () => {
      setReady(false)
      setErrorLoading(true)
    }
    document.head.appendChild(element)
    return () => {
      urlCache.delete(url)
      document.head.removeChild(element)
    }
  }, [url])
  return {
    errorLoading,
    ready,
  }
}

const componentCache = new Map()
const useFederatedComponent = (remoteUrl: string, scope: string, module: string) => {
  const key = `${remoteUrl}-${scope}-${module}`
  const [Component, setComponent] = React.useState(null)
  const { ready, errorLoading } = useDynamicScript(remoteUrl)
  React.useEffect(() => {
    if (Component) setComponent(null)
    // Only recalculate when key changes
  }, [key])
  React.useEffect(() => {
    if (ready && !Component) {
      const Comp = React.lazy(loadComponent(scope, module))
      componentCache.set(key, Comp)
      setComponent(Comp)
    }
    // key includes all dependencies (scope/module)
  }, [Component, ready, key, module, scope])
  return { errorLoading, Component }
}
// NOTE: to make dynamica import work, we need to pass the container to a global variable which should be defined in the remote app's webpack config
// Find the file withModuleFederationPlugin.js in the remote app's webpack config
// 1. remove the code of setting "outputModule: true"
// 2. update the library in modulefederation from value 'module' to {
//   type: 'global',
//   name: options.name.replace('-', '_'), // the name does not accept dash and backslash
// }
const App = () => {
  const { Component: FederatedComponent, errorLoading } = useFederatedComponent(
    'http://localhost:9002/remoteEntry.js',
    'delivery-test',
    './Module',
  )
  return (
    <React.Suspense fallback="Loading System">
      {errorLoading
        ? `Error loading module "${module}"`
        : FederatedComponent && <FederatedComponent />}
    </React.Suspense>
  )
}
```

Use the above codes in the host project, and call the `App` directly with `<App />`.
