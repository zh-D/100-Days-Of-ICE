开发一个前端脚手架的步骤是

1、开发模板，2、开发创建器，3、开发构建工具。

### 一、开发模板

直接 npm init ice template，选择 simple 模板，然后随便加个页面，以区分这是你的模板。

例如，我使用 icework 添加了一个名为 Test、路由为 /test 的页面。

### 二、开发创建器

找到 ice/packages/create-cli，这就是一个创建器。

我们就使用这个创建器。

根据 npm 文档，npm init ice xxx 会被翻译成 npm exec create-ice xxx，这个命令会先在本地找，然后再去远端找。我们把 create-ice 这个包改一下，让执行命令的时候找到它。

来看看怎么改,

首先，我们把 packag.json 里面的 bin 字段改一下：

```json
  "bin": {
    "create-myice": "lib/index.js"
  },
```

然后执行 npm link，现在我们执行 npm init myice xxx 就会找到这个包。

因为我想使用的是刚刚下载的那个模板，他在本地，所以我把 index 改成下面那样

```javascript
#!/usr/bin/env node
import * as path from 'path';
import { Command } from 'commander';
import create from './create';

(async function() {
  
  const program = new Command();

  program
    .name(`create-myice version myToy`)
    .usage('<command> [options]');

  program.on('--help', () => {
    console.log('');
    console.log('Examples:');
    console.log('  $ npm init myice');
    process.exit(0);
  });

  program.parse(process.argv);

  const dirname: string = program.args[0] ? program.args[0] : '.';

  console.log('create-myice version:', "toy version");
  console.log('create-myice args', dirname);

  const dirPath = path.join(process.cwd(), dirname);
  await create(dirPath, "toy-template", dirname);
})();

```

create 函数我改成下面这样：

```javascript
import * as inquirer from 'inquirer';
import * as fs from 'fs-extra';
import { downloadAndGenerateProject, checkEmpty } from '@iceworks/generate-project';

// eslint-disable-next-line
const chalk = require('chalk');

export default async function create(dirPath: string, templateName: string, dirname: string): Promise<void> {

  await fs.ensureDir(dirPath);
  const empty = await checkEmpty(dirPath);

  if (!empty) {
    const { go } = await inquirer.prompt({
      type: 'confirm',
      name: 'go',
      message:
        'The existing file in the current directory. Are you sure to continue ？',
      default: false,
    });
    if (!go) process.exit(1);
  }

  await downloadAndGenerateProject(dirPath, templateName);

  console.log();
  console.log('Initialize project successfully.');
  console.log();
  console.log('Starts the development server.');
  console.log();
  console.log(chalk.cyan(`    cd ${dirname}`));
  console.log(chalk.cyan('    npm install'));
  console.log(chalk.cyan('    npm start'));
  console.log();
}

```

link 一下 generate-project 这个包

downloadAndGenerateProject 改成这样：

```javascript
// 坑，先留着，感觉先要看 nodejs 的 fs 模块
```

