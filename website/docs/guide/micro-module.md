# 微模块

微应用的更小粒度，通常是一个模块或页面，跟页面路由无关，可以随处挂载，也会出现多个微模块同时渲染运行。

## 通过脚手架创建

:::tip
如果是阿里内部的同学，参考文档 <a href="https://yuque.alibaba-inc.com/ice/rdy99p/mmhh1b">微模块开发接入 DEF</a>
:::

```shell
# 创建文件夹
$ mkdir micro-module & cd micro-module

# 初始化
$ iceworks init component @icedesign/template-icestark-module

# 安装依赖
$ npm install
$ npm start
```

### 模块开发

```shell
$ cd my-component
$ npm install
$ npm start
```

### 模块目录

```text
.
├── demo                  # 模块 demo
│   └── usage.md
├── src                   # 模块源码
│   ├── index.scss
│   └── index.tsx
├── lib/                  # 构建产物，编译为 ES5 的代码
├── es/                   # 构建产物，编译为 es module 规范的代码
├── dist/                 # 构建产物，编译为 umd 规范的代码
├── jest.config.js
├── build.json            # 构建配置
├── README.md
├── abc.json
├── package.json
└── tsconfig.json
```

#### 模块入口

模块入口文件为 `src/index.js` 。

```javascript
import React from 'react';

export default function ExampleComponent(props) {
  const { type, ...others } = props;

  return (
    <div className="ExampleComponent" {...others}>Hello ExampleComponent</div>
  );
}
```

#### 模块配置

模块开发工程需要在 `build.json` 中引入 `build-plugin-component` 和 `build-plugin-stark-module` 。

```json
// build.json
{
  "plugins": [
    "build-plugin-component",
    ["build-plugin-stark-module", {
      // ...options
    }]
  ]
}
```

`build-plugin-component` 配置请参考 [组件工程配置](https://ice.work/docs/materials/guide/component#%E7%BB%84%E4%BB%B6%E5%B7%A5%E7%A8%8B%E9%85%8D%E7%BD%AE)。 `build-plugin-stark-module` 的工程配置如下：

##### outputDir

构建结果目录。

- 类型： String
- 默认值： 'dist'
```json
// build.json
{
  "plugins": [
    ["build-plugin-stark-module", {
      "outputDir": 'build' // umd 构建结果打包至项目 build 目录
    }]
  ]
}
```
##### modules

配置多微模块入口。正常情况下，插件默认使用 `src/index.js` 作为微模块入口。


- 类型： object
- 默认值： {}

```json
// build.json
{
  "plugins": [
    ["build-plugin-stark-module", {
      "modules": {
        "branch-detail": "./src/branch-detail/index.tsx",
        "edit-info": "./src/edit-info/index.tsx"
      }
    }]
  ]
}
```

如上配置后，会在构建目录打包两个微模块资源。

```
.
├── branch-detail
│   ├── index.css
│   └── index.js
├── edit-info
	  ├── index.css
	  └── index.js
```

##### moduleExternals

构建时移除三方依赖。详见 [使用进阶 - 性能优化](#性能优化) 小节。

- 类型：object
- 默认值： {}

```json
{
  "plugins": [
    ...
    ["build-plugin-stark-module", {
      "moduleExternals": {
        "react": {
          "root": "React",
          "url": "https://g.alicdn.com/code/lib/react/16.14.0/umd/react.production.min.js",
        },
        "react-dom": {
          "root": "ReactDOM",
          "url": "https://g.alicdn.com/code/lib/react-dom/16.14.0/umd/react-dom.production.min.js"
        }
      }
    }],
  ...
  ]
}
```

##### filenameStrategy

默认情况，所有模块构建资源会平铺在构建目录下。通过该配置，可自定义产物目录结构，比如：

```json
{
  "plugins": [
    ...
    ["build-plugin-stark-module", {
      "outputDir": "build",
      "filenameStrategy": "modules/[name]"
    }],
  ...
  ]
}
```

上述配置会将产物构建值 `build/modules` 目录下，文件名为 `ModuleA.js` 、`ModuleB.js`。


## 已有项目改造为微模块

将已有项目改造为微模块的方式与 [微应用](./use-child/react) 类似，主要包含两步：

#### 1. 应用入口导出生命周期函数

下面是 React 模块的导出方式：

```js
import * as ReactDOM from 'react-dom';

const SampleModule = () => {
  return <div>Sample</div>;
}

// 声明 mount 生命周期
export function mount(ModuleComponent, targetNode, props) {
  ReactDOM.render(<ModuleComponent {...props} />, targetNode);
}

// 声明 unmount 生命周期
export function unmount(targetNode) {
  ReactDOM.unmountComponentAtNode(targetNode);
}

export default SampleModule;
```

下面是 Vue 2.x 模块导出的方式：

```js
import Vue from 'vue';
import SampleModule from './SampleModule';

let vue = null;

// 声明 mount 生命周期
export function mount(ModuleComponent, targetNode, props) {
  vue = new Vue({
    components: { ModuleComponent }
  }).$mount();

  // for vue don't replace mountNode
  container.innerHTML = '';
  container.appendChild(vue.$el);
}

// 声明 unmount 生命周期
export function unmount(targetNode) {
  vue && vue.$destroy();
}

export default SampleModule;
```

#### 2. 将模块构建为 UMD 产物

以 webpack 工程为例：

```js
module.exports = {
  output: {
    // 设置模块导出规范为 umd
    libraryTarget: 'umd',
    // 可选，设置模块在 window 上暴露的名称
    library: 'microApp',
  }
}
```

## 如何使用

### React 项目中使用

在 React 项目中使用微模块，我们推荐使用 `<MicroModule />` 组件快速接入。

```js
import { MicroModule } from '@ice/stark-module';

const App = () => {
  const moduleInfo = {
    name: 'moduleName',
    url: 'https://localhost/module.js',
  }
  return <MicroModule moduleInfo={moduleInfo} />;
}
```

也可以通过 `mountModule/unmountModule` 等 API 的方式接入。

```javascript
import { mountModule, unmoutModule } from '@ice/stark-module';
import { useRef } from 'useRef';

const moduleInfo = {
  name: 'moduleName',
  url: 'https://localhost/module.js',
};

const ModuleComponent = () => {
  const renderNode = useRef(null);
  useEffect(() => {
    mountModule(moduleInfo, renderNode.current, {});
    return () => {
      unmoutModule(moduleInfo, renderNode.current);
    }
  }, []);
  return (<div ref={renderNode}></div>);
};
```

### Vue 项目中使用

Vue 项目中需要使用 `mountModule/unmountModule` 的方式接入。

```html
<template>
  <div ref="mountNode"></div>
</template>

<script>
import { mountModule, unmoutModule } from '@ice/stark-module';

const moduleInfo = {
  name: 'moduleName',
  url: 'https://localhost/module.js',
};

export default {
  const mountNode = this.$refs.mountNode.$el;
	mounted () {
  	mountModule(moduleInfo, mountNode);
  },
  destroyed () {
  	unmoutModule(moduleInfo, mountNode)
  }
}
</script>
```

## 使用进阶

### 微模块中心化注册

如果需要前置中心化注册微模块，可以使用 `registerModules` 方法。

```js
import { MicroModule, registerModules, getModules } from '@ice/stark-module';

registerModules([
  {
    url: 'https://localhost/module-a.js',
    name: 'module-a',
  },
  {
    url: 'https://localhost/module-b.js',
    name: 'module-b',
  },
]);

const App = () => {
  // 中心化注册过，可以通过模块名直接指定要加载的微模块
  return (
    <div>
      <MicroModule moduleName="module-a" />
      <MicroModule moduleName="module-b" />
    </div>
  );
}
```

### 自定义生命周期

可以在微模块不导出 `mount` 和 `unmount` 的情况下，自定义生命周期。

```js
import { MicroModule } from '@ice/stark-module';
import ReactDOM from 'react-dom';

const App = () => {
  const moduleInfo = {
    name: 'moduleName',
    url: 'https://localhost/module.js',
    mount: (ModuleComponent, mountNode, props) => {
      console.log('custom mount');
      ReactDOM.render(<ModuleComponent />, mountNode, props);
    },
  }
  return <MicroModule moduleInfo={moduleInfo} />;
}
```

值得注意的是，若微模块导出了生命周期，其优先级高于自定义生命周期。

### 注册本地模块

在一些场景下，需要支持直接渲染内建模块，可以通过 `render` 属性直接渲染一个本地模块。

```js
import LocalComponent from './localComponent';

registerModules([{
  name: 'moduleName',
  render: () => LocalComponent,
}]);

const App = () => {
  return (
    <div>
      <MicroModule moduleName="moduleName" />
    </div>
  );
}
```

### 性能优化

通常情况下，为了减少模块体积，希望抽离一些公共的三方库。

**首先，微模块需要在构建时移除依赖的三方库。**

```javascript
// webpack.config.js
export default {
  ...
  externals: {
   react: {
      root: 'React',
      commonjs2: 'react',
      commonjs: 'react',
      amd: 'react',
    },
    'react-dom': {
      root: 'ReactDOM',
      commonjs2: 'react-dom',
      commonjs: 'react-dom',
      amd: 'react-dom',
    }
  }
}
```

**然后，在应用项目中，声明微模块的依赖**。

```javascript
import { MicroModule } from '@ice/stark-module';
import ReactDOM from 'react-dom';

const App = () => {
  const moduleInfo = {
    name: 'moduleName',
    url: 'https://localhost/module.js',
    // 声明模块 moduleName 所需的三方依赖
    runtime: [
      {
        id: "react@16",
        url: [
          "https://g.alicdn.com/code/lib/react/16.14.0/umd/react.production.min.js"
        ]
      },
      {
        id: "react-dom@16",
        url: [
          "https://g.alicdn.com/code/lib/react-dom/16.14.0/umd/react-dom.production.min.js"
        ]
      },
    ]
  }
  return <MicroModule moduleInfo={moduleInfo} />;
}
```

如果使用官方 `build-plugin-stark-module` 插件来构建的微模块，只需要在 build.json 中配置：

```javascript
{
  "plugins": [
    ...
    ["build-plugin-stark-module", {
      "moduleExternals": {
        "react": {
          "root": "React",
          "url": "https://g.alicdn.com/code/lib/react/16.14.0/umd/react.production.min.js",
        },
        "react-dom": {
          "root": "ReactDOM",
          "url": "https://g.alicdn.com/code/lib/react-dom/16.14.0/umd/react-dom.production.min.js"
        }
      }
    }],
	...
  ]
}
```

该插件会将三方依赖从模块中移除，并在产物目录生成一份依赖信息 `runtime.json`，模块发布时，需要将 `runtime.json` 一起发布。这样，在应用项目中，可以使用 `runtime.json` 作为依赖信息。

```javascript
import { MicroModule } from '@ice/stark-module';
import ReactDOM from 'react-dom';

const App = () => {
  const moduleInfo = {
    name: 'moduleName',
    url: 'https://localhost/module.js',
    // 声明模块 moduleName 需要的依赖文件地址
    runtime: 'https://xxx.com/moduleName.runtime.json'
  }
  return <MicroModule moduleInfo={moduleInfo} />;
}
```

## API

请移步 [API -> @ice/stark-module](../api/stark-module)
