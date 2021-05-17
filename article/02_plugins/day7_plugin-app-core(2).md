本节主要讨论 runtime 是如何运行的？

src/generator 的结构如下：

```
generator
├── template
│   ├── app
│       ├── rax
│           ├── index.tx.ejs
│           └── ErrorBoundary
│               ├── ErrorBoundaryFallback.tsx.ejs
│               └── index.tsx.ejs
│       ├── react
│           ├── genWithSearchParams.tx.ejs
│           ├── index.tx.ejs
│           ├── lazy.tx.ejs
│           └── ErrorBoundary
│               ├── ErrorBoundaryFallback.tsx.ejs
│               └── index.tsx.ejs
│   └── common
│       ├── appConfig.ts.ejs
│       ├── loadRuntimeModules.ts.ejs
│       ├── loadStaticModules.ts.ejs
│       ├── render.ts.ejs
│       ├── runApp.ts.ejs
│       ├── staticConfig.ts.ejs
│       └── types.ts.ejs      
└── index.ts		  
```

根据这些模板生成的 .ice 

```
.ice
│  └── ErrorBoundary
│      ├── ErrorBoundaryFallback.tsx
│      └── index.tsx
│  ├── genWithSearchParams.tx
│  ├── index.tx
│  ├── lazy.tx
│  ├── appConfig.ts
│  ├── loadRuntimeModules.ts
│  ├── loadStaticModules.ts
│  ├── render.ts
│  ├── runApp.ts
│  ├── staticConfig.ts
│  └── types.ts
└── index.ts	
```

实际上生成的多了这么一些东西：

```diff
.ice
+│  └── auth
+│      ├── index.tsx
+│      ├── model.ts
+│      ├── store.ts
+│      └── types.ts
│  └── ErrorBoundary
│      ├── ErrorBoundaryFallback.tsx
│      └── index.tsx
+│  └── helpers
+│      ├── cookie.tsx
+│      ├── index.ts
+│      └── urlParse.tsx
+│  └── logger
+│      ├── types
+│          ├── index.d.ts
+│          └── index.js
+│      └── index.ts
+│  └── logger
+│      ├── types
+│          ├── base.d.ts
+│          ├── base.js
+│          ├── index.d.ts
+│          └── index.js
+│      ├── createAxiosInstance.js
+│      ├── request.ts
+│      └── useRequest.ts
+│  └── router
+│      ├── types
+│          ├── base.ts
+│          └── index.ts
+│      ├── history.ts
+│      ├── index.ts
+│      └── react-router-dom.ts
│  ├── appConfig.ts
+│  ├── config.ts
│  ├── genWithSearchParams.ts
│  ├── index.tx
│  ├── lazy.tx
│  ├── loadRuntimeModules.ts
│  ├── loadStaticModules.ts
│  ├── render.ts
+│  ├── routes.js
│  ├── runApp.ts
│  ├── staticConfig.ts
│  └── types.ts
└── index.ts	
```

它们的模板分布在各个插件里面。

现在来看一下 plugin-app-core 的 runtime.tsx

```javascript
const module = ({ addProvider, appConfig, wrapperRouteComponent, getSearchParams, context: { createElement } }) => {
  const { app = {} } = appConfig;
  const { parseSearchParams = true } = app;
  const wrapperPageComponent = (PageComponent) => {
    const WrapperedPageComponent = (props) => {
      const searchParams = parseSearchParams && getSearchParams();
      return createElement(PageComponent, {...Object.assign({}, props, { searchParams })});
    };
    return WrapperedPageComponent;
  };

  wrapperRouteComponent(wrapperPageComponent);

  if (appConfig.app && appConfig.app.addProvider) {
    addProvider(appConfig.app.addProvider);
  }
};

export default module;

```

它是如何运行的呢？

在 .ice/runApp.ts 里面：

```javascript
import loadRuntimeModules from './loadRuntimeModules';
```

我们看一下 loadRuntimeModules

```javascript
interface IRuntime<T> {
  loadModule: (module: { default: T } | T) => void;
}

function loadRuntimeModules(runtime: IRuntime<Function>) {
  runtime.loadModule(
    require('C:/Users/86155/Desktop/ice/examples/simple/node_modules/build-plugin-app-core/lib/runtime.js')
  );

  runtime.loadModule(
    require('C:/Users/86155/Desktop/ice/examples/simple/node_modules/build-plugin-ice-router/lib/runtime.js')
  );

  runtime.loadModule(
    require('C:/Users/86155/Desktop/ice/examples/simple/node_modules/build-plugin-ice-logger/lib/runtime.js')
  );

  runtime.loadModule(
    require('C:/Users/86155/Desktop/ice/examples/simple/node_modules/build-plugin-ice-auth/lib/runtime.js')
  );
}

export default loadRuntimeModules;

```

然后在 loadRuntimeModules 作为参数 传递给 createShareAPI

```javascript
const {
  createBaseApp,
  withRouter,
  createHistory,
  getHistory,
  emitLifeCycles,
  usePageShow,
  usePageHide,
  withPageLifeCycle,
  pathRedirect,
  registerNativeEventListeners,
  addNativeEventListener,
  removeNativeEventListener,
  getSearchParams,
} = createShareAPI(
  {
    createElement,
    useEffect,
    withRouter: defaultWithRouter,
    initHistory: buildConfig.router !== false,
  },
  loadRuntimeModules
);
```

因为

```javascript
import createShareAPI, { history } from 'create-app-shared';
```

所以我们看一下 create-app-shared，我现在只关注这个 loadRuntimeModules，所以看一下这一行代码

```javascript
createBaseApp: createBaseApp({ loadRuntimeModules, createElement, initHistory }),
```

所以找到 createBaseApp

```javascript
export default ({ loadRuntimeModules, createElement, initHistory = true }) => {
  const createBaseApp = (appConfig, buildConfig, context: any = {}) => {

    // Merge default appConfig to user appConfig
    appConfig = deepmerge(DEFAULE_APP_CONFIG, appConfig);

    // Set history
    let history: any = {};
    if (!isMiniAppPlatform && initHistory) {
      const { router } = appConfig;
      const { type, basename, history: customHistory } = router;
      const location = context.initialContext ? context.initialContext.location : null;
      history = createHistory({ type, basename, location, customHistory });
      appConfig.router.history = history;
    }

    context.createElement = createElement;

    // Load runtime modules
    const runtime = new RuntimeModule(appConfig, buildConfig, context);
    loadRuntimeModules(runtime);

    // Collect app lifeCyle
    collectAppLifeCycle(appConfig);
    return {
      history,
      runtime,
      appConfig
    };
  };

  return createBaseApp;
};
```

所以我们来看一下 RuntimeModule 这个对象，它一共 117 行代码，定义了 10 个public 方法，和 8 个（或 9 个） 有用的 private 属性。

runtime.loadModule 暴露的属性或方法如下：

```javascript
  public loadModule(module) {
    const runtimeAPI = {
      setRenderRouter: this.setRenderRouter,
      addProvider: this.addProvider,
      addDOMRender: this.addDOMRender,
      modifyRoutes: this.modifyRoutes,
      wrapperRouteComponent: this.wrapperRouteComponent,
      modifyRoutesComponent: this.modifyRoutesComponent,
      createHistory,
      getSearchParams
    };

    if (module) (module.default || module)({
      ...runtimeAPI,
      appConfig: this.appConfig,
      buildConfig: this.buildConfig,
      context: this.context
    });
  }
```

plugin-app-core 拿到了 如下属性或方法

```javascript
{ addProvider, appConfig, wrapperRouteComponent, getSearchParams, context: { createElement } }
```

我们来一个个看一下代码。

**addProvider**

```javas
  public addProvider = (Provider) => {
    this.AppProvider.push(Provider);
  }
```

放进去然后呢？到时候会遍历这些数组，一层层在 app 上包 Provider。

**appConfig**

appConfig就是

```javascript
const DEFAULE_APP_CONFIG = {
  app: {
    rootId: 'root'
  },
  router: {
    type: 'hash'
  }
};
appConfig = deepmerge(DEFAULE_APP_CONFIG, appConfig);
// appConfig 就是 app.ts 里面配置的那个 appConfig。
```

**wrapperRouteComponent**

```javascript
  public wrapperRouteComponent = (wrapperRoute) => {
    this.wrapperRouteRegistration.push(wrapperRoute);
  }
```

这里传进来的 wrapperRoute 是个这样的回调函数：

```javascript
  const wrapperPageComponent = (PageComponent) => {
    const WrapperedPageComponent = (props) => {
      const searchParams = parseSearchParams && getSearchParams();
      return createElement(PageComponent, {...Object.assign({}, props, { searchParams })});
    };
    return WrapperedPageComponent;
  };
```

官网上说 wrapperRouteComponent 是为所有页面级组件做一层包裹：这里的话添加了属性 searchParams。

**getSearchParams**

```javascript
import * as queryString from 'query-string';
import { getHistory } from './history';

export default function() {
  const history = getHistory();
  if (history && history.location && history.location.search) {
    return queryString.parse(history.location.search);
  }
  return {};
}
```

```javascript
function getHistory() {
  return isMiniAppPlatform ? window.history : history;
}
```

最后 createElement 直接来自 react，作用是创建 react 元素。

```javascript
createElement(PageComponent, {...Object.assign({}, props, { searchParams })});
```