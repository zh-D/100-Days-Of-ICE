## 准备工作

昨天跑了 simple 这个例子，打算用这个例子查看 ice 运行的参数，先把包 link 进来了。

### link 包

npm link ../../packages/icejs

npm link ../../packages/create-cli-utils

npm link ../../packages/build-scripts/packages/build-scripts --force

packages 里面没有 build-scripts 这个包，要自己下载 packages 里面 

需要 lib 的包就要给他 npm start

**我给每一个包都加上一个 console.log()，结果只有一个 icejs 这个包会 log 出信息。**

找到 icejs 发现它引用的 create-cli-utils 来自 icejs/node_modules，所以我在 icejs 里面 link create-cli-utils

```powershell
npm link ../create-cli-utils
```

启动项目后报错

```shell
Error: Cannot find module 'C:\Users\86155\Desktop\ice\packages\icejs\node_modules\create-cli-utils\lib\index.js'. Please verify that the package.json has a valid "main" entry
```

其实是没有 lib，所以我在 create-cli-utils 里面 npm start 一下，发现没有生成 lib ?

我发现 src 里面的代码本来就使用的是 commonJS，所以我把 main 改为 src/index 问题就解决了。

**打印 build-scripts 这个包的信息。**

同理，我们在 create-cli-utils 下面 npm link ../build-scripts/packages/build-scripts ，在 smple 里面 npm start 之后，相应信息都会 log 出来，表明现在所有包都 link 成功了，所以现在能够去对这些包进行操作了。

### 学习调试

文档：[调试](https://ice.work/docs/iceworks/guide/flow)

我发现调试好像只支持 src 下面的调试，在 packages 里面打的断点显示 "unbound breakpoint(未绑定的断点)"，没有办法调试。所以接下来我会使用 console.log 的方法去调试。

## 调试开始

### 找到 ice-cli.js，为之添加如下信息：

```diff
#!/usr/bin/env node
const utils = require('create-cli-utils');
const getBuiltInPlugins = require('../lib/getBuiltInPlugins');
const packageInfo = require('../package.json');

const forkChildProcessPath = require.resolve('./child-process-start');

(async () => {
+  console.log("1icejs");
+  console.log("packageInfo",packageInfo);
  await utils.createCli(getBuiltInPlugins, forkChildProcessPath, packageInfo);
})();

```

packageInfo 其实就是 icejs/package.json 里面的所有代码。

### 找到 utils.createCli，为之添加如下信息：

```javascript
console.log(process.argv);
console.log(program.runningCommand);
console.log(program.args);
// 结果：
[
  'C:\\Front-end\\node\\node.exe',
  'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\node_modules\\ice.js\\bin\\ice-cli.js',      
  'start'
]
undefined
undefined
```

### 找到 create-cli-utils/src/start.js

#### 先看 restartProcess(forkChildProcessPath);

根据:

const argv = await modifyInspectArgv(process.execArgv, rawArgv);

const nProcessArgv = process.argv.slice(2).filter((arg) => arg.indexOf('--inspect') === -1);

```javascript
    console.log(process.execArgv);
    console.log(rawArgv);
    console.log(argv);
    console.log(nProcessArgv);
    console.log(child);
[]
{ _: [ 'start' ] }
[]
[ 'start' ]
ChildProcess {
  _events: [Object: null prototype] { internalMessage: [Function] },
  _eventsCount: 1,
  _maxListeners: undefined,
  _closesNeeded: 2,
  _closesGot: 0,
  connected: true,
  signalCode: null,
  exitCode: null,
  killed: false,
  spawnfile: 'C:\\Front-end\\node\\node.exe',
  _handle: Process {
    onexit: [Function],
    pid: 22000,
    [Symbol(owner)]: [Circular]
  },
  spawnargs: [
    'C:\\Front-end\\node\\node.exe',
    'C:\\Users\\86155\\Desktop\\ice\\packages\\icejs\\bin\\child-process-start.js',
    'start'
  ],
  pid: 22000,
  stdin: null,
  stdout: null,
  stderr: null,
  stdio: [ null, null, null, null ],
  channel: Pipe {
    pendingHandle: null,
    sockets: { got: {}, send: {} },
    [Symbol(kJSONBuffer)]: '',
    [Symbol(kStringDecoder)]: undefined
  },
  _channel: [Getter/Setter],
  _handleQueue: null,
  _pendingMessage: null,
  send: [Function],
  _send: [Function],
  disconnect: [Function],
  _disconnect: [Function],
  [Symbol(kCapture)]: false
}
```

#### 回到 start，看一下watcher

```javascript
console.log(watcher)
FSWatcher {
  _events: [Object: null prototype] {},
  _eventsCount: 0,
  _maxListeners: undefined,
  _watched: Map {},
  _closers: Map {},
  _ignoredPaths: Set {},
  _throttled: Map {},
  _symlinkPaths: Map {},
  _streams: Set {},
  closed: false,
  _pendingUnlinks: Map {},
  _emitReady: [Function],
  _emitRaw: [Function],
  _readyEmitted: false,
  options: {
    ignoreInitial: true,
    persistent: true,
    ignorePermissionErrors: false,
    interval: 100,
    binaryInterval: 300,
    disableGlobbing: false,
    enableBinaryInterval: true,
    useFsEvents: false,
    usePolling: false,
    atomic: true,
    followSymlinks: true,
    awaitWriteFinish: false
  },
  _nodeFsHandler: NodeFsHandler { fsw: [Circular], _boundHandleError: [Function] },
  _userIgnored: [Function],
  _readyCount: 1,
  [Symbol(kCapture)]: false
}
```

### 再看 fork(forkChildProcessPath, nProcessArgv, { execArgv: argv });

所以我们来到 child-process-start-->utils.childProcessStart

```javascript
console.log(rawArgv)
{ _: [ 'start' ], port: 3333 }
```

### 再看 build-scripts/start

```javascript
console.log(context)
Context {
  registerConfig: [Function],
  getProjectFile: [Function],
  getUserConfig: [Function],
  mergeModeConfig: [Function],
  resolvePlugins: [Function],
  getAllPlugin: [Function],
  registerTask: [Function],
  cancelTask: [Function],
  registerMethod: [Function],
  applyMethod: [Function],
  hasMethod: [Function],
  modifyUserConfig: [Function],
  getAllTask: [Function],
  onGetWebpackConfig: [Function],
  onGetJestConfig: [Function],
  runJestConfig: [Function],
  onHook: [Function],
  applyHook: [Function],
  setValue: [Function],
  getValue: [Function],
  registerUserConfig: [Function],
  registerCliOption: [Function],
  runPlugins: [Function],
  checkPluginValue: [Function],
  runUserConfig: [Function],
  runCliOption: [Function],
  runWebpackFunctions: [Function],
  setUp: [Function],
  command: 'start',
  commandArgs: { port: 3333 },
  rootDir: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple',
  configArr: [],
  modifyConfigFns: [],
  modifyJestConfig: [],
  eventHooks: {},
  internalValue: {},
  userConfigRegistration: {},
  cliOptionRegistration: {
    port: { name: 'port', commands: [Array] },    host: { name: 'host', commands: [Array] },    disableAsk: { name: 'disableAsk', commands: [Array] },
    config: { name: 'config', commands: [Array] }
  },
  methodRegistration: {},
  cancelTaskNames: [],
  pkg: {
    name: 'example-simple',
    dependencies: { 'ice.js': '^1.0.0', react: '^16.4.1', 'react-dom': '^16.4.1' },
    devDependencies: { '@types/react': '^16.9.20', '@types/react-dom': '^16.9.5' },
    scripts: { start: 'icejs start', build: 'icejs build' },
    engines: { node: '>=8.0.0' }
  },
  userConfig: { entry: 'src/index.tsx', disableRuntime: true, plugins: [] },
  webpack: [Function: webpack] {
    version: '4.46.0',
    WebpackOptionsDefaulter: [Function: WebpackOptionsDefaulter],
    WebpackOptionsApply: [Function: WebpackOptionsApply],
    Compiler: [Function: Compiler],
    MultiCompiler: [Function: MultiCompiler], 
    NodeEnvironmentPlugin: [Function: NodeEnvironmentPlugin],
    validate: [Function: bound validateSchema],
    validateSchema: [Function: validateSchema],
    WebpackOptionsValidationError: [Function: 
WebpackOptionsValidationError],
    AutomaticPrefetchPlugin: [Getter],        
    BannerPlugin: [Getter],
    CachePlugin: [Getter],
    ContextExclusionPlugin: [Getter],
    ContextReplacementPlugin: [Getter],       
    DefinePlugin: [Getter],
    Dependency: [Getter],
    DllPlugin: [Getter],
    DllReferencePlugin: [Getter],
    EnvironmentPlugin: [Getter],
    EvalDevToolModulePlugin: [Getter],        
    EvalSourceMapDevToolPlugin: [Getter],     
    ExtendedAPIPlugin: [Getter],
    ExternalsPlugin: [Getter],
    HashedModuleIdsPlugin: [Getter],
    HotModuleReplacementPlugin: [Getter],     
    IgnorePlugin: [Getter],
    LibraryTemplatePlugin: [Getter],
    LoaderOptionsPlugin: [Getter],
    LoaderTargetPlugin: [Getter],
    MemoryOutputFileSystem: [Getter],
    Module: [Getter],
    ModuleFilenameHelpers: [Getter],
    NamedChunksPlugin: [Getter],
    NamedModulesPlugin: [Getter],
    NoEmitOnErrorsPlugin: [Getter],
    NormalModuleReplacementPlugin: [Getter],  
    PrefetchPlugin: [Getter],
    ProgressPlugin: [Getter],
    ProvidePlugin: [Getter],
    SetVarMainTemplatePlugin: [Getter],       
    SingleEntryPlugin: [Getter],
    SourceMapDevToolPlugin: [Getter],
    Stats: [Getter],
    Template: [Getter],
    UmdMainTemplatePlugin: [Getter],
    WatchIgnorePlugin: [Getter],
    dependencies: { DependencyReference: [Getter] },
    optimize: {
      AggressiveMergingPlugin: [Getter],      
      AggressiveSplittingPlugin: [Getter],    
      ChunkModuleIdRangePlugin: [Getter],     
      LimitChunkCountPlugin: [Getter],        
      MinChunkSizePlugin: [Getter],
      ModuleConcatenationPlugin: [Getter],    
      OccurrenceOrderPlugin: [Getter],        
      OccurrenceModuleOrderPlugin: [Getter],  
      OccurrenceChunkOrderPlugin: [Getter],   
      RuntimeChunkPlugin: [Getter],
      SideEffectsFlagPlugin: [Getter],        
      SplitChunksPlugin: [Getter],
      UglifyJsPlugin: [Getter],
      CommonsChunkPlugin: [Getter]
    },
    web: {
      FetchCompileWasmTemplatePlugin: [Getter],
      JsonpTemplatePlugin: [Getter]
    },
    webworker: { WebWorkerTemplatePlugin: [Getter] },
    node: {
      NodeTemplatePlugin: [Getter],
      ReadFileCompileWasmTemplatePlugin: [Getter]
    },
    debug: { ProfilingPlugin: [Getter] },     
    util: { createHash: [Getter] }
  },
  plugins: [
    {
      name: 'build-plugin-react-app',
      pluginPath: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\node_modules\\build-plugin-react-app\\lib\\index.js',
      fn: [Function],
      options: undefined
    },
    {
      name: 'build-plugin-ice-mpa',
      pluginPath: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\node_modules\\build-plugin-ice-mpa\\lib\\index.js',
      fn: [Function: plugin],
      options: undefined
    }
  ]
}
```

```javascript
console.log("args",args);
console.log("configArr",configArr);(configArr = await context.setUp();)
console.log("webpack",webpack);
console.log("compiler",compiler);

args { port: 3333 }
configArr [
  {
    name: '',
    chainConfig: {
      parent: undefined,
      store: [Map],
      devServer: [Object],
      entryPoints: [Object],
      module: [Object],
      node: [Object],
      optimization: [Object],
      output: [Object],
      performance: [Object],
      plugins: [Object],
      resolve: [Object],
      resolveLoader: [Object],
      shorthands: [Array],
      amd: [Function],
      bail: [Function],
      cache: [Function],
      context: [Function],
      devtool: [Function],
      externals: [Function],
      loader: [Function],
      mode: [Function],
      name: [Function],
      parallelism: [Function],
      profile: [Function],
      recordsInputPath: [Function],
      recordsPath: [Function],
      recordsOutputPath: [Function],
      stats: [Function],
      target: [Function],
      watch: [Function],
      watchOptions: [Function]
    },
    modifyFunctions: [ [Function], [Function] 
]
  }
]
webpack [Function: webpack] {
  version: '4.46.0',
  WebpackOptionsDefaulter: [Function: WebpackOptionsDefaulter],
  WebpackOptionsApply: [Function: WebpackOptionsApply],
  Compiler: [Function: Compiler],
  MultiCompiler: [Function: MultiCompiler],   
  NodeEnvironmentPlugin: [Function: NodeEnvironmentPlugin],
  validate: [Function: bound validateSchema], 
  validateSchema: [Function: validateSchema], 
  WebpackOptionsValidationError: [Function: WebpackOptionsValidationError],
  AutomaticPrefetchPlugin: [Getter],
  BannerPlugin: [Getter],
  CachePlugin: [Getter],
  ContextExclusionPlugin: [Getter],
  ContextReplacementPlugin: [Getter],
  DefinePlugin: [Getter],
  Dependency: [Getter],
  DllPlugin: [Getter],
  DllReferencePlugin: [Getter],
  EnvironmentPlugin: [Getter],
  EvalDevToolModulePlugin: [Getter],
  EvalSourceMapDevToolPlugin: [Getter],       
  ExtendedAPIPlugin: [Getter],
  ExternalsPlugin: [Getter],
  HashedModuleIdsPlugin: [Getter],
  HotModuleReplacementPlugin: [Getter],       
  IgnorePlugin: [Getter],
  LibraryTemplatePlugin: [Getter],
  LoaderOptionsPlugin: [Getter],
  LoaderTargetPlugin: [Getter],
  MemoryOutputFileSystem: [Getter],
  Module: [Getter],
  ModuleFilenameHelpers: [Getter],
  NamedChunksPlugin: [Getter],
  NamedModulesPlugin: [Getter],
  NoEmitOnErrorsPlugin: [Getter],
  NormalModuleReplacementPlugin: [Getter],    
  PrefetchPlugin: [Getter],
  ProgressPlugin: [Getter],
  ProvidePlugin: [Getter],
  SetVarMainTemplatePlugin: [Getter],
  SingleEntryPlugin: [Getter],
  SourceMapDevToolPlugin: [Getter],
  Stats: [Getter],
  Template: [Getter],
  UmdMainTemplatePlugin: [Getter],
  WatchIgnorePlugin: [Getter],
  dependencies: { DependencyReference: [Getter] },
  optimize: {
    AggressiveMergingPlugin: [Getter],        
    AggressiveSplittingPlugin: [Getter],      
    ChunkModuleIdRangePlugin: [Getter],       
    LimitChunkCountPlugin: [Getter],
    MinChunkSizePlugin: [Getter],
    ModuleConcatenationPlugin: [Getter],      
    OccurrenceOrderPlugin: [Getter],
    OccurrenceModuleOrderPlugin: [Getter],    
    OccurrenceChunkOrderPlugin: [Getter],     
    RuntimeChunkPlugin: [Getter],
    SideEffectsFlagPlugin: [Getter],
    SplitChunksPlugin: [Getter],
    UglifyJsPlugin: [Getter],
    CommonsChunkPlugin: [Getter]
  },
  web: {
    FetchCompileWasmTemplatePlugin: [Getter], 
    JsonpTemplatePlugin: [Getter]
  },
  webworker: { WebWorkerTemplatePlugin: [Getter] },
  node: {
    NodeTemplatePlugin: [Getter],
    ReadFileCompileWasmTemplatePlugin: [Getter]
  },
  debug: { ProfilingPlugin: [Getter] },       
  util: { createHash: [Getter] }
}
compiler MultiCompiler {
  _pluginCompat: SyncBailHook {
    _args: [ 'options' ],
    taps: [ [Object], [Object] ],
    interceptors: [],
    call: [Function: lazyCompileHook],        
    promise: [Function: lazyCompileHook],     
    callAsync: [Function: lazyCompileHook],   
    _x: undefined
  },
  hooks: {
    done: SyncHook {
      _args: [Array],
      taps: [],
      interceptors: [],
      call: [Function: lazyCompileHook],      
      promise: [Function: lazyCompileHook],   
      callAsync: [Function: lazyCompileHook], 
      _x: undefined
    },
    invalid: MultiHook { hooks: [Array] },    
    run: MultiHook { hooks: [Array] },        
    watchClose: SyncHook {
      _args: [],
      taps: [],
      interceptors: [],
      call: [Function: lazyCompileHook],      
      promise: [Function: lazyCompileHook],   
      callAsync: [Function: lazyCompileHook], 
      _x: undefined
    },
    watchRun: MultiHook { hooks: [Array] },   
    infrastructureLog: MultiHook { hooks: [Array] }
  },
  compilers: [
    Compiler {
      _pluginCompat: [SyncBailHook],
      hooks: [Object],
      name: undefined,
      parentCompilation: undefined,
      outputPath: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\build',
      outputFileSystem: [NodeOutputFileSystem],
      inputFileSystem: [CachedInputFileSystem],
      recordsInputPath: undefined,
      recordsOutputPath: undefined,
      records: {},
      removedFiles: Set {},
      fileTimestamps: Map {},
      contextTimestamps: Map {},
      resolverFactory: [ResolverFactory],     
      infrastructureLogger: [Function: logger],
      resolvers: [Object],
      options: [Object],
      context: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple',
      requestShortener: [RequestShortener],   
      running: false,
      watchMode: false,
      _assetEmittingSourceCache: [WeakMap],   
      _assetEmittingWrittenFiles: Map {},     
      watchFileSystem: [NodeWatchFileSystem], 
      watch: [Function],
      dependencies: undefined
    }
  ],
  running: false
}
```

```javascript
console.log(devServer)
Server {
  compiler: MultiCompiler {
    _pluginCompat: SyncBailHook {
      _args: [Array],
      taps: [Array],
      interceptors: [],
      call: [Function: lazyCompileHook],      
      promise: [Function: lazyCompileHook],   
      callAsync: [Function: lazyCompileHook], 
      _x: undefined
    },
    hooks: {
      done: [SyncHook],
      invalid: [MultiHook],
      run: [MultiHook],
      watchClose: [SyncHook],
      watchRun: [MultiHook],
      infrastructureLog: [MultiHook]
    },
    compilers: [ [Compiler] ],
    running: true
  },
  options: {
    port: 3333,
    host: '0.0.0.0',
    https: false,
    disableHostCheck: true,
    compress: true,
    transportMode: { server: 'ws', client: 'ws' },
    logLevel: 'silent',
    clientLogLevel: 'none',
    hot: true,
    publicPath: '/',
    quiet: false,
    watchOptions: { ignored: /node_modules/, aggregateTimeout: 600 },
    before: [Function: before],
    contentBase: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple',
    contentBasePublicPath: '/'
  },
  log: LogLevel {
    type: 'LogLevel',
    options: {
      name: 'wds',
      level: 'silent',
      prefix: [Object],
      factory: null,
      unique: true,
      timestamp: undefined
    },
    methodFactory: PrefixFactory {
      options: [Object],
      [Symbol(levels)]: [Object],
      [Symbol(instance)]: [Circular]
    },
    name: 'wds',
    currentLevel: 5,
    trace: [Function: noop],
    debug: [Function: noop],
    info: [Function: noop],
    warn: [Function: noop],
    error: [Function: noop],
    log: [Function: noop]
  },
  heartbeatInterval: 30000,
  socketServerImplementation: [Function: WebsocketServer],
  originalStats: {},
  sockets: [],
  contentBaseWatchers: [],
  hot: true,
  headers: undefined,
  progress: undefined,
  serveIndex: true,
  clientOverlay: undefined,
  clientLogLevel: 'none',
  publicHost: undefined,
  allowedHosts: undefined,
  disableHostCheck: true,
  watchOptions: { ignored: /node_modules/, aggregateTimeout: 600 },
  sockPath: '/sockjs-node',
  app: [Function: app] EventEmitter {
    _events: [Object: null prototype] { mount: [Function: onmount] },
    _eventsCount: 1,
    _maxListeners: undefined,
    setMaxListeners: [Function: setMaxListeners],
    getMaxListeners: [Function: getMaxListeners],
    emit: [Function: emit],
    addListener: [Function: addListener],     
    on: [Function: addListener],
    prependListener: [Function: prependListener],
    once: [Function: once],
    prependOnceListener: [Function: prependOnceListener],
    removeListener: [Function: removeListener],
    off: [Function: removeListener],
    removeAllListeners: [Function: removeAllListeners],
    listeners: [Function: listeners],
    rawListeners: [Function: rawListeners],   
    listenerCount: [Function: listenerCount], 
    eventNames: [Function: eventNames],       
    init: [Function: init],
    defaultConfiguration: [Function: defaultConfiguration],
    lazyrouter: [Function: lazyrouter],       
    handle: [Function: handle],
    use: [Function: use],
    route: [Function: route],
    engine: [Function: engine],
    param: [Function: param],
    set: [Function: set],
    path: [Function: path],
    enabled: [Function: enabled],
    disabled: [Function: disabled],
    enable: [Function: enable],
    disable: [Function: disable],
    acl: [Function],
    bind: [Function],
    checkout: [Function],
    connect: [Function],
    copy: [Function],
    delete: [Function],
    get: [Function],
    head: [Function],
    link: [Function],
    lock: [Function],
    'm-search': [Function],
    merge: [Function],
    mkactivity: [Function],
    mkcalendar: [Function],
    mkcol: [Function],
    move: [Function],
    notify: [Function],
    options: [Function],
    patch: [Function],
    post: [Function],
    propfind: [Function],
    proppatch: [Function],
    purge: [Function],
    put: [Function],
    rebind: [Function],
    report: [Function],
    search: [Function],
    source: [Function],
    subscribe: [Function],
    trace: [Function],
    unbind: [Function],
    unlink: [Function],
    unlock: [Function],
    unsubscribe: [Function],
    all: [Function: all],
    del: [Function],
    render: [Function: render],
    listen: [Function: listen],
    request: IncomingMessage { app: [Circular] },
    response: ServerResponse { app: [Circular] },
    cache: {},
    engines: {},
    settings: {
      'x-powered-by': true,
      etag: 'weak',
      'etag fn': [Function: generateETag],    
      env: 'development',
      'query parser': 'extended',
      'query parser fn': [Function: parseExtendedQueryString],
      'subdomain offset': 2,
      'trust proxy': false,
      'trust proxy fn': [Function: trustNone],      view: [Function: View],
      views: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\views',
      'jsonp callback name': 'callback'       
    },
    locals: [Object: null prototype] { settings: [Object] },
    mountpath: '/',
    _router: [Function: router] {
      params: {},
      _params: [],
      caseSensitive: false,
      mergeParams: undefined,
      strict: false,
      stack: [Array]
    }
  },
  middleware: [Function: middleware] {        
    close: [Function: close],
    context: {
      state: false,
      webpackStats: null,
      callbacks: [],
      options: [Object],
      compiler: [MultiCompiler],
      watching: [MultiWatching],
      forceRebuild: false,
      log: [LogLevel],
      rebuild: [Function: rebuild],
      fs: [MemoryFileSystem]
    },
    fileSystem: MemoryFileSystem { data: {} },    getFilenameFromUrl: [Function: bound getFilenameFromUrl],
    invalidate: [Function: invalidate],       
    waitUntilValid: [Function: waitUntilValid]  },
  websocketProxies: [],
  listeningApp: Server {
    insecureHTTPParser: undefined,
    _events: [Object: null prototype] {       
      request: [EventEmitter],
      connection: [Array],
      error: [Function],
      listening: [Function]
    },
    _eventsCount: 4,
    _maxListeners: undefined,
    _connections: 0,
    _handle: null,
    _usingWorkers: false,
    _workers: [],
    _unref: false,
    allowHalfOpen: true,
    pauseOnConnect: false,
    httpAllowHalfOpen: false,
    timeout: 120000,
    keepAliveTimeout: 5000,
    maxHeadersCount: null,
    headersTimeout: 40000,
    kill: [Function],
    [Symbol(IncomingMessage)]: [Function: IncomingMessage],
    [Symbol(ServerResponse)]: [Function: ServerResponse],
    [Symbol(kCapture)]: false,
    [Symbol(asyncId)]: -1
  },
  hostname: '0.0.0.0'
}
```

### context.constructor

```javascript
console.log("webpackPath",webpackPath);
console.log(BUILTIN_CLI_OPTIONS);
console.log(getBuiltInPlugins(this.userConfig));
console.log(this.plugins);
webpackPath webpack
[
  { name: 'port', commands: [ 'start' ] },    
  { name: 'host', commands: [ 'start' ] },    
  { name: 'disableAsk', commands: [ 'start' ] 
},
  { name: 'config', commands: [ 'start', 'build', 'test' ] }
]
[ 'build-plugin-react-app', 'build-plugin-ice-mpa' ]
[
  {
    name: 'build-plugin-react-app',
    pluginPath: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\node_modules\\build-plugin-react-app\\lib\\index.js',
    fn: [Function],
    options: undefined
  },
  {
    name: 'build-plugin-ice-mpa',
    pluginPath: 'C:\\Users\\86155\\Desktop\\ice\\examples\\simple\\node_modules\\build-plugin-ice-mpa\\lib\\index.js',
    fn: [Function: plugin],
    options: undefined
  }
]
```

