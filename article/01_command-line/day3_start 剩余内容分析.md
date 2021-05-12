### start 里剩下的内容

start 还剩下 100 多行代码

const { applyHook, webpack } = context; 它从 context 里面拿了这两个值，所以我们看一下它们。

**applyHook**

```typescript
  public applyHook = async (key: string, opts = {}): Promise<void> => {
    const hooks = this.eventHooks[key] || [];

    for (const fn of hooks) {
      // eslint-disable-next-line no-await-in-loop
      await fn(opts);
    }
  }
  
  private eventHooks: {
    [name: string]: IOnHookCallback[];
  }
```

所以 applyHook 就是根据 key 执行一系列 Hooks

其实 eventHooks 初始值是 {}，那么不存在 Hooks，可以通过 onHooks 注册相应的 hooks。

webpack 可能是用户配置的 webpack5

**configArr = await context.setUp();**

```javascript
  public setUp = async (): Promise<ITaskConfig[]> => {
    await this.runPlugins();
    await this.runUserConfig();
    await this.runWebpackFunctions();
    await this.runCliOption();
    // filter webpack config by cancelTaskNames
    this.configArr = this.configArr.filter((config) => !this.cancelTaskNames.includes(config.name));
    return this.configArr;
  }
```

所以接下来看一下 **runPlugins、runUserConfig、runWebpackFunctions、runCliOption**

**runPlugins**

```javascript
  private runPlugins = async (): Promise<void> => {
    for (const pluginInfo of this.plugins) {
      const { fn, options } = pluginInfo;

      const pluginContext = _.pick(this, PLUGIN_CONTEXT_KEY);

      const pluginAPI = {
        log,
        context: pluginContext,
        registerTask: this.registerTask,
        getAllTask: this.getAllTask,
        getAllPlugin: this.getAllPlugin,
        cancelTask: this.cancelTask,
        onGetWebpackConfig: this.onGetWebpackConfig,
        onGetJestConfig: this.onGetJestConfig,
        onHook: this.onHook,
        setValue: this.setValue,
        getValue: this.getValue,
        registerUserConfig: this.registerUserConfig,
        registerCliOption: this.registerCliOption,
        registerMethod: this.registerMethod,
        applyMethod: this.applyMethod,
        hasMethod: this.hasMethod,
        modifyUserConfig: this.modifyUserConfig,
      };
      // eslint-disable-next-line no-await-in-loop
      await fn(pluginAPI, options);
    }
  }

```

这个 fn 其实就是 plugin 的 index.js。options 是用户在 build.json 配置，pluginAPI 给用户开发插件的时候用。

看一下这个 const pluginContext = _.pick(this, PLUGIN_CONTEXT_KEY);

```javascript
const PLUGIN_CONTEXT_KEY = [
  'command' as 'command',
  'commandArgs' as 'commandArgs',
  'rootDir' as 'rootDir',
  'userConfig' as 'userConfig',
  'pkg' as 'pkg',
  'webpack' as 'webpack',
];
```

结合 loadash.pick，我们可以知道插件的 context 里面只有以上六个值可以拿到。

**runUserConfig**

```javascript
  private runUserConfig = async (): Promise<void> => {
    for (const configInfoKey in this.userConfig) {
      if (!['plugins', 'customWebpack'].includes(configInfoKey)) {
        const configInfo = this.userConfigRegistration[configInfoKey];

        if (!configInfo) {
          throw new Error(`[Config File] Config key '${configInfoKey}' is not supported`);
        }

        const { name, validation } = configInfo;
        const configValue = this.userConfig[name];

        if (validation) {
          let validationInfo;
          if (_.isString(validation)) {
            const fnName = VALIDATION_MAP[validation as ValidationKey];
            if (!fnName) {
              throw new Error(`validation does not support ${validation}`);
            }
            assert(_[VALIDATION_MAP[validation as ValidationKey]](configValue), `Config ${name} should be ${validation}, but got ${configValue}`);
          } else {
            // eslint-disable-next-line no-await-in-loop
            validationInfo = await validation(configValue);
            assert(validationInfo, `${name} did not pass validation, result: ${validationInfo}`);
          }
        }

        if (configInfo.configWebpack) {
          // eslint-disable-next-line no-await-in-loop
          await this.runConfigWebpack(configInfo.configWebpack, configValue);
        }
      }
    }
  }
```

遍历每一项 userConfig，如果不是 plugins 或者 customWebpack 作为键，那么就要检查这个键的合法性。

**runWebpackFunctions**

```javascript
  private runWebpackFunctions = async (): Promise<void> => {
    this.modifyConfigFns.forEach(([name, func]) => {
      const isAll = _.isFunction(name);
      if (isAll) {  // modify all
        this.configArr.forEach(config => {
          config.modifyFunctions.push(name as IPluginConfigWebpack);
        });
      } else { // modify named config
        this.configArr.forEach(config => {
          if (config.name === name) {
            config.modifyFunctions.push(func);
          }
        });
      }
    });

    for (const configInfo of this.configArr) {
      for (const func of configInfo.modifyFunctions) {
        // eslint-disable-next-line no-await-in-loop
        await func(configInfo.chainConfig);
      }
    }
  }
```

遍历 modifyConfigFns，利用 modifyConfigFns 为 configArr 的 config 添加 modifyConfigFns。再遍历 configArr，调用这些 modifyConfigFns。

**runCliOption**

```javascript
  private runCliOption = async (): Promise<void> => {
    for (const cliOpt in this.commandArgs) {
      // allow all jest option when run command test
      if (this.command !== 'test' || cliOpt !== 'jestArgv') {
        const { commands, name, configWebpack } = this.cliOptionRegistration[cliOpt] || {};
        if (!name || !(commands || []).includes(this.command)) {
          throw new Error(`cli option '${cliOpt}' is not supported when run command '${this.command}'`);
        }
        if (configWebpack) {
          // eslint-disable-next-line no-await-in-loop
          await this.runConfigWebpack(configWebpack, this.commandArgs[cliOpt]);
        }
      }
    }
  }
```

执行命令行选项，主要是错误处理。

**runConfigWebpack**

```javascript
  private async runConfigWebpack(fn: IUserConfigWebpack, configValue: JsonValue): Promise<void> {
    for (const webpackConfigInfo of this.configArr) {
      const userConfigContext: UserConfigContext = {
        ..._.pick(this, PLUGIN_CONTEXT_KEY),
        taskName: webpackConfigInfo.name,
      };
      // eslint-disable-next-line no-await-in-loop
      await fn(webpackConfigInfo.chainConfig, configValue, userConfigContext);
    }
  }
```

所以，遍历插件，如果 this.cliOptionRegistration 有 configWebpack，则执行configWebpack。

**applyHook**

在 applyHook 之前需要执行 onHook 往 context 里面放事件。这个 onHook 的执行是在某个插件里面执行的。就是 runPlugins() 执行的时候。

如：

```javascript
await applyHook(`before.${command}.load`, { args, webpackConfig: configArr });
await applyHook(`error`, { err: new Error(errorMsg) });
```

它们的 onHook 是在哪个插件执行的呢。

**configArr**

filter webpack config by cancelTaskNames

```javascript
  for (const item of configArr) {
    const { chainConfig } = item;
    const config = chainConfig.toConfig();
    if (config.devServer) {
      devServerConfig = deepmerge(devServerConfig, config.devServer);
    }
    // if --port or process.env.PORT has been set, overwrite option port
    if (process.env.USE_CLI_PORT) {
      devServerConfig.port = args.port;
    }
  }
```

**compiler**

```javascript
  compiler = webpack(webpackConfig);
  let isFirstCompile = true;
  compiler.hooks.done.tap('compileHook', async (stats) => {
    const isSuccessful = webpackStats({
      urls,
      stats,
      isFirstCompile,
    });
    if (isSuccessful) {
      isFirstCompile = false;
    }
    await applyHook(`after.${command}.compile`, {
      url: serverUrl,
      urls,
      isFirstCompile,
      stats,
    });
  });
```

所以我们来看一下 **webpack**

```javascript
const webpack = /** @type {WebpackFunctionSingle & WebpackFunctionMulti} */ (
	/**
	 * @param {WebpackOptions | (WebpackOptions[] & MultiCompilerOptions)} options options
	 * @param {Callback<Stats> & Callback<MultiStats>=} callback callback
	 * @returns {Compiler | MultiCompiler}
	 */
	(options, callback) => {
		const create = () => {
			if (!webpackOptionsSchemaCheck(options)) {
				getValidateSchema()(webpackOptionsSchema, options);
			}
			/** @type {MultiCompiler|Compiler} */
			let compiler;
			let watch = false;
			/** @type {WatchOptions|WatchOptions[]} */
			let watchOptions;
			if (Array.isArray(options)) {
				/** @type {MultiCompiler} */
				compiler = createMultiCompiler(options, options);
				watch = options.some(options => options.watch);
				watchOptions = options.map(options => options.watchOptions || {});
			} else {
				/** @type {Compiler} */
				compiler = createCompiler(options);
				watch = options.watch;
				watchOptions = options.watchOptions || {};
			}
			return { compiler, watch, watchOptions };
		};
		if (callback) {
			try {
				const { compiler, watch, watchOptions } = create();
				if (watch) {
					compiler.watch(watchOptions, callback);
				} else {
					compiler.run((err, stats) => {
						compiler.close(err2 => {
							callback(err || err2, stats);
						});
					});
				}
				return compiler;
			} catch (err) {
				process.nextTick(() => callback(err));
				return null;
			}
		} else {
			const { compiler, watch } = create();
			if (watch) {
				util.deprecate(
					() => {},
					"A 'callback' argument need to be provided to the 'webpack(options, callback)' function when the 'watch' option is set. There is no way to handle the 'watch' option without a callback.",
					"DEP_WEBPACK_WATCH_WITHOUT_CALLBACK"
				)();
			}
			return compiler;
		}
	}
);
```

根据传参，webpack 的可执行代码是这个：

```javascript
			const { compiler, watch } = create();
			if (watch) {
				util.deprecate(
					() => {},
					"A 'callback' argument need to be provided to the 'webpack(options, callback)' function when the 'watch' option is set. There is no way to handle the 'watch' option without a callback.",
					"DEP_WEBPACK_WATCH_WITHOUT_CALLBACK"
				)();
			}
			return compiler;
```

所以我们看一下 create()

```javascript
		const create = () => {
			if (!webpackOptionsSchemaCheck(options)) {
				getValidateSchema()(webpackOptionsSchema, options);
			}
			/** @type {MultiCompiler|Compiler} */
			let compiler;
			let watch = false;
			/** @type {WatchOptions|WatchOptions[]} */
			let watchOptions;
			if (Array.isArray(options)) {
				/** @type {MultiCompiler} */
				compiler = createMultiCompiler(options, options);
				watch = options.some(options => options.watch);
				watchOptions = options.map(options => options.watchOptions || {});
			} else {
				/** @type {Compiler} */
				compiler = createCompiler(options);
				watch = options.watch;
				watchOptions = options.watchOptions || {};
			}
			return { compiler, watch, watchOptions };
		};
```

所以看一下 **createCompiler**

```javascript
const createCompiler = rawOptions => {
	const options = getNormalizedWebpackOptions(rawOptions);
	applyWebpackOptionsBaseDefaults(options);
	const compiler = new Compiler(options.context);
	compiler.options = options;
	new NodeEnvironmentPlugin({
		infrastructureLogging: options.infrastructureLogging
	}).apply(compiler);
	if (Array.isArray(options.plugins)) {
		for (const plugin of options.plugins) {
			if (typeof plugin === "function") {
				plugin.call(compiler, compiler);
			} else {
				plugin.apply(compiler);
			}
		}
	}
	applyWebpackOptionsDefaults(options);
	compiler.hooks.environment.call();
	compiler.hooks.afterEnvironment.call();
	new WebpackOptionsApply().process(options, compiler);
	compiler.hooks.initialize.call();
	return compiler;
};
```

这个 compliler 对象有 1000 多行代码，比较复杂，不过还好它只使用了 compiler.hooks.done.tap 这个函数，所以我们来找到它。

所以我们来看一下 hooks.done，它是 new AsyncSeriesHook(["stats"])，而 AsyncSeriesHook 来自 tapable 这个 npm 包。

**AsyncSeriesHook**

```javascript
function AsyncSeriesHook(args = [], name = undefined) {
	const hook = new Hook(args, name);
	hook.constructor = AsyncSeriesHook;
	hook.compile = COMPILE;
	hook._call = undefined;
	hook.call = undefined;
	return hook;
}
```

所以我们来看一下 **Hook**，大概 150 行代码，看一它的下 **tap()**

```javascript
	tap(options, fn) {
		this._tap("sync", options, fn);
	}

```

```javascript
	_tap(type, options, fn) {
		if (typeof options === "string") {
			options = {
				name: options.trim()
			};
		} else if (typeof options !== "object" || options === null) {
			throw new Error("Invalid tap options");
		}
		if (typeof options.name !== "string" || options.name === "") {
			throw new Error("Missing name for tap");
		}
		if (typeof options.context !== "undefined") {
			deprecateContext();
		}
		options = Object.assign({ type, fn }, options);
		options = this._runRegisterInterceptors(options);
		this._insert(options);
	}
```

typeof(stats) is webpack.compilation.MultiStats，MultiStats 到底指的什么呢。

**webpackStats**

```javascript
const webpackStats: IWebpackStats = ({ urls, stats, statsOptions = defaultOptions, isFirstCompile }) => {
  const statsJson = (stats as webpack.Stats).toJson({
    all: false,
    errors: true,
    warnings: true,
    timings: true,
  });
  // compatible with webpack 5
  ['errors', 'warnings'].forEach((jsonKey: string) => {
    (statsJson as any)[jsonKey] = ((statsJson as any)[jsonKey] || []).map((item: string | IJsonItem ) => ((item as IJsonItem).message || item));
  });
  const messages = formatWebpackMessages(statsJson);
  const isSuccessful = !messages.errors.length;
  if (!process.env.DISABLE_STATS) {
    log.info('WEBPACK', stats.toString(statsOptions));
    if (isSuccessful) {
      // @ts-ignore
      if (stats.stats) {
        log.info('WEBPACK', 'Compiled successfully');
      } else {
        log.info(
          'WEBPACK',
          `Compiled successfully in ${(statsJson.time / 1000).toFixed(1)}s!`,
        );
      }
      if (isFirstCompile && urls) {
        console.log();
        log.info('WEBPACK', chalk.green('Starting the development server at:'));
        log.info('   - Local  : ', chalk.underline.white(urls.localUrlForBrowser));
        log.info('   - Network: ', chalk.underline.white(urls.lanUrlForTerminal));
        console.log();
      }
    } else if (messages.errors.length) {
      log.error('', messages.errors.join('\n\n'));
    } else if (messages.warnings.length) {
      log.warn('', messages.warnings.join('\n\n'));
    }
  }
  return isSuccessful;
};
```

所以没有错误就返回 true

**devServer**

```javascript
  const DevServer = require('webpack-dev-server');
  const devServer = new DevServer(compiler, devServerConfig);
 devServer.listen(devServerConfig.port, devServerConfig.host, async (err: Error) => {
    if (err) {
      log.info('WEBPACK',chalk.red('[ERR]: Failed to start webpack dev server'));
      log.error('WEBPACK', (err.stack || err.toString()));
    }

    await applyHook(`after.${command}.devServer`, {
      url: serverUrl,
      urls,
      devServer,
      err,
    });
  });

  return devServer;
}

```

现在看得比较粗略，明天的内容试一下 npm link ，看一下一些变量具体的值。

