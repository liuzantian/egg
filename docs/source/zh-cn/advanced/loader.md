title: Loader
---

# Loader

egg 在 koa 的基础上进行增强最重要的就是基于一定的约定，根据功能差异将代码放到不同的目录下管理，对整体团队的开发成本提升有着明显的效果。Loader 实现了这套约定，并抽象了很多底层 API 可以进一步扩展。

## 应用、框架和插件

egg 是一个底层框架，应用可以直接使用，但 egg 本身的插件比较少，应用需要自己配置插件增加各种特性，比如 MySQL。

```js
// 应用配置
// package.json
{
  "dependencies": {
    "egg": "^1.0.0",
    "egg-mysql": "^1.0.0"
  }
}

// config/plugin.js
module.exports = {
  mysql: {
    enable: true,
    package: 'egg-mysql',
  },
}
```

当应用达到一定数量，我们会发现大部分应用的配置都是类似的，这时可以基于 egg 扩展出一个框架，应用的配置就会简化很多。

```js
// 框架配置
// package.json
{
  "name": "framework1",
  "version": "1.0.0",
  "dependencies": {
    "egg-mysql": "^1.0.0",
    "egg-view-nunjucks": "^1.0.0"
  }
}

// config/plugin.js
module.exports = {
  mysql: {
    enable: false,
    package: 'egg-mysql',
  },
  view: {
    enable: false,
    package: 'egg-view-nunjucks',
  }
}

// 应用配置
// package.json
{
  "dependencies": {
    "framework1": "^1.0.0",
  }
}

// config/plugin.js
module.exports = {
  // 开启插件
  mysql: true,
  view: true,
}
```

从上面的使用场景可以看到应用、插件和框架三者之间的关系。

- 我们在应用中完成业务，需要指定一个框架才能运行起来，当需要某个特性场景的功能时可以配置插件（比如 MySQL）。
- 插件只完成特定功能，当两个独立的功能有互相依赖时，还是分开两个插件，但需要配置依赖。
- 框架是一个启动器（默认就是 egg），必须有它才能运行起来。框架还是一个封装器，将插件的功能聚合起来统一提供，框架也可以配置插件。
- 在框架的基础上还可以扩展出新的框架，也就是说**框架是可以无限级继承的**，有点像类的继承。

```
+-----------------------------------+--------+
|      app1, app2, app3, app4       |        |
+-----+--------------+--------------+        |
|     |              |  framework3  |        |
+     |  framework1  +--------------+ plugin |
|     |              |  framework2  |        |
+     +--------------+--------------+        |
|                   egg             |        |
+-----------------------------------+--------|
|                   koa                      |
+-----------------------------------+--------+
```

## loadUnit

egg 将应用、框架和插件都称为加载单元（loadUnit），因为在代码结构上几乎没有什么差异，下面是目录结构

```
loadUnit
├── package.json
├── app.js
├── agent.js
├── app
│   ├── extend
│   |   ├── helper.js
│   |   ├── request.js
│   |   ├── response.js
│   |   ├── context.js
│   |   ├── application.js
│   |   └── agent.js
│   ├── service
│   ├── middleware
│   ├── router.js
│   └── init.js
└── config
    ├── config.default.js
    ├── config.prod.js
    ├── config.test.js
    ├── config.local.js
    └── config.unittest.js
```

不过还存在着一些差异

文件 | 应用 | 框架 | 插件
--- | --- | --- | ---
app/init.js | ✔︎ | |
app/router.js | ✔︎ | |
app/controller | ✔︎ | |
app/middleware | ✔︎ | ✔︎ | ✔︎
app/service | ✔︎ | ✔︎ | ✔︎
app/extend | ✔︎ | ✔︎ | ✔︎
app.js | ✔︎ | ✔︎ | ✔︎
agent.js | ✔︎ | ✔︎ | ✔︎
config/config.{serverEnv}.js | ✔︎ | ✔︎ | ✔︎
config/plugin.js | ✔︎ | ✔︎ |
package.json | ✔︎ | ✔︎ | ✔︎

在加载过程中，egg 会遍历所有的 loadUnit 加载上述的文件（应用、框架、插件各有不同），加载时有一定的优先级

- 按插件 => 框架 => 应用依次加载
- 插件之间的顺序由依赖关系决定，被依赖方先加载，无依赖按 object key 配置顺序加载，具体可以查看[插件章节](./plugin.md)
- 框架按继承顺序加载，越底层越先加载。

比如有这样一个应用配置了如下依赖

```
app
| ├── plugin2 (依赖 plugin3)
| └── plugin3
└── framework1
    | └── plugin1
    └── egg
```

最终的加载顺序为

```
=> plugin1
=> plugin3
=> plugin2
=> egg
=> framework1
=> app
```

plugin1 为 framework1 依赖的插件，配置合并后 object key 的顺序会优先级于 plugin2/plugin3。因为 plugin2 和 plugin3 的依赖关系，所以交换了位置。framework1 继承了 egg，顺序会晚于 egg。应用最后加载。

请查看 [Loader.getLoadUnits](https://github.com/eggjs/egg-core/blob/65ea778a4f2156a9cebd3951dac12c4f9455e636/lib/loader/egg_loader.js#L233) 方法

### 文件顺序

上面已经列出了默认会加载的文件，egg 会按如下文件顺序加载，每个文件或目录再根据 loadUnit 的顺序去加载（应用、框架、插件各有不同）。

- 加载 [plugin](./plugin.md)，找到应用和框架，加载 `config/plugin.js`
- 加载 [config](../basics/config.md), 遍历 loadUnit 加载 `config/config.{serverEnv}.js`
- 加载 [extend](../basics/extend.md), 遍历 loadUnit 加载 `app/extend/xx.js`
- [自定义初始化](../basics/app-start.md)，遍历 loadUnit 加载 `app.js` 和 `agent.js`
- 加载 [service](../basics/service.md), 遍历 loadUnit 加载 `app/service` 目录
- 加载 [middleware](../basics/middleware.md), 遍历 loadUnit 加载 `app/middleware` 目录
- 加载 [controller](../basics/router-controller.md), 加载应用的 `app/controller` 目录
- 加载 [router](../basics/router-controller.md), 加载应用的 `app/router.js`
- [应用自定义初始化](../basics/app-start.md)，加载 loadUnit 的 `app/init.js`

注意

- `app.js/agent.js` 和 `init.js` 使用的方法是一致的，但是加载时机不同。`init.js` 在整个 Loader 同步加载之后才加载是因为可以使用所有已加载的 API。比如应用初始化要调用 service 的一个方法，这时必须等 service 加载完才可以使用。
- 加载时如果遇到同名的会覆盖，比如想要覆盖 `ctx.ip` 可以直接在应用的 `app/extend/context.js` 定义 ip 就可以了。

## 扩展 Loader

[Loader] 是一个基类，并根据文件加载的规则提供了一些内置的方法，但基本本身并不会去调用，而是由继承类调用。

- loadPlugin()
- loadConfig()
- loadAgentExtend()
- loadApplicationExtend()
- loadRequestExtend()
- loadResponseExtend()
- loadContextExtend()
- loadHelperExtend()
- loadCustomAgent()
- loadCustomApp()
- loadService()
- loadMiddleware()
- loadController()
- loadRouter()
- loadAppInit()

egg 基于 Loader 实现了 [AppWorkerLoader] 和 [AgentWorkerLoader]，上层框架基于这两个类来扩展，**Loader 的扩展只能在框架进行**。

```js
// 自定义 AppWorkerLoader
// lib/loader.js
const AppWorkerLoader = require('egg').AppWorkerLoader;

class CustomAppWorkerLoader extends AppWorkerLoader {
  constructor(opt) {
    super(opt);
    // 自定义初始化
  }

  loadConfig() {
    super.loadConfig();
    // 对 config 进行处理
  }

  load() {
    super.load();
    // 自定义加载其他目录
    // 或对已加载的文件进行处理
  }
}

exports.AppWorkerLoader = CustomAppWorkerLoader;

// index.js
// 拷贝一份 egg 的 API
Object.assign(exports, require('egg'));
// 将自定义的 Loader exports 出来
exports.AppWorkerLoader = require('./lib/loader').AppWorkerLoader;
```

以上只是说明 Loader 的写法，具体可以查看[框架开发](./framework.md)。

## Loader API

Loader 还提供一些底层的 API，在扩展时可以简化代码，全部 API 请[查看](https://github.com/eggjs/egg-core#eggloader)

### loadFile

用于加载一个文件，比如加载 `app.js` 就是使用这个方法。

```js
// app/xx.js
module.exports = app => {
  console.log(app.config);
};

// app.js
// 以 app/xx.js 为例，我们可以在 app.js 加载这个文件
const path = require('path');
module.exports = app => {
  app.loader.loadFile(path.join(app.config.baseDir, 'app/xx.js'));
};
```

如果文件 export 一个函数会被调用，并将 app 作为参数，否则直接使用这个值。

### loadToApp

用于加载一个目录下的文件到 app，比如 `app/controller/home.js` 会加载到 `app.controller.home`。

```js
// app.js
// 以下只是示例，加载 controller 请用 loadController
module.exports = app => {
  const directory = path.join(app.config.baseDir, 'app/controller');
  app.loader.loadToApp(directory, 'controller');
};
```

一共有三个参数 `loadToApp(directory, property, LoaderOptions)`

1. directory 可以为 String 或 Array，Loader 会从这些目录加载文件
1. property 为 app 的属性
1. [LoaderOptions](#LoaderOptions) 为一些配置

### loadToContext

与 loadToApp 有一点差异，loadToContext 是加载到 ctx 上而非 app，而且是懒加载。加载时会将文件都放到一个临时对象上，在调用 ctx API 时才实例化对象。

比如 service 的加载就是使用这种模式

```js
// 以下为示例，请使用 loadService
// app/service/user.js
module.exports = app => {
  return class UserService extends app.Service {};
};

// 获取所有的 loadUnit
const servicePaths = app.loader.getLoadUnits().map(unit => {
  return path.join(unit.path, 'app/service');
});

app.loader.loadToContext(servicePaths, 'service', {
  // service 需要继承 app.Service，所以要拿到 app 参数
  // 设置 call 在加载时会调用函数返回 UserService
  call: true,
  // 将文件加载到 app.serviceClasses
  fieldClass: 'serviceClasses',
});
```

文件加载后 `app.serviceClasses.user` 就是 UserService，当调用 `ctx.service.user` 时会实例化 UserService，
所以这个类只有每次请求中首次访问时才会实例化，实例化后会被缓存，同一个请求多次调用也只会实例化一次。

### LoaderOptions

#### ignore [String]

ignore 可以忽略一些文件，支持 glob，默认为空

```js
app.loader.loadToApp(directory, 'controller', {
  // 忽略 app/controller/util 下的文件
  ignore: 'util/**',
});
```

#### initializer [Function]

对每个文件 export 出来的值进行处理，默认为空

```js
// app/model/user.js
module.exports = class User {
  constructor(app, path) {}
}

// 从 app/model 目录加载，加载时可做一些初始化处理
const directory = path.join(app.config.baseDir, 'app/model');
app.loader.loadToApp(directory, 'model', {
  initializer(model, opt) {
    // 第一个参数为 export 的对象
    // 第二个参数为一个对象，只包含当前文件的路径
    return new model(app, opt.path);
  },
});
```

#### lowercaseFirst [Boolean]

是否强制将第一个字符转成小写，默认为 false。

比如加载 `app/controller/User.js`，如果为 true 加载后生成 `app.controller.user`。

在加载不同文件时配置不同

文件 | 配置
--- | ---
app/controller | true
app/middleware | true
app/service | true

#### override [Boolean]

遇到已经存在的文件时是直接覆盖还是抛出异常，默认为 false

比如同时加载应用和插件的 `app/service/user.js` 文件，如果为 true 应用会覆盖插件的，否则加载应用的文件时会报错。

在加载不同文件时配置不同

文件 | 配置
--- | ---
app/controller | true
app/middleware | false
app/service | false

#### call [Boolean]

当 export 的对象为函数时则调用，并获取返回值，默认为 true

在加载不同文件时配置不同

文件 | 配置
--- | ---
app/controller | true
app/middleware | false
app/service | true

[Loader]: https://github.com/eggjs/egg-core/blob/master/lib/loader/egg_loader.js
[AppWorkerLoader]: https://github.com/eggjs/egg/blob/master/lib/loader/app_worker_loader.js
[AgentWorkerLoader]: https://github.com/eggjs/egg/blob/master/lib/loader/agent_worker_loader.js
