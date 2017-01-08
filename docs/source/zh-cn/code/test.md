title: 单元测试
---

# 单元测试

> 测试的意义在于，在用户消费产出的代码之前，开发者首先消费它，给予其重要的质量保证。这里值得提醒的是，JavaScript开发者需要转变观念，正视自己的代码，对自己产出的代码负责。为自己的代码写测试用例则是一种行之有效的方法，它能够让开发者明确掌握到代码的行为和性能等。

web应用中的单元测试更加重要，在 web 产品快速迭代的时期，每个测试用例的覆盖都是一种可能的承诺。如果API升级时，测试用例可以很好地检查是否向下兼容。对于各种可能的输入，一旦测试覆盖，都能明确它的输出。代码改动后，可以通过测试结果判断代码的改动是否影响已确定的结果。

# 如何书写一个测试

## 工具

Egg 的命令行工具```egg-bin```使用 ```egg-bin test``` 执行测试，```egg-bin cov``` 计算测试用例覆盖率。

## 测试用例

### 维护用例

测试用例统一维护在``` /test ```目录下，命名为 ```xxxx.test.js```。

// test/egg.test.js

describe('egg test', function() {
  it('should ok', function(done) {
      assert(1===true);
  });
});

### 书写用例

#### 断言

平时我们都会使用 ```should.js```之类来写断言，egg内置了强大的断言工具power-assert，他有以下的优点：

- 没有 API 就是最好的 API，不需要任何记忆，只需 assert 即可。
- 强大的错误信息反馈

具体可以看这篇文章<https://github.com/atian25/blog/issues/16>

#### before和after

如果你需要对用例进行前置和后置处理，如mock数据，初始化应用等，可以使用```mocha```提供的一系列api。

下面是一个例子：

```javascript
describe('Egg', function() {
  before(() => console.log('order 1'));
  before(() => console.log('order 2'));
  after(() => console.log('order 6'));
  beforeEach(() => console.log('order 3'));
  afterEach(() => console.log('order 5'));
  it('should worker', () => console.log('order 4'));
});
```

每个用例按照 before -> beforeEach -> it -> afterEach -> after 的顺序执行

#### 异步测试

```javascript
it('fs.readFile should be ok', function (done) {
  fs.readFile('file_path', 'utf-8', function (err, data) {
    should.not.exist(err);
    done();
  });
});
```

#### 用例超时

mocha的默认超时时间为2000毫秒。一般情况下，通过`mocha -t <ms>`设置所有用例的超时时间。

若需更细粒度地设置超时时间，可以在测试用例it中调用`this.timeout(ms)`实现对单个用例的特殊设置，示例代码如下：

```javascript
it('should take less than 500ms', function (done) {
  this.timeout(500);
  setTimeout(done, 300);
});
```

也可以在描述 describe 中调用`this.timeout(ms)`设置描述下当前层级的所有用例：

```javascript
describe('a suite of tests', function(){
  this.timeout(500);
  it('should take less than 500ms', function (done) {
    setTimeout(done, 300);
  });

  it('should take less than 500ms as well', function (done) {
    setTimeout(done, 200);
  });
});
```

#### 覆盖率

#### mock 

在测试领域里，模拟数据或者异常有着一个特殊的名词：mock。我们通过伪造被调用方来测试上层代码的健壮性等。

// mm.mockSession

// mm.mockProxyError

#### egg-mock

# 常见测试场景

## web api测试

```javascript

const mm = require('egg-mock');
const request = require('supertest');
const assert = require('assert');

describe('Egg', function() {
  let app;
  before(function() {
    app = mm.app();
    return app.ready();
  });
  after(function() {
    app.close();
  });
  it('should return ok', function(done) {
    request(app.callback())
    .get('/status')
    .expect(200, function(err, res){
        assert(res.text === 'ok');
    });
  });
})

使用``` mm.app ```创建应用, 使用supertest发起请求，并判断```status```接口是否返回```'ok'```

```

## 插件测试

如果你需要给```egg```编写插件，那完善的测试是必不可少的。插件只需要在```package.json```中指定 ```eggPlugin``` 即可。
比如我需要写一个```egg-status```插件，对```/status```请求返回```'ok'```

```javascript
    const mm = require('egg-mock');
    const request = require('supertest');
    describe('Egg', function() {
        let app;
        before(function() {
            app = mm.app({
                baseDir: 'apps/egg-status',
            });
            return app.ready();
        });
        after(function() {
            app.close();
        });
        it('should return ok', function(done) {
            request(app.callback())
            .get('/status')
            .expect(200, function(err, res){
                assert(res.text === 'ok');
            });
        });
    })
```

### 命令行工具测试

可以使用coffee对命令行工具进行测试。

```javascript
'use strict';

const coffee = require('coffee');
const mm = require('egg-mock');
const should = require('should');
const path = require('path');
const fs = require('fs');

describe('test/command.test.js', function() {

  const command = require.resolve('../bin/command');
  afterEach(mm.restore);

  it('should list demo', function(done) {
    coffee.fork(command, ['-t', '/demo'])
      .expect('code', 0)
      .end(function(err, res) {
        assert(!err);
        done();
      });
  });

});

```
