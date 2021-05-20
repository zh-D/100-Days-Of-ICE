前面和官方文档差不多，从 index/src 才开始看代码。

plugin-router 支持约定式路由和配置式路由。判定规则是如果 src/routes.ts 或者 build.json 里面配置的 configFile 对应文件存在，则使用配置式路由，否则使用约定式路由。

## 约定式路由

假设我们使用约定式路由，这意味着不存在 src/routes.ts，且 build.json 没有配置 configFile。我们看一下约定式路由可以使用的配置。它分为构建时和运行时。

**构建时：**

```json
{
  "router": {
    "ignorePaths": ["stores", "components"]
  }
}
```

- caseSensitive：boolean，默认是 false，插件会根据项目目录自己生成路由。
- ignoreRoutes：string[]，默认值是 []，路由配置不会被生成。
- ignorePaths：string[]，默认值 ['components']，忽略在 components 目录的每一个路径。

**运行时：**

```javascript
import { runApp } from 'ice'

const appConfig = {
  router: {
    type: 'browser',
    basename: '/seller',
    modifyRoutes: (routes) => {
      return routes;
    }
  }
};

runApp(appConfig);
```

如上所示，它有三个字段：type: 'browser';basename: '/seller';modifyroutes: ()=>{};

**生成规则：**

它可以为你生成基础路由、嵌套路由和动态路由，目录结构和它们的命名决定生成什么样的路由：

- 基础路由：

```javascript
src/pages
└── About
    └── index.tsx
└── Dashboard
    ├── a.tsx
    └── b.tsx
```

生成路由配置如下：

```js
[
  {
    path: '/dashboard',
    exact: true,
    component: PageDashboard
  },
  {
    path: '/home/a',
    exact: true,
    component: PageHomeA
  },
  {
    path: '/home/b',
    exact: true,
    component: PageHomeB
  }
]
```

- 嵌套路由：

```text
src/pages
└── About
    ├── _layout.tsx
    ├── a.tsx
    └── b.tsx
└── Dashboard
    └── index.tsx
```

生成路由配置如下：

```js
[
  {
    path: '/about',
    exact: false,
    component: LayoutAbout,
    children: [
      {
        path: '/a',
        exact: true,
        component: PageAboutA
      },
      {
        path: '/b',
        exact: true,
        component: PageAboutB
      },
    ],
  },
  {
    path: '/dashboard',
    exact: true,
    component: PageDashboard
  }
]
```

- 动态路由分为可选和不可选两种：

- 路径中`$`作为文件夹或文件名的首个字符，标识一个动态路由，如 src/pages/app/$uid.tsx 会生成路由  /app/:uid
- 路径中文件夹或文件名的首个和最后一个字符同时为`$`，标识一个可选的动态路由，如 src/pages/app/$uid$.tsx 会生成路由  /app/:uid?

## 配置式路由

这表明存在 src/routes.ts，且 build.json 没有配置 configFile。

一样分为构建时配置和运行时配置。

**构建时：**只有一个配置：

- configPath: string, 默认是 src/routes.[ts|js]。我想没人去改这个吧？

**运行时：**

这个和约定式路由是一模一样的配置，它可以支持无限嵌套。

## plugin/index.ts

**先看一下这个做了什么？**

```javascript
  registerUserConfig({
    name: 'router',
    validation: (value) => {
      return validation('router', value, 'object|boolean');
    },
  });
```

我们找到这个函数 registerConfig：

```javascript
  private registerConfig = (type: string, args: MaybeArray<IUserConfigArgs> | MaybeArray<ICliOptionArgs>, parseName?: (name: string) => string): void => {
    const registerKey = `${type}Registration` as 'userConfigRegistration' | 'cliOptionRegistration';
    if (!this[registerKey]) {
      throw new Error(`unknown register type: ${type}, use available types (userConfig or cliOption) instead`);
    }
    const configArr = _.isArray(args) ? args : [args];
    configArr.forEach((conf): void => {
      const confName = parseName ? parseName(conf.name) : conf.name;
      if (this[registerKey][confName]) {
        throw new Error(`${conf.name} already registered in ${type}`);
      }

      this[registerKey][confName] = conf;

      // set default userConfig
      if (type === 'userConfig'
        && _.isUndefined(this.userConfig[confName])
        && Object.prototype.hasOwnProperty.call(conf, 'defaultValue')) {
        this.userConfig[confName] = (conf as IUserConfigArgs).defaultValue;
      }
    });
  }
```

看传参：第一个参数 type 为 “userConfig”；

第二个参数 args 为一个对象：

```javascript
  {
    name: 'router',
    validation: (value) => {
      return validation('router', value, 'object|boolean');
    },
  }
```

第三个参数为 undefined。

看一下函数体：

执行到 const configArr = _.isArray(args) ? args : [args]; 这一行，configArr 现在是：

```javascript
  [{
    name: 'router',
    validation: (value) => {
      return validation('router', value, 'object|boolean');
    },
  }]

```

对它进行 foeach 的时候就是，执行了这一条语句：

```javascript
this[registerKey][confName] = conf

即：this["userConfigRegistration"]["router"] =   {
    name: 'router',
    validation: (value) => {
      return validation('router', value, 'object|boolean');
    },
  }
```

**再看一下这个:**

```javascript
  const { routesPath, isConfigRoutes } = applyMethod('getRoutes', {
    rootDir,
    tempDir: iceTempPath,
    configPath,
    projectType,
    isMpa: isMpa || disableRouter,
    srcDir
  });
```

首先这个 getRoutes 是在 plugin-app-core 里面注册的：registerMethod('getRoutes', getRoutes)，所以我们才可以用它。我们看一下这个方法（plugin-app-core/src/utils/getRoutes）。

```javascript
function getRoutes({ rootDir, tempDir, configPath, projectType, isMpa, srcDir }: IParams): IResult {
  // if is mpa use empty router file
  if (isMpa) {
    const routesTempPath = path.join(tempDir, 'routes.ts');
    fse.writeFileSync(routesTempPath, 'export default [];', 'utf-8');
    configPath = routesTempPath;
    return {
      routesPath: configPath,
      isConfigRoutes: true
    };
  }

  const routesPath = configPath
    ? path.join(rootDir, configPath)
    : path.join(rootDir, srcDir, `/routes.${projectType}`);

  // 配置式路由
  const configPathExists = fse.existsSync(routesPath);
  if (configPathExists) {
    return {
      routesPath,
      isConfigRoutes: true
    };
  }

  // 约定式路由
  return {
    routesPath: path.join(tempDir, `routes.${projectType}`),
    isConfigRoutes: false
  };
}
```

所以配置式路由就使用的用户 src 下面的 routes.ts，约定式路由就使用的模板。

**再看一下 webpack 的配置：**

```javascript
  // modify webpack config
  onGetWebpackConfig((config) => {
    // add alias
    TEM_ROUTER_SETS.forEach(i => {
      config.resolve.alias.set(i, routesPath);
    });
    // alias for runtime/Router
    config.resolve.alias.set('$ice/Router', path.join(__dirname, 'runtime/Router'));

    // alias for runtime/history
    config.resolve.alias.set('$ice/history', path.join(iceTempPath, 'router/history'));

    // alias for runtime/ErrorBoundary
    config.resolve.alias.set('$ice/ErrorBoundary', path.join(iceTempPath, 'ErrorBoundary'));

    // alias for react-router-dom
    const routerName = 'react-router-dom';
    config.resolve.alias.set(routerName, require.resolve(routerName));

    // config historyApiFallback for router type browser
    config.devServer.set('historyApiFallback', true);
  });
```

这几行代码对应的代码太长了**留个坑**

**看一下这个：**

```javascript
if (!disableRouter) {
    xxxx
}
```

实际上文档里面对这个 disableRouter 一点也没讲。**也留个坑。**

## plugin/runtime

**看一下这个 modifyRoutes**

```javascript
  modifyRoutes(() => {
    return renderComponent ? [{ component: renderComponent }] : formatRoutes(appConfigRouter.routes || defaultRoutes, '');
  });
```

这个函数在 create-app-shared

```javascript
  public modifyRoutes = (modifyFn) => {
    this.modifyRoutesRegistration.push(modifyFn);
  }

```

官方文档上对 renderComponent 有一句解释：

可选，用于渲染一个简单组件，不再需要耦合 react-router 的路由系统，需要配合设置 build.json 的 router 项为 false。不太懂。留个坑。

**formatRoutes(appConfigRouter.routes || defaultRoutes, '')**

```javascript
export default function formatRoutes(routes: IRouterConfig[], parentPath: string) {
  return routes.map((item) => {
    if (item.path) {
      const joinPath = path.join(parentPath || '', item.path);
      item.path = joinPath === '/' ? '/' : joinPath.replace(/\/$/, '');
    }
    if (item.children) {
      item.children = formatRoutes(item.children, item.path);
    } else if (item.component) {
      const itemComponent = item.component as any;
      itemComponent.pageConfig = Object.assign({}, itemComponent.pageConfig, { componentName: itemComponent.name });
    }
    return item;
  });
}
```

**看一下 modifyRoutesComponent**

```javascript
  public modifyRoutesComponent = (modify: (routesComponent: IRoutesComponent) => IRoutesComponent) => {
    this.routesComponent = modify(this.routesComponent);
  }
```

**wrapperRoutesComponent**

```javascript
  public wrapperRouteComponent = (wrapperRoute) => {
    this.wrapperRouteRegistration.push(wrapperRoute);
  }
```

看一下 

```javascript
function createHistory({ routes, customHistory, type, basename, location }: any) {
  if (process.env.__IS_SERVER__) {
    history = createMemoryHistory();
    history.location = location;
  } else if (customHistory) {
    history = customHistory;
  } else {
    // Force memory history when env is weex or kraken
    if (isWeex || isKraken) {
      type = 'memory';
    }
    if (type === 'hash') {
      history = createHashHistory({ basename });
    } else if (type === 'browser') {
      history = createBrowserHistory({ basename });
    } else if (isMiniAppPlatform) {
      (window as any).history = createMiniAppHistory(routes);
      window.location = (window.history as any).location;
      history = window.history;
    } else {
      history = createMemoryHistory();
    }
  }
  return history;
}

```

感觉这个 plugin-router 写得很好，完全感觉不到在使用 react-router，但感觉现在没必要理解这么多，先放一放。