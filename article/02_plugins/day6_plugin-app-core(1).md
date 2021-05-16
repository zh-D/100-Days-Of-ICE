npm start 一直追溯到 build-scripts 的 start 命令里，发现有很多方法在 build-scripts 里面没有体现。应该是是在 setUp/runPlugins() 的时候注册这些 API。所以我应该看一下 plugins 们做了什么，所以先看一下 plugin-app-core。因为 plugin-app-core 是这样被描述的：The core plugin for icejs and rax app。

**看到 src/index:**

首先

```javascript
  const hasJsxRuntime = (() => {
    try {
      // auto detect of jsx runtime
      // eslint-disable-next-line
      const tsConfig = require(path.join(rootDir, 'tsconfig.json'));
      if (tsConfig?.compilerOptions?.jsx !== 'react-jsx') {
        return false;
      }
      // ensure react/jsx-runtime
      require.resolve('react/jsx-runtime');
      return true;
    } catch (e) {
      return false;
    }
  })();
```

注释上说 "自动检测 jsx runtime"，"确保 react/jsx-runtime"，其实我不懂 jsx-runtime，先跳过吧。

```javascript
setValue('HAS_JSX_RUNTIME', hasJsxRuntime)
```
看一下 setValue

```javascript
  public setValue = (key: string | number, value: any): void => {
    this.internalValue[key] = value;
  }
```

紧接着是下面的代码，不展开了。

```javascript
  // Check target
  checkTargets(targets); // 检测 目标产物

  // Set temporary directory
  // eg: .ice or .rax
  setTempDir(api, options);// 看生成 .rax 还是 .ice

  // Set project type
  // eg: ts | js
  setProjectType(api);// 看入口是 js 还是 ts

  // Modify default entry to src/app
  // eg: src/app.(ts|js)
  setEntry(api, options); // 如果框架是 react，而且 build.json 里面没有 entry，那么 设置 entry 为 'src/app'
```

看一下 setAlias

```javascript
export default (api, options: IOptions) => {
  const { onGetWebpackConfig, context, getValue } = api;
  const { alias, framework } = options;
  const { rootDir } = context;
  const tempPath = getValue(TEMP_PATH);

  const aliasMap = [
    [`${alias}$`, path.join(tempPath, 'index.ts')],
    [`${alias}`, path.join(tempPath, 'pages') ],
    ['@', path.join(rootDir, 'src')],
    // add alias for modular import
    [`$$${alias}`, tempPath],
    ['$$framework', tempPath],
  ];

  onGetWebpackConfig((config: any) => {
    // eslint-disable-next-line
    aliasMap.forEach(alias => {
      const hasAlias = config.resolve.alias.has(alias[0]);
      if(!hasAlias) {
        config.resolve.alias.set(alias[0], alias[1]);
      }
    });

    // add alias of basic dependencies
    let basicDependencies = [];
    if (framework === 'react') {
      basicDependencies = [
        ['react', rootDir],
        ['react-dom', rootDir]
      ];
    } else if (framework === 'rax') {
      basicDependencies = [
        ['rax', rootDir]
      ];
    }

    basicDependencies.forEach((dep: string[] | string): void => {
      const [depName, searchFolder] = Array.isArray(dep) ? dep : [dep];
      const aliasPath = searchFolder
        ? require.resolve(depName, { paths: [searchFolder] })
        : require.resolve(depName);
      config.resolve.alias.set(depName, path.dirname(aliasPath));
    });
  });

```

首先声明了一个 aliasMap，然后传递了一个 回调函数给 onGetWebpackConfig 这个 API。onGetWebpackConfig 这个 API 会把回调函数注册到 modifyConfigFn 这样一个数组里面，这个数组会在 runWebpackFunctions 的时候遍历一遍。

setRegisterUserConfig

```javascript
export default (api) => {
  const { registerUserConfig } = api;
  USER_CONFIG.forEach(item => registerUserConfig({ ...item }));
};

export const USER_CONFIG = [
  {
    name: 'store',
    validation: 'boolean'
  },
  {
    name: 'ssr',
    validation: 'boolean'
  },
  {
    name: 'auth',
    validation: 'boolean'
  },
  {
    name: 'sourceDir',
    validation: 'string',
  },
  {
    // add generateRuntime in case of runtimes do not pass the ts checker
    name: 'generateRuntime',
    validation: 'boolean',
    defaultValue: false,
  }
];
```

generator

```javascript
const generator = initGenerator(api, { ...options, debugRuntime: userConfig.generateRuntime, hasJsxRuntime });
```

initGenerator

```javascript
function initGenerator(api, options) {
  const { getAllPlugin, context, log, getValue } = api;
  const { userConfig, rootDir } = context;
  const { framework, debugRuntime, hasJsxRuntime } = options;
  const plugins = getAllPlugin();
  const { targets = [], ssr = false } = userConfig;
  const isMiniapp = targets.includes('miniapp') || targets.includes('wechat-miniprogram') || targets.includes('bytedance-microapp');
  const targetDir = getValue(TEMP_PATH);
  return new Generator({
    rootDir,
    targetDir,
    defaultData: {
      framework,
      isReact: framework === 'react',
      isRax: framework === 'rax',
      isMiniapp,
      ssr,
      buildConfig: JSON.stringify(getBuildConfig(userConfig)),
      hasJsxRuntime,
    },
    log,
    plugins,
    debugRuntime,
  });
}

// getAllPlugins 
  public getAllPlugin: IGetAllPlugin = (dataKeys = ['pluginPath', 'options', 'name']) => {
    return this.plugins.map((pluginInfo): Partial<IPluginInfo> => {
      // filter fn to avoid loop
      return _.pick(pluginInfo, dataKeys);
    });
  }

```

initGenerator 会返回一个 generator 对象

这个 generator 对象也有 200 多行代码，12 个 public API，1 个 private API

constructor 代码也不多：

```javascript
  constructor({ rootDir, targetDir, defaultData, log, plugins, debugRuntime }) {
    this.rootDir = rootDir;
    this.targetDir = targetDir;
    this.renderData = defaultData;
    this.contentRegistration = {};
    this.rerender = false;
    this.log = log;
    this.showPrettierError = true;
    this.renderTemplates = [];
    this.renderDataRegistration = [];
    this.plugins = plugins;
    this.debugRuntime = debugRuntime;
    this.disableRuntimePlugins = [];
  }

```

setRegisterMethod

setRegisterMethod 大概有 100 多行代码，它可以分为两段

1、它一直在重复调用一个 api：registerMethod。

```javascript
  // register utils method
  registerMethod('getPages', getPages);
  registerMethod('formatPath', formatPath);
  registerMethod('getRoutes', getRoutes);
  registerMethod('getSourceDir', getSourceDir);

  // registerMethod for modify page
  registerMethod('addPageExport', generator.addPageExport);
  registerMethod('removePageExport', generator.removePageExport);

  // registerMethod for render content
  registerMethod('addRenderFile', generator.addRenderFile);
  registerMethod('addTemplateDir', generator.addTemplateDir);
  registerMethod('modifyRenderData', generator.modifyRenderData);
  registerMethod('addDisableRuntimePlugin', generator.addDisableRuntimePlugin);

  function addImportDeclaration(data) {
    const { importSource, exportMembers, exportDefault, alias } = data;
    if (importSource) {
      if (exportMembers) {
        exportMembers.forEach((exportMember) => {
          // import { withAuth } from 'ice' -> import { withAuth } from 'ice/auth';
          importDeclarations[exportMember] = {
            value: importSource,
            type: 'normal',
          };
        });
      } else if (exportDefault) {
        // import { Helmet } from 'ice' -> import Helmet from 'ice/helmet';
        importDeclarations[exportDefault] = {
          value: importSource,
          type: 'default',
        };
      }
      if (alias) {
        Object.keys(alias).forEach(exportMember => {
          // import { Head } from 'ice'; -> import { Helmet as Head } from 'react-helmet';
          importDeclarations[exportMember] = {
            value: importSource,
            type: 'normal',
            alias: alias[exportMember],
          };
        });
      }
    }
  }

  registerMethod('addImportDeclaration', addImportDeclaration);
```

registerMethod 之后，用户就可以调用 applyMethod

2、

```javascript
api.setValue('importDeclarations', importDeclarations);
  // registerMethod for add export
  const apiKeys = getExportApiKeys();
  apiKeys.forEach((apiKey) => {
    registerMethod(apiKey, (exportData) => {
      addImportDeclaration(exportData);
      generator.addExport(apiKey, exportData);
    });
    registerMethod(apiKey.replace('add', 'remove'), (removeExportName) => {
      generator.removeExport(apiKey, removeExportName);
    });
  });

  const registerAPIs = {
    addEntryImports: {
      apiKey: 'addContent',
    },
    addEntryCode: {
      apiKey: 'addContent',
    },
  };

  Object.keys(registerAPIs).forEach((apiName) => {
    registerMethod(apiName, (code, position = 'after') => {
      const { apiKey } = registerAPIs[apiName];
      generator[apiKey](apiName, code, position);
    });
  });
```

先看一下这个 api.setValue('importDeclarations', importDeclarations)

看一下 importDeclarations:

```javascript
const importDeclarations: any = {};
const defaultDeclarations = {
  // default export in app
  '$$framework/runApp': [
    'runApp', 'createApp',
    // router api
    'withRouter', 'history', 'getHistory', 'getSearchParams', 'useSearchParams', 'withSearchParams', 'getInitialData',
    // LifeCycles api
    'usePageShow', 'usePageHide', 'withPageLifeCycle',
    // events api
    'registerNativeEventListeners',
    'addNativeEventListener',
    'removeNativeEventListener',
    'ErrorBoundary',
  ],
  // export lazy
  '$$ice/lazy': ['lazy'],
  // export types
  '$$framework/types': ['IApp', 'IAppConfig'],
};

Object.keys(defaultDeclarations).forEach((importSource) => {
  defaultDeclarations[importSource].forEach((apiKey) => {
    importDeclarations[apiKey] = { value: importSource };
  });
});

export default importDeclarations;
```

我把里面的代码跑了一下：

```javascript
{
  runApp: { value: '$$framework/runApp' },
  createApp: { value: '$$framework/runApp' },       
  withRouter: { value: '$$framework/runApp' },      
  history: { value: '$$framework/runApp' },
  getHistory: { value: '$$framework/runApp' },      
  getSearchParams: { value: '$$framework/runApp' }, 
  useSearchParams: { value: '$$framework/runApp' }, 
  withSearchParams: { value: '$$framework/runApp' },
  getInitialData: { value: '$$framework/runApp' },  
  usePageShow: { value: '$$framework/runApp' },     
  usePageHide: { value: '$$framework/runApp' },
  withPageLifeCycle: { value: '$$framework/runApp' },
  registerNativeEventListeners: { value: '$$framework/runApp' },
  addNativeEventListener: { value: '$$framework/runApp' },
  removeNativeEventListener: { value: '$$framework/runApp' },
  ErrorBoundary: { value: '$$framework/runApp' },
  lazy: { value: '$$ice/lazy' },
  IApp: { value: '$$framework/types' },
  IAppConfig: { value: '$$framework/types' }
}
```

再看一下 const apiKeys = getExportApiKeys()：

```javascript
export const EXPORT_API_MPA = [
  {
    name: ['addIceExport', 'addExport'],
    value: ['imports', 'exports']
  },
  {
    name: ['addIceTypesExport', 'addTypesExport'],
    value: ['typesImports', 'typesExports'],
  },
  {
    name: ['addIceAppConfigTypes', 'addAppConfigTypes'],
    value: ['appConfigTypesImports', 'appConfigTypesExports']
  },
  {
    name: ['addIceAppConfigAppTypes', 'addAppConfigAppTypes'],
    value: ['appConfigAppTypesImports', 'appConfigAppTypesExports']
  }
];
export const getExportApiKeys = () => {
  let apiKeys = [];
  EXPORT_API_MPA.forEach(item => {
    apiKeys = apiKeys.concat(item.name);
  });
  return apiKeys;
};

```
我把里面的代码跑了一下:
```javas
[
  'addIceExport',     
  'addExport',        
  'addIceTypesExport',
  'addTypesExport',   
  'addIceAppConfigTypes',
  'addAppConfigTypes',
  'addIceAppConfigAppTypes',
  'addAppConfigAppTypes'
]
```

看一下 apiKeys.forEach，就是为这些键 registerMethod，包括 removeMethod

这里要看一下 generator.addExport(apiKey, exportData);和generator.removeExport(apiKey, removeExportName);

```javascript
  public addExport = (registerKey, exportData: IExportData | IExportData[]) => {
    const exportList = this.contentRegistration[registerKey] || [];
    checkExportData(exportList, exportData, registerKey);
    this.addContent(registerKey, exportData);
    if (this.rerender) {
      this.render();
    }
  }
```

```javascript
  public removeExport = (registerKey: string, removeExportName: string | string[]) => {
    const exportList = this.contentRegistration[registerKey] || [];
    this.contentRegistration[registerKey] = removeExportData(exportList, removeExportName);
    if (this.rerender) {
      this.render();
    }
  }
```

还有 Object.keys(registerAPIs).forEach，要看一下 generator.addContent：

```javascript
  public addContent(apiName, ...args) {
    const apiKeys = getExportApiKeys();
    if (!apiKeys.includes(apiName)) {
      throw new Error(`invalid API ${apiName}`);
    }
    const [data, position] = args;
    if (position && !['before', 'after'].includes(position)) {
      throw new Error(`invalid position ${position}, use before|after`);
    }
    const registerKey = position ? `${apiName}_${position}` : apiName;
    if (!this.contentRegistration[registerKey]) {
      this.contentRegistration[registerKey] = [];
    }
    const content = Array.isArray(data) ? data : [data];
    this.contentRegistration[registerKey].push(...content);
  }
```

回到 src/index，看一下剩余的代码：

```javascript
  // add core template for framework
  const templateRoot = path.join(__dirname, './generator/templates');
  [`./app/${framework}`, './common'].forEach((templateDir) => {
    generator.addTemplateDir(path.join(templateRoot, templateDir));
  });

  // watch src folder
  if (command === 'start') {
    dev(api, { render: generator.render });
  }

  onHook(`before.${command}.run`, async () => {
    await generator.render();
  });
```

所以看一下 generator.addTemplateDir：

```javascript
  public addTemplateDir = (templateDir: string, extraData: IRenderData = {}) => {
    const templates = globby.sync(['**/*'], { cwd: templateDir });
    templates.forEach((templateFile) => {
      this.addRenderFile(path.join(templateDir, templateFile), path.join(this.targetDir, templateFile), extraData);
    });
    if (this.rerender) {
      this.debounceRender();
    }
  }
```

所以看一下 addRenderFile

```javascript
  public addRenderFile = (templatePath: string, targetPath: string, extraData: IRenderData = {}) => {
    // check target path if it is already been registed
    const renderIndex = this.renderTemplates.findIndex(([, templateTarget]) => templateTarget === targetPath);
    if (renderIndex > -1) {
      const targetTemplate = this.renderTemplates[renderIndex];
      if (targetTemplate[0] !== templatePath) {
        this.log.error('[template]', `path ${targetPath} already been rendered as file ${targetTemplate[0]}`);
      }
      // replace template with lastest content
      this.renderTemplates[renderIndex] = [templatePath, targetPath, extraData];
    } else {
      this.renderTemplates.push([templatePath, targetPath, extraData]);
    }
    if (this.rerender) {
      this.debounceRender();
    }
  }
```

看一下 render：

```javascript
  public render = () => {
    this.rerender = true;
    const plugins = this.plugins.filter((plugin) => {
      return !this.disableRuntimePlugins.includes(plugin.name);
    });

    this.renderData = this.renderDataRegistration.reduce((previousValue, currentValue) => {
      if (typeof currentValue === 'function') {
        return currentValue(previousValue);
      }
      return previousValue;
    }, this.parseRenderData());

    this.renderData.runtimeModules = getRuntimeModules(plugins, this.targetDir, this.debugRuntime);

    this.renderTemplates.forEach((args) => {
      this.renderFile(...args);
    });
  };
```

所以看一下 renderFile

```javascript
  public renderFile: IRenderFile = (templatePath, targetPath, extraData = {}) => {
    const renderExt = '.ejs';
    if (path.extname(templatePath) === '.ejs') {
      const templateContent = fse.readFileSync(templatePath, 'utf-8');
      let content = ejs.render(templateContent, { ...this.renderData, ...extraData });
      try {
        content = prettier.format(content, {
          parser: 'typescript',
          singleQuote: true
        });
      } catch (error) {
        if (this.showPrettierError) {
          this.log.warn(`Prettier format error: ${error.message}`);
          this.showPrettierError = false;
        }
      }
      const realTargetPath = targetPath.replace(renderExt, '');
      fse.ensureDirSync(path.dirname(realTargetPath));
      fse.writeFileSync(realTargetPath, content, 'utf-8');
    } else {
      fse.ensureDirSync(targetPath);
      fse.copyFileSync(targetPath, targetPath);
    }
  }
```

其实就是生成 .ice 下面的文件