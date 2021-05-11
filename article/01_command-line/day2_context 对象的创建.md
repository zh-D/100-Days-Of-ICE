## context 对象的创建

```javascript
  start({
      args: { ...rawArgv },
      getBuiltInPlugins
  })

// build-script 的 start 函数
export = async function({
  args,
  rootDir,
  eject,
  plugins,
  getBuiltInPlugins,
}: {
  args: CommandArgs;
  rootDir: string;
  eject?: boolean;
  plugins?: IPluginList;
  getBuiltInPlugins?: IGetBuiltInPlugins;
}): Promise<void | ITaskConfig[] | WebpackDevServer> {
  const command = 'start';

  const context = new Context({
    args,
    command,
    rootDir,
    plugins,
    getBuiltInPlugins,
  });

  此处省略...
  
}
```

传递给 start 的参数 args 是 { mode: 'local', port: 3333 }

### Context 里面有什么。

所以这个 683 行的 Context 里面前面两百行都是 ts 的 interface，然后 Context 这个对象拥有 7 个 public 的属性，9 个 private 的属性。维持了 11 个 private 的方法，18 个 public 方法。

**constructor 的代码：**

```typescript
  constructor({
    command,
    rootDir = process.cwd(),
    args = {} ,
    plugins = [],
    getBuiltInPlugins = () => [],
  }: IContextOptions) {
      
	此处省略 20 行（给对象属性初始化值：this.XXX = xxx）...
    
    this.pkg = this.getProjectFile(PKG_FILE);
    this.userConfig = this.getUserConfig();
    // custom webpack
    const webpackPath = this.userConfig.customWebpack ? require.resolve('webpack', { paths: [this.rootDir] }) : 'webpack';
    this.webpack = require(webpackPath);
    if (this.userConfig.customWebpack) {
      hijackWebpackResolve(this.webpack, this.rootDir);
    }
    // register buildin options
    this.registerCliOption(BUILTIN_CLI_OPTIONS);
    const builtInPlugins: IPluginList = [...plugins, ...getBuiltInPlugins(this.userConfig)];
    this.checkPluginValue(builtInPlugins); // check plugins property
    this.plugins = this.resolvePlugins(builtInPlugins);
  }
```

**所以看一下 this.getProjectFile(PKG_FILE);**

```typescript
  private getProjectFile = (fileName: string): Json => {
    const configPath = path.resolve(this.rootDir, fileName);

    let config = {};
    if (fs.existsSync(configPath)) {
      try {
        config = fs.readJsonSync(configPath);
      } catch (err) {
        log.info('CONFIG', `Fail to load config file ${configPath}, use empty object`);
      }
    }

    return config;
  }
```

这里有：

- PKG_FILE <==> 'package.json'
- this.rootDir <==> process.cwd()

`process.cwd()` 方法会返回 Node.js 进程的当前工作目录，child（devServer） 进程的工作目录，所以就是项目根目录，所以 getProjectFile 的代码就是判断根目录下面有没有一个 package.json 文件。

**再看一下 getUserConfig()**

```typescript
  private getUserConfig = (): IUserConfig => {
    const { config } = this.commandArgs;
    let configPath = '';
    if (config) {
      configPath = path.isAbsolute(config) ? config : path.resolve(this.rootDir, config);
    } else {
      configPath = path.resolve(this.rootDir, USER_CONFIG_FILE);
    }
    let userConfig: IUserConfig = {
      plugins: [],
    };
    const isJsFile = path.extname(configPath) === '.js';
    if (fs.existsSync(configPath)) {
      try {
        userConfig = isJsFile ? require(configPath) : JSON5.parse(fs.readFileSync(configPath, 'utf-8')); // read build.json
      } catch (err) {
        log.info('CONFIG', `Fail to load config file ${configPath}, use default config instead`);
        log.error('CONFIG', (err.stack || err.toString()));
        process.exit(1);
      }
    }

    return this.mergeModeConfig(userConfig);
  }
```

我们先忽略掉这个 commandArgs 里面的 config，所以看一下 configPath = path.resolve(this.rootDir, USER_CONFIG_FILE);

这里有：

- USER_CONFIG_FILE <==> build.json

**所以 configPath 就是指 build.json 的 path**

再看一下 const isJsFile = path.extname(configPath) === '.js';

这个一定不是一个 jsFile 所以 isJsFile 应该是 false，如果是 true 我觉得这里应该是和 commandArgs 的 config 有关，暂时放一放。

最后 **userConfig** 为 “JSON5.parse(fs.readFileSync(configPath, 'utf-8'))” 的返回值

- fs.readFileSync(configPath, 'utf-8') 同步读取文件内容，把内容转化成 utf-8 再返回。
- JSON5 是 JSON 的一个超集，通过引入部分 ECMAScript 5.1 的特性来扩展 JSON 的语法，以减少 JSON 格式的某些限制。同时，保持兼容现有的 JSON 格式。在这里会返回**一个 JavaScript 对象**。

**this.mergeModeConfig(userConfig)**

```javascript
  private mergeModeConfig = (userConfig: IUserConfig): IUserConfig => {
    const { mode } = this.commandArgs;
    // modify userConfig by userConfig.modeConfig
    if (userConfig.modeConfig && mode && (userConfig.modeConfig as IModeConfig)[mode]) {
      const { plugins, ...basicConfig } = (userConfig.modeConfig as IModeConfig)[mode] as IUserConfig;
      const userPlugins = [...userConfig.plugins];
      if (Array.isArray(plugins)) {
        const pluginKeys = userPlugins.map((pluginInfo) => {
          return Array.isArray(pluginInfo) ? pluginInfo[0] : pluginInfo;
        });
        plugins.forEach((pluginInfo) => {
          const [pluginName] = Array.isArray(pluginInfo) ? pluginInfo : [pluginInfo];
          const pluginIndex = pluginKeys.indexOf(pluginName);
          if (pluginIndex > -1) {
            // overwrite plugin info by modeConfig
            userPlugins[pluginIndex] = pluginInfo;
          } else {
            // push new plugin added by modeConfig
            userPlugins.push(pluginInfo);
          }
        });
      }
      return { ...userConfig, ...basicConfig, plugins: userPlugins};
    }
    return userConfig;
  }
```

先看注释，注释里面说，通过 userConfig.modeConfig 修改 userConfig，由于 userConfig 是一个对象形式的 build.json，所以修改起来很方便。

这里对应有官方文档的[环境配置](https://ice.work/docs/guide/basic/config)

根据 icejs start --mode local，这个mode 应该是一个 local

对于 if (userConfig.modeConfig && mode && (userConfig.modeConfig as IModeConfig)[mode]) 这句话，userConfig.modeConfig 返回的是一个对象，mode 为键。就是说看一下 build.json 里面有没有一个 userConfig.modeConfig.local，没有就直接返回，有的话执行 if 的语句。

if 里，userPlugins 这个变量指的是在 build.json 直接配置的 plugins，而 plugins 这个变量指的是对应 mode 配置的 plugin，所以把对应 mode 的 plugins 全部给 merge 到 userPlugins 里面去。

最后 return { ...userConfig, ...basicConfig, plugins: userPlugins}，把 mode 里面的其他内容（basicConfig）原封不动的放到 build.json 的外层去。

**分析了 userConfig 的由来，现在我们再回到 constructor**

**custom webpack**

const webpackPath = this.userConfig.customWebpack ? require.resolve('webpack', { paths: [this.rootDir] }) : 'webpack';

以及 hijackWebpackResolve（好奇为什么是 hijack）

用来 icejs 在使用 webpack5 能力上的兼容处理

[官方文档](https://ice.work/docs/guide/develop/plugin-list)

*通过* `customWebpack` *配置的开启，工程中使用  webpack 的版本将会以项目中依赖的 webpack 版本为准*

**register buildin options**

```javascript
    this.registerCliOption(BUILTIN_CLI_OPTIONS);
    const builtInPlugins: IPluginList = [...plugins, ...getBuiltInPlugins(this.userConfig)];
    this.checkPluginValue(builtInPlugins); // check plugins property
    this.plugins = this.resolvePlugins(builtInPlugins);
```

**先看 registerCliOption**

```javascript
const BUILTIN_CLI_OPTIONS = [
  { name: 'port', commands: ['start'] },
  { name: 'host', commands: ['start'] },
  { name: 'disableAsk', commands: ['start'] },
  { name: 'config', commands: ['start', 'build', 'test'] },
];
```

根据名字就可以看出 registerCliOption 就是注册 cli 选项

```javascript
  public registerCliOption = (args: MaybeArray<ICliOptionArgs>): void => {
    this.registerConfig('cliOption', args, (name) => {
      return camelCase(name, { pascalCase: false });
    });
  }
```

看一下 camelCase，是来自 camelcase 这个包，它会将 name 转化为驼峰形式，如：

```
camelCase('foo-bar');
//=> 'fooBar'
camelCase('--foo.bar', {pascalCase: false});
//=> 'fooBar'
等
```

registerConfig

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

      this[registerKey][confName] = conf; // 就是把 BUILTIN_CLI_OPTIONS 里面的命令一个个注册

      // set default userConfig
      if (type === 'userConfig'
        && _.isUndefined(this.userConfig[confName])
        && Object.prototype.hasOwnProperty.call(conf, 'defaultValue')) {
        this.userConfig[confName] = (conf as IUserConfigArgs).defaultValue;
      }
    });
  }
```

**再看 builtInPlugins 相关**

const builtInPlugins: IPluginList = [...plugins, ...getBuiltInPlugins(this.userConfig)];

resolvePlugins(builtInPlugins)

```javascript
  private resolvePlugins = (builtInPlugins: IPluginList): IPluginInfo[] => {
    const userPlugins = [...builtInPlugins, ...(this.userConfig.plugins || [])].map((pluginInfo): IPluginInfo => {
      let fn;
      if (_.isFunction(pluginInfo)) {
        return {
          fn: pluginInfo,
          options: {},
        };
      }
      const plugins: [string, IPluginOptions] = Array.isArray(pluginInfo) ? pluginInfo : [pluginInfo, undefined];
      const pluginResolveDir = process.env.EXTRA_PLUGIN_DIR ? [process.env.EXTRA_PLUGIN_DIR, this.rootDir] : [this.rootDir];
      const pluginPath = path.isAbsolute(plugins[0]) ? plugins[0] : require.resolve(plugins[0], { paths: pluginResolveDir });
      const options = plugins[1];

      try {
        fn = require(pluginPath) // eslint-disable-line
      } catch (err) {
        log.error('CONFIG', `Fail to load plugin ${pluginPath}`);
        log.error('CONFIG', (err.stack || err.toString()));
        process.exit(1);
      }

      return {
        name: plugins[0],
        pluginPath,
        fn: fn.default || fn || ((): void => {}),
        options,
      };
    });

    return userPlugins;
  }
```

对于 const userPlugins = [...builtInPlugins, ...(this.userConfig.plugins || [])].map((pluginInfo) => {xxx}

这段代码就是把 build.json 的 Mode 对应的 plugins、build.json 外层的配置、getBuiltInPlugins、的 plugins 的名字都集合到一起（根据 mergeModeConfig，this.userConfig.plugins），然后根据这个名字找到 plugins 的物理位置。

可以看到 pluginResolveDir = [process.env.EXTRA_PLUGIN_DIR, this.rootDir] : [this.rootDir]

**突然出现一个 EXTRA_PLUGIN_DIR，怎么来的呢？留个坑**

所以 plugin 的目录是 EXTRA_PLUGIN_DIR 或者是项目根目录。这里还是会有很多细节的。