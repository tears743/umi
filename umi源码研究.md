# UMI 源码研究

## 1、准备工作

+ 获取 [umi 源码](git@github.com:umijs/umi.git)( master 分支，当前最新版本3.1.0 )
  + fork 上述源码
  + clone 到本机
+ 调试准备
  + 工具 vscode
  + 调试配置
    ```javascript
       // v3.1.0
      {
        "version": "0.2.0",
        "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "umi_debug",
           
            "program": "${workspaceFolder}/packages/umi/bin/umi.js",
            "autoAttachChildProcesses": true,
            "args": [
            "dev",
            "--cwd=${workspaceFolder}/packages/test/bin/umi-test.js"
            ],
            "env": {
            "DEBUG":"umi-build-dev:Service"
            }
        }
        ]
    }
    ```
## 2、源码目录
  ### 目录结构
```
    umi
    |
    |---docs
    |---e2e
    |---memo
    |---packages
    |---scripts
```
------
          说明：
          目录结构到了3.x版本已经相当简洁，
          docs是一些说明文档，
          e2e没看明白是什么作用，
          memo是一些改动和注意事项的备注，
          packages是各个子模块（lerna），scripts脚本命令。
    
  ### package.json 分析
  
----
            一般情况下，独立的npm包中package.json文件都会有个main，或者bin之类的键来表示入口或者启动，umi使用lerna管理，package.json中没有找到相应字段，所以我们换个策略，看看script内容：
----        
    
```javascript
         "scripts": {
            "bootstrap": "node ./scripts/bootstrap.js",
            "build": "father-build",
            "docs": "node ./scripts/docs.js",
            "docs:build": "node ./packages/umi/bin/umi.js build",
            "docs:dev": "node ./packages/umi/bin/umi.js dev",
            "prettier": "prettier --write '**/*.{js,jsx,tsx,ts,less,md,json}'",
            "link:umi": "cd packages/umi && yarn link && cd -",
            "release": "node ./scripts/release.js",
            "test": "umi-test",
            "test:coverage": "umi-test --coverage",
            "sync:tnpm": "node -e 'require(\"./scripts/syncTNPM\")()'",
            "now-build": "echo \"Hello\"",
            "update:deps": "yarn upgrade-interactive --latest"
        }
```
---
          可以看到有release的命令和link命令，link中链接了目录 /packages/umi这个包，同时查看release.js脚本发现发布也是遍历了packages目录下的所有包进行发布，所以，可以定位到，我们使用umi时的命令应该在packages/umi这个包中，下面去看一下。
### packages/umi 入口分析
---
        packages/umi/package.json 看到 bin字段指向"bin/umi.js"，我们以此作为入口分析。
        在具体分析前，我们构建下整个包，构建流程：
        pre install： 安装lerna,
        yarn global add lerna
        1、yarn install
        2、lerna bootstrap
        3、yarn build
        下面我们看下目录中的umi.js
### bin/umi.js 分析 
    代码：(注释分析)
```javascript 

const resolveCwd = require('resolve-cwd');

const { name, bin } = require('../package.json');
// 解析本地命令行脚本
const localCLI = resolveCwd.silent(`${name}/${bin['umi']}`);
console.log('cli:',__filename,localCLI)
// 使用哪里的cli
if (!process.env.USE_GLOBAL_UMI && localCLI && localCLI !== __filename) {
  const debug = require('@umijs/utils').createDebug('umi:cli');
  debug('Using local install of umi');
  require(localCLI);
} else {
  require('../lib/cli');
}
// 最后都是执行/lib/cli
```            
---
  接着来看cli文件，为了方便我们看src目录下的cli

```javascript
import { join } from 'path';
import { chalk, yParser } from '@umijs/utils';
import { existsSync } from 'fs';
import { Service } from './ServiceWithBuiltIn';
import fork from './utils/fork';
import getCwd from './utils/getCwd';
import getPkg from './utils/getPkg';

// process.argv: [node, umi.js, command, args]
// version v, help h , version 为bool
const args = yParser(process.argv.slice(2), {
  alias: {
    version: ['v'],
    help: ['h'],
  },
  boolean: ['version'],
});
// args={_:[],xx:xx} _数组处理，这里的处理原因在下面说
if (args.version && !args._[0]) {
  args._[0] = 'version';
  const local = existsSync(join(__dirname, '../.local'))
    ? chalk.cyan('@local')
    : '';
  console.log(`umi@${require('../package.json').version}${local}`);
} else if (!args._[0]) {
  args._[0] = 'help';
}

(async () => {
  try {
    // 使用 args._[] 判断命令行指令的原因是因为：umi支持插件自定义命令，这里只有dev命令是内置固定命令，剩下命令全部使用插件主动注册的方式统一处理和调用，具体细节后面再看。  
    switch (args._[0]) {
      case 'dev':
          // 开启一个新进程运行dev脚本
        const child = fork({
          scriptPath: require.resolve('./forkedDev'),
        });
        // ref:
        // http://nodejs.cn/api/process/signal_events.html
        // 响应 ctrl + c等键盘指令终结运行
        process.on('SIGINT', () => {
          child.kill('SIGINT');
        });
        process.on('SIGTERM', () => {
          child.kill('SIGTERM');
        });
        break;
        // 其他主动注册命令此处无法判断，所以统一处理
      default:
        const name = args._[0];
        if (name === 'build') {
          process.env.NODE_ENV = 'production';
        }
        // 核心处理在这里面
        await new Service({
          cwd: (),
          pkg: getPkg(process.cwd()),
        }).run({
          name,
          args,
        });
        break;
    }
  } catch (e) {
    console.error(chalk.red(e.message));
    console.error(e.stack);
    process.exit(1);
  }
})();
// 到这里，我们基本明白脚本启动的核心是Service文件里面，下面我们主要分析一下dev命令的执行过程
```    
---
```javascript
// forkedDev.ts
// 这里我们只看核心的几行代码
const service = new Service({
      cwd: getCwd(),
      pkg: getPkg(process.cwd()),
    });
    // 仍然是使用service.run 方法启动。
    await service.run({
      name: 'dev',
      args,
    });
// 下面我们看 Service.ts 
```
```javascript 
// Service.ts
// 这里为了方便理解，我们按照逻辑来部分展示其中代码
/* 首先，上面的代码回忆一下，都是new 了一个Service 实例后，使用run方法，参数传入name[命令名称],args[命令参数]来执行对应命令。所以目前能知道两点，1、Service的构造函数
2、run方法
下面我们先看下构造函数都做了什么
*/
// ServiceWithBuildIn.ts
import { dirname, join } from 'path';
import { IServiceOpts, Service as CoreService } from '@umijs/core';
// 继承了core/Service
class Service extends CoreService {
  constructor(opts: IServiceOpts) {
    process.env.UMI_VERSION = require('../package').version;
    process.env.UMI_DIR = dirname(require.resolve('../package'));
    // 代理到core/Service的构造
    super({
      ...opts,
      presets: [
          // 默认加载preset-built-in内置预设
        require.resolve('@umijs/preset-built-in'),
        ...(opts.presets || []),
      ],
      // 默认加载/plugins/umiAlias插件
      plugins: [require.resolve('./plugins/umiAlias'), ...(opts.plugins || [])],
    });
  }
}
// 下面大家应该都知道要去core/Service 里面看看神奇的东西是怎么发生的了

```
---
```javascript
// /core/Service.ts
// 先看下构造函数
/*
* 还记得构造时传入的参数么？
* {
    ...opt,
    presets:[
        require.resolve('@umijs/preset-built-in'),
        ...(opts.presets || [])
        ],
    plugins: [require.resolve('./plugins/umiAlias'), ...(opts.plugins || [])]
}
展开下opt 
{
    cwd:'',
    pkg:'{workspace}/package.json',
    presets:[
        require.resolve('@umijs/preset-built-in'),
        ...(opts.presets || [])
        ],
    plugins: [require.resolve('./plugins/umiAlias'), ...(opts.plugins || [])]
}
*
*/
constructor(opts: IServiceOpts) {
    super();

    logger.debug('opts:');
    logger.debug(opts);
    this.cwd = opts.cwd || process.cwd();
    // repoDir should be the root dir of repo
    this.pkg = opts.pkg || this.resolvePackage();
    this.env = opts.env || process.env.NODE_ENV;

    assert(existsSync(this.cwd), `cwd ${this.cwd} does not exist.`);

    // register babel before config parsing
    this.babelRegister = new BabelRegister();

    // load .env or .local.env
    logger.debug('load env');
    this.loadEnv();

    // get user config without validation
    logger.debug('get user config');
    // 配置初始化
    this.configInstance = new Config({
      cwd: this.cwd,
      service: this,
      localConfig: this.env === 'development',
    });
    // 加载用户配置
    this.userConfig = this.configInstance.getUserConfig();
    logger.debug('userConfig:');
    logger.debug(this.userConfig);

    // get paths
    this.paths = getPaths({
      cwd: this.cwd,
      config: this.userConfig!,
      env: this.env,
    });
    logger.debug('paths:');
    logger.debug(this.paths);

    // setup initial presets and plugins
    const baseOpts = {
      pkg: this.pkg,
      cwd: this.cwd,
    };
    // 预设初始化
    this.initialPresets = resolvePresets({
      ...baseOpts,
      presets: opts.presets || [],
      userConfigPresets: this.userConfig.presets || [],
    });
    // 插件初始化
    this.initialPlugins = resolvePlugins({
      ...baseOpts,
      plugins: opts.plugins || [],
      userConfigPlugins: this.userConfig.plugins || [],
    });
    // babel 转码
    this.babelRegister.setOnlyMap({
      key: 'initialPlugins',
      value: lodash.uniq([
        ...this.initialPresets.map(({ path }) => path),
        ...this.initialPlugins.map(({ path }) => path),
      ]),
    });
    logger.debug('initial presets:');
    logger.debug(this.initialPresets);
    logger.debug('initial plugins:');
    logger.debug(this.initialPlugins);
  }
  /* 上面的注释，表示构造函数有几个流程，
  配置初始化=>获取用户配置=> 初始化预设=> 初始化插件=> babel转码配置
  这里暂停一下，回忆一下整个流程 
  1、umi dev => 运行工程
  2、bin/umi.js 命令行
  3、cli.ts 实际脚本执行，初始化service
    1）ServiceWithBuildIn 委托父类core/Service构造初始化
    2）core/Service 配置初始化
    3）core/Service 获取用户配置
    4）预设初始化
    5）插件初始化
    6）Balbel register 挂载require进行即时编译转码
  4、 service.run 执行
  */
```
### 初始化配置与获取用户配置

---
    通过上面的分析，我们已经知道加载配置的代码在哪里，下面我们详细看下：

``` javascript
// /core/Service.ts
// 构造一个config实例
 this.configInstance = new Config({
      cwd: this.cwd,
      service: this,
      localConfig: this.env === 'development',
    });
    // 获取用户配置
    this.userConfig = this.configInstance.getUserConfig();
    logger.debug('userConfig:');
    logger.debug(this.userConfig);
// 回忆下在使用umi时，会有几种配置方式以文件的形式存在，下面我们看下Config和其中的getUserConfig方法
```
---
```javascript
// /core/Config/Config.ts
// 构造函数
constructor(opts: IOpts) {
    // 设置路径
    this.cwd = opts.cwd || process.cwd();
    // service 设置
    this.service = opts.service;
    // 本地配置标志
    this.localConfig = opts.localConfig;
  }
  // 配置实例的构造比较简单，我们看下获取用户配置的代码
```
---
```javascript
// /core/Config/Config.ts

    getUserConfig() {
    // 获取配置文件，此方法在后面贴
    const configFile = this.getConfigFile();
    this.configFile = configFile;
    // 潜在问题：
    // .local 和 .env 的配置必须有 configFile 才有效
    if (configFile) {
      let envConfigFile;
      if (process.env.UMI_ENV) {
          //配置文件加前缀（这一特性可能是为了区分不同环境用的）
        envConfigFile = this.addAffix(configFile, process.env.UMI_ENV);
        if (!existsSync(join(this.cwd, envConfigFile))) {
          throw new Error(
            `get user config failed, ${envConfigFile} does not exist, but process.env.UMI_ENV is set to ${process.env.UMI_ENV}.`,
          );
        }
      }
      const files = [
        configFile,
        envConfigFile,
        this.localConfig && this.addAffix(configFile, 'local'),
      ]
        .filter((f): f is string => !!f)
        .map((f) => join(this.cwd, f))
        .filter((f) => existsSync(f));

      // clear require cache and set babel register
      // 配置文件读入并载入依赖
      const requireDeps = files.reduce((memo: string[], file) => {
          // 依赖合并（crequire）
        memo = memo.concat(parseRequireDeps(file));
        return memo;
      }, []);
      // 将依赖做babel转码
      requireDeps.forEach(cleanRequireCache);
      this.service.babelRegister.setOnlyMap({
        key: 'config',
        value: requireDeps,
      });

      // require config and merge
      // 做配置合并
      return this.mergeConfig(...this.requireConfigs(files));
    } else {
        // 没有用户配置文件返回空对象
      return {};
    }
  }
  //用户配置文件名
  const CONFIG_FILES = [
  '.umirc.ts',
  '.umirc.js',
  'config/config.ts',
  'config/config.js',
];
getConfigFile(): string | null {
    // TODO: support custom config file
    // 判断当前路径下是否存在用户自定义配置文件就是CONFIG_FILES了，
    const configFile = CONFIG_FILES.find((f) => existsSync(join(this.cwd, f)));
    // windows路径兼容
    return configFile ? winPath(configFile) : null;
  }

  // 用户配置的加载分析基本完成，总结一下
  /*
  1、配置构造主要设置配置上下文，包括service，path等
  2、预定几个用户配置文件路径进行查找
  3、加载配置依赖，并进行babel转码
  4、读入并合并配置，这里在读入的时候调用了cojmpatESModuleRequire的泛型方法，主要作用是对
  es模块做适配，这里涉及到babel的模块机制，大家感兴趣可以自行baidu进行了解。

  */
```
    


               