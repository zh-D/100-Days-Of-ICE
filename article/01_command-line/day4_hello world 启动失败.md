## hello-world

- 重新 fork ice
- npm install
- cd examples/hello-world
- npm install
- npm start

结果：

```shell
Error: Cannot find module 'create-cli-utils/lib/cli'
Require stack:
-C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\ice.js\bin\ice-cli.js
...
```

于是在 node_modules 找到 ice.js\bin\ice-cli.js，是下面这行代码报错：

```javascript
const createCli = require('create-cli-utils/lib/cli');
```

找到 create-cli-utils/lib 发现根本没有 cli 这个模块。

应该是 create-cli，所以我把上面这个模块改成：

```javascript
const createCli = require('create-cli-utils/lib/create-cli');
```

**再次 npm start**

结果出现：

```shell
ERR! Cannot find module '../../packages/plugin-fast-refresh/lib/index.js'
Require stack:
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\@alib\build-scripts\lib\core\Context.js
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\@alib\build-scripts\lib\commands\build.js
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\@alib\build-scripts\lib\index.js
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\create-cli-utils\lib\build.js
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\create-cli-utils\lib\index.js
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\ice.js\bin\child-process-start.js
```

fast-refresh 这个插件，它为什么要去 packages 里面找呢？

原来在 build.json 里面配置了。

```javascript
{
  "plugins": [
    [
      "../../packages/plugin-fast-refresh/lib/index.js"
    ]
  ]
}

```

在 plugin-fast-refresh/package.json 里面修改：

```diff
  "scripts": {
+    "start": "tsc",
    "test": "echo \"Error: run tests from root\" && exit 1"
  },
```

然后 npm install 然后 npm start，现在有了 lib。

现在我们回到 examples/hello-world，再一次 npm start

```shell
ERR! CONFIG Error: Cannot find module 'webpack'    
ERR! CONFIG Require stack:
ERR! CONFIG - C:\Users\86155\Desktop\ice\packages\plugin-fast-refresh\node_modules\@pmmmwh\react-refresh-webpack-plugin\lib\index.js
```

找到那一行代码：

```javascript
const { DefinePlugin, ModuleFilenameHelpers, ProvidePlugin, Template } = require('webpack');
```

这个 webpack 的路径显示为下：

```javascript
module "C:/Users/86155/AppData/Local/Microsoft/TypeScript/4.2/node_modules/webpack/types"
```

我把这个 webpack 给删掉：

再次 npm start，又失败了：还是原来的错误：

```javascript
ERR! CONFIG Error: Cannot find module 'webpack'
ERR! CONFIG Require stack:
ERR! CONFIG - C:\Users\86155\Desktop\ice\packages\plugin-fast-refresh\node_modules\@pmmmwh\react-refresh-webpack-plugin\lib\index.js
```

这个 webpack 的路径显示为下：

```shell
module "C:\Users\86155\AppData\Local\Microsoft\TypeScript\4.2\node_modules\@types\webpack\index"
```

我又删除这个，然后再 npm start，还是找不到 webpack，所以我在 plugin-fast-refresh 里 npm i webpack -D

然后再 npm start：

```shell
TypeError: Cannot read property 'get' of undefined
    at exports.provide (C:\Users\86155\Desktop\ice\packages\plugin-fast-refresh\node_modules\webpack\lib\util\MapHelpers.js:17:20)    at C:\Users\86155\Desktop\ice\packages\plugin-fast-refresh\node_modules\webpack\lib\DefinePlugin.js:289:51
```

所以，我该怎么办呢？

我把 build.json 的代码删除，然后 npm start。

```shell
ERR! CONFIG Fail to load plugin C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\build-plugin-react-app\src\index.js
ERR! CONFIG Error: Cannot find module 'webpack'
ERR! CONFIG Require stack:
ERR! CONFIG - C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\mini-css-extract-plugin\dist\index.js
```

看来 webpack 不装不行了。

npm install webpack -g 依然找不到 webpack，为什么？

我找到 require 命令的加载规则：

- 如果参数字符串以“/”开头，则表示加载的是一个位于绝对路径的模块文件。比如，`require('/home/marco/foo.js')`将加载`/home/marco/foo.js`。
- 如果参数字符串以“./”开头，则表示加载的是一个位于相对路径（跟当前执行脚本的位置相比）的模块文件。比如，`require('./circle')`将加载当前脚本同一目录的`circle.js`。
- 如果参数字符串不以“./“或”/“开头，则表示加载的是一个默认提供的核心模块（位于Node的系统安装目录中），或者一个位于各级node_modules目录的已安装模块（全局安装或局部安装）。

对于 C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\mini-css-extract-plugin\dist\index.js 执行 require("webpack") 命令

根据第三条规则，Node 会依次从以下位置找 webpack：

- /usr/local/lib/node/webpack
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\mini-css-extract-plugin\node_modules\webpack
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\node_modules\webpack
- C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\webpack
- C:\Users\86155\Desktop\ice\examples\node_modules\webpack
- C:\Users\86155\Desktop\ice\node_modules\webpack

全局安装没有用，所以我试试 npm install webpack --save-dev，然后 npm start

```shell
ice.js 1.7.0-7
C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\webpack\lib\Dependency.js:311
                throw new Error(
                ^

Error: module property was removed from Dependency (use compilation.moduleGraph.updateModule(dependency, module) instead)
    at ProvidedDependency.set (C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\webpack\lib\Dependency.js:311:9)
    at iterationDependencies (C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\@alib\build-scripts\node_modules\webpack\lib\Compilation.js:940:21)
```

现在，我去跑一下其它的 examples。

## basic-hooks-store

在终端 执行命令 git checkout feat/hooks-store，然后在example 里面可以找到 basic-hooks-store，因为我对这个 plugin 比较熟悉，所以我用这个来试一下。

在 build.json 里面会有这个：

```javascript
{
  "store": false,
  "plugins": [
    "build-plugin-hooks-store"
  ]
}
```

这个会告诉 icejs 要去运行这个插件。

其实 node_modules 里面的 build-plugin-hooks-store 是跑不起来的，所以我们去 plugin-hooks-store，npm i、npm start

然后在回来 npm link ../../packages/plugin-hooks-store，解决一下 template 的问题，最后再 npm start，就可以跑起来。可能是没有 webpack 相关的。

**所以，为什么 hello-world 跑不起来呢？**

我看到 packages/icejs/bin/ice-cli.js 和 hello-world 的 ice 里面的不一样，hello world 里面 ice.js 版本为 1.7.0-7，而 basic-hooks-store 里面是 1.0.0，里面的代码差别蛮大的，我猜想是 ice.js 的原因，所以我回到 hello-world， npm link 试一下。

```shell
npm link ../../packages/icejs --force
```

使用 --force 覆盖了 node_modules/bin 里面 icejs 命令，要不然执行这个命令会报错，**可是为什么要覆盖呢**？留个坑。

然后，在 packages/icejs 添加

```javascript
  "scripts": {
    "start": "tsc -w",
    "test": "echo \"Error: run tests from root\" && exit 1"
  },
```

npm install、npm start

最后再在 hello-world 里面跑，发现

```shell
ERR! CONFIG Fail to load plugin C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\build-plugin-react-app\src\index.js
ERR! CONFIG Error: Cannot find module 'webpack'
ERR! CONFIG Require stack:
ERR! CONFIG - C:\Users\86155\Desktop\ice\examples\hello-world\node_modules\mini-css-extract-plugin\dist\index.js
```

居然先不是 plugin-fast-refresh 说找不到 webpack，而是 mini-css-extract-plugin 找不到。难道 plugin-fast-refresh 找得到吗？

我再去跑一个 example 了。

## simple

cd 到 example/simple 里面，现在这个项目跑起来是这样的：

```shell
Error: Child compilation failed:
  Entry module not found: Error: Can't resolve   'C:\Users\86155\Desktop\ice\examples\simple  \public\index.html' in 'C:\Users\86155\Deskt  op\ice\examples\simple':
```

所以我加一个 html 文件试一试。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div id="ice-container"></div>
</body>
</html>
```

现在能跑起来了。