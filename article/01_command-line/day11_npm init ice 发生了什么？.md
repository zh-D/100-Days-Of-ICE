根据 npm init [文档](https://docs.npmjs.com/cli/init/)，npm init ice xxx 会被转化为 npm exec create-ice xxx。

根据 npm exec [文档](https://docs.npmjs.com/cli/v7/commands/npm-exec)，他会从本地或者是远程调用 create-ice 命令，因为本地肯定没有，这里当然是远程。

我们把 create-ice 下载到本地，然后 npm link 把它注册，并且生成 lib 目录。

现在 npm init ice xxx 使用的就是本地的 create-ice 了。

我们发现使用的模板来自于 @iceworks/generate-project 这个包，

```javascript
import { downloadAndGenerateProject, checkEmpty } from '@iceworks/generate-project';
// 下载模板的代码
await downloadAndGenerateProject(dirPath, templateName);
```

看一下这个 `downloadAndGenerateProject`方法

```javascript
export async function downloadAndGenerateProject(
  projectDir: string,
  npmName: string,
  version?: string,
  registry?: string,
  projectName?: string,
  ejsOptions?: IEjsOptions,
): Promise<void> {
  registry = registry || (await getNpmRegistry(npmName));

  // 根据模板创建项目支持的参数
  ejsOptions = {
    targets: ['web'],
    miniappType: 'runtime',
    mpa: false,
    pha: false,
    ...ejsOptions,
  };

  let tarballURL: string;
  try {
    tarballURL = await getNpmTarball(npmName, version || 'latest', registry);
  } catch (error) {
    if (error.statusCode === 404) {
      return Promise.reject(new Error(`获取模板 npm 信息失败，当前的模板地址是：${registry}/${npmName}。`));
    } else {
      return Promise.reject(error);
    }
  }

  console.log('download tarballURL', tarballURL);

  const spinner = ora('download npm tarball start').start();
  await getAndExtractTarball(
    projectDir,
    tarballURL,
    (state) => {
      spinner.text = `download npm tarball progress: ${Math.floor(state.percent * 100)}%`;
    },
  );
  spinner.succeed('download npm tarball successfully.');

  try {
    await formatScaffoldToProject(projectDir, projectName, ejsOptions);
  } catch (err) {
    console.warn('format scaffold to project error', err.message);
  }
}

```

一段一段稍微看一下：

**第一：**

```javascript
registry = registry || (await getNpmRegistry(npmName));
```

getNpmRegistry(npmName) 是这样的：

```javascript
// npmName 就是 templateName
async function getNpmRegistry(npmName: string): Promise<string> {
  if (process.env.REGISTRY) {
    return process.env.REGISTRY;
  } else if (isAliNpm(npmName)) {
    return ALI_NPM_REGISTRY;
  } else {
    return 'https://registry.npm.taobao.org';
  }
}
```

isAliNpm 这个包来自于 ice-npm-utils:

```javascript
function isAliNpm(npmName?: string): boolean {
  return /^(@alife|@ali|@alipay|@kaola)\//.test(npmName);
}
```

**第二：**

```javascript
  ejsOptions = {
    targets: ['web'],
    miniappType: 'runtime',
    mpa: false,
    pha: false,
    ...ejsOptions,
  };
```

**第三：**

下载 URL

```javascript
  let tarballURL: string;
  try {
    tarballURL = await getNpmTarball(npmName, version || 'latest', registry);
  } catch (error) {
    if (error.statusCode === 404) {
      return Promise.reject(new Error(`获取模板 npm 信息失败，当前的模板地址是：${registry}/${npmName}。`));
    } else {
      return Promise.reject(error);
    }
  }

  console.log('download tarballURL', tarballURL);
```

**第四：**

获得和提取 tarball

spinner 是转圈圈提升用户体验

```javascript
  const spinner = ora('download npm tarball start').start();  
  await getAndExtractTarball(
    projectDir,
    tarballURL,
    (state) => {
      spinner.text = `download npm tarball progress: ${Math.floor(state.percent * 100)}%`;
    },
  );
  spinner.succeed('download npm tarball successfully.');
```

看一下这个 getAndExtractTarball

意思就是下好了进行文件读写

```javascript
function getAndExtractTarball(
  destDir: string,
  tarball: string,
  // eslint-disable-next-line
  progressFunc = (state) => {},
  formatFilename = (filename: string): string => {
    // 为了兼容
    if (filename === '_package.json') {
      return filename.replace(/^_/, '');
    } else {
      return filename.replace(/^_/, '.');
    }
  },
): Promise<string[]> {
  return new Promise((resolve, reject) => {
    const allFiles = [];
    const allWriteStream = [];
    const dirCollector = [];

    progress(
      request({
        url: tarball,
        timeout: 10000,
      }),
    )
      .on('progress', progressFunc)
      .on('error', reject)
      // @ts-ignore
      .pipe(zlib.Unzip())
      // @ts-ignore
      .pipe(new tar.Parse())
      .on('entry', (entry) => {
        if (entry.type === 'Directory') {
          entry.resume();
          return;
        }

        const realPath = entry.path.replace(/^package\//, '');

        let filename = path.basename(realPath);
        filename = formatFilename(filename);

        const destPath = path.join(destDir, path.dirname(realPath), filename);
        const dirToBeCreate = path.dirname(destPath);
        if (!dirCollector.includes(dirToBeCreate)) {
          dirCollector.push(dirToBeCreate);
          mkdirp.sync(dirToBeCreate);
        }

        allFiles.push(destPath);
        allWriteStream.push(
          new Promise((streamResolve) => {
            entry
              .pipe(fs.createWriteStream(destPath))
              .on('finish', () => streamResolve())
              .on('close', () => streamResolve()); // resolve when file is empty in node v8
          }),
        );
      })
      .on('end', () => {
        if (progressFunc) {
          progressFunc({
            percent: 1,
          });
        }

        Promise.all(allWriteStream)
          .then(() => resolve(allFiles))
          .catch(reject);
      });
  });
}
```

**第五：**

格式化模板

```javascript
  try {
    await formatScaffoldToProject(projectDir, projectName, ejsOptions);
  } catch (err) {
    console.warn('format scaffold to project error', err.message);
  }
```

看一下 formatScaffoldToProject

上面是把文件都下载过来，这里就是 format 成想要的样子。

```javascript
export default async function formatScaffoldToProject(projectDir: string, projectName?: string, ejsOptions: any = {}) {
  // format filename
  const files = readFiles(projectDir);
  files.forEach((file) => {
    fse.renameSync(path.join(projectDir, file), path.join(projectDir, formatFilename(file)));
  });
  // render ejs template
  await ejsRenderDir(projectDir, ejsOptions);
  // format project
  await formatProject(projectDir, projectName, ejsOptions);
}
```

