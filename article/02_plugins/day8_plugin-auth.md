昨天看到 createBaseApp:

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

可以看到它返回了一个函数，还是叫做 createBaseApp。这个 createBaseApp 会作为回调传递给 reactAppRenderer（来自于 react-app-renderer 这个包）

renderer 接收的参数如下：

```diff
  renderer(
    {
      appConfig,
      staticConfig: staticConfig || defaultStaticConfig,
      buildConfig,
      setAppConfig,
+      createBaseApp,
      createHistory,
      getHistory,
      emitLifeCycles,
      pathRedirect,
      loadStaticModules,
      ErrorBoundary,
    },
    {
      createElement,
      mount,
      unmount,
      Component,
    }
  );
```

这里传了两个对象，但其实 reactAppRenderer 只接收一个对象：

```javascript
export async function reactAppRenderer(options) {
  const { appConfig, setAppConfig, loadStaticModules } = options || {};

  setAppConfig(appConfig);

  loadStaticModules(appConfig);

  if (process.env.__IS_SERVER__) return;

  renderInBrowser(options);
}
```

传递两个对象和那么多参数可能是因为 rax 场景。

可以看一下 renderInBrowser，他有一句这样的代码：

```javascript
context.initialData = await appConfig.app.getInitialData(initialContext);
```

可以知道  context.initialData 就是 appConfig.app.getInitialData。

## plugin-auth

这个 plugin 解决的问题是权限管理相关。

### src/index

```javascript
import * as path from 'path';
import * as fse from 'fs-extra';

const PLUFIN_AUTH_DIR = 'auth';

export default async function (api) {
  const { getValue, onGetWebpackConfig, applyMethod } = api;
  const iceTemp = getValue('TEMP_PATH');

  // 复制模板到 .ice/auth 目录下
  const templateSourceDir = path.join(__dirname, '../template');
  const TemplateTargetDir = path.join(iceTemp, PLUFIN_AUTH_DIR);
  fse.ensureDirSync(TemplateTargetDir);
  fse.copySync(templateSourceDir, TemplateTargetDir);

  onGetWebpackConfig((config) => {
    // 设置 $ice/authStore -> .ice/auth/store.ts
    config.resolve.alias.set('$ice/authStore', path.join(iceTemp, PLUFIN_AUTH_DIR, 'store.ts'));
  });

  // 导出接口
  // import { useAuth, withAuth } from 'ice';
  applyMethod('addExport', { source: './auth', importSource: '$$ice/auth', exportMembers: ['withAuth', 'useAuth'] });

  // 设置类型
  // export interface IAppConfig {
  //   auth?: IAuth;
  // }
  applyMethod('addAppConfigTypes', { source: './auth/types', specifier: '{ IAuth }', exportName: 'auth?: IAuth' });

}
```

做了什么注释说得很清楚。

看一下 .ice/auth，它的 index 导出了两个方法

```javascript
import React from 'react';
import store from './store';

// 函数组件支持 hooks 用法
function useAuth() {
  const [auth, { setAuth }] = store.useModel('auth');
  return [
    auth,
    setAuth
  ] as [Record<string, boolean>, typeof setAuth];
}

// class 组件支持 Hoc 用法
function withAuth(Component) {
  const AuthWrappered = props => {
    const [auth, setAuth] = useAuth();
    return <Component {...Object.assign({}, props, { auth, setAuth })} />;
  };
  return AuthWrappered;
}

export {
  useAuth,
  withAuth
};

```

下面结合官方文档看一下怎么回事。

### 初始化权限数据

```javascript
import { runApp, request, IAppConfig } from 'ice';

const appConfig: IAppConfig = {
  app: {
    getInitialData: async () => {
      // 模拟服务端返回的数据
      const data = await request('/api/auth');
      const { role, starPermission, followPermission } = data;

      // 约定权限必须返回一个 auth 对象
      // 返回的每个值对应一条权限
      return {
        auth: {
          admin: role === 'admin',
          guest: role === 'guest',
          starRepo: starPermission,
          followRepo: followPermission
        }
      }
    },
  },
  auth: {
    // 可选的，设置无权限时的展示组件，默认为 null
    NoAuthFallback: <div>没有权限...</div>,
    // 或者传递一个函数组件
    // NoAuthFallback: () => <div>没有权限..</div>
  }
};

runApp(appConfig);
```

由 appConfig.app.getInitialData 可知 context.initialData 为：

```javascript
        auth: {
          admin: role === 'admin',
          guest: role === 'guest',
          starRepo: starPermission,
          followRepo: followPermission
        }
```

**提出问题，如何拿到 initialData ？**

可以看到 runtime.tsx:

```javascript
export default ({ context, appConfig, addProvider, wrapperRouteComponent }) => {
  const initialData = context && context.initialData ? context.initialData : {};
  const initialAuth = initialData.auth || {} ;
  const authConfig = appConfig.auth || {};

  const AuthStoreProvider = ({children}) => {
    return (
      <AuthStore.Provider initialStates={{ auth: initialAuth }}>
        {children}
      </AuthStore.Provider>
    );
  };

  addProvider(AuthStoreProvider);
  wrapperRouteComponent(wrapperComponentFn(authConfig));
};
```

可以看到 runtime 和官方文档的对应关系。initialAuth 放到 app，authConfig 放到 page。

看一下 **页面权限** 和 wrapperConpoonentFn 的关系：

页面权限:

```javascript
import React from 'react';

const Home = () => {
  return <>Home Page</>;
};

Home.pageConfig = {
  // 可选，配置准入权限，若不配置则代表所有角色都可以访问
  auth: ['admin'],
};
```

wrapperConpoonentFn:

```javascript
const wrapperComponentFn = (authConfig) => (PageComponent) => {
  const { pageConfig = {} } = PageComponent;

  const AuthWrapperedComponent = (props) => {
    const { auth, ...rest } = props;
    const [authState] = auth;
    const pageConfigAuth = pageConfig.auth;
    if(pageConfigAuth && !Array.isArray(pageConfigAuth)) {
      throw new Error('pageConfig.auth must be an array');
    }
    const hasAuth = Array.isArray(pageConfigAuth) && pageConfigAuth.length
      ? Object.keys(authState).filter(item => pageConfigAuth.includes(item) ? authState[item] : false).length
      : true;
    if (!hasAuth) {
      if (authConfig.NoAuthFallback) {
        if (typeof authConfig.NoAuthFallback === 'function') {
          return <authConfig.NoAuthFallback />;
        }
        return authConfig.NoAuthFallback;
      }
      return null;
    }
    return <PageComponent {...rest} />;
  };
  return AuthStore.withModel('auth')(AuthWrapperedComponent);
};
```

pageConfigAuth 会获取到当前页面的权限，如果没有权限，就会看 authConfig 里面有没有一个 ”NoAuthFallback“，有就返回这个组件，没有就返回 null。PS:只有 pageConfig.auth 里面配置的用户才有权限

### 操作权限

获取权限数据、设置权限数据

就是对应权限数据的增删改查操作

获取权限数据

```javascript
import React from 'react';
import { useAuth } from 'ice';

function Foo() {
  const [auth] = useAuth();
  return (
    <>
      当前用户权限数据：
      <code>{JSON.stringify(auth)}</code>
    </>
  );
}
```

设置权限数据

```javascript
import React from 'react';
import { useAuth } from 'ice';

function Foo() {
  const [auth, setAuth] = useAuth();

  // 更新权限
  function updateAuth() {
    setAuth({ starRepo: false, followRepo: false });
  }

  return (
    <>
      当前用户角色：
      <code>{JSON.stringify(auth)}</code>
      <button type="button" onClick={updateAuth}>更新权限</button>
    </>
  );
}
```

**自定义权限组件**

对于操作类权限，通常我们可以自定义封装权限组件，以便更细粒度的控制权限和复用。

```javascript
import React from 'react';
import { useAuth } from 'ice';
import NoAuth from '@/components/NoAuth';

function Auth({ children, authKey, fallback }) {
  const [auth] = useAuth();
  // 判断是否有权限
  const hasAuth = auth[authKey];

  // 有权限时直接渲染内容
  if (hasAuth) {
    return children;
  } else {
    // 无权限时显示指定 UI
    return fallback || NoAuth
  }
};

export default Auth;
```

使用如下：

```javascript
function Foo () {
  return (
    <Auth authKey={'starRepo'}>
      <Button type="button">Star</Button>
    </Auth>
  )
}
```

### 接口鉴权

留个坑，要看 plugin-request