---
title: Webpack4 Vue2.6 + Typescript典型配置
tag: frontend
date: 2020-04-23
updated: 2020-04-23
---

最基本的能打包的配置：
基本结构
* webpack.base.js // 公用基础webpack配置
* webpack.dev.js // devserver配置
* webpack.prod.js // 生产打包配置

### webpack.base.js 基本配置要点：
* 入口(main.ts vue入口指定)
* 出口（生成的chunk指定路径，文件名，hash配置）
* resolve：路径别名（@路径），默认模块名称配置（当在代码里未指定后缀名自动搜索补全的后缀名称列表）
* module：各种类型的文件（模块）loader的配置
* plugins：webpack打包时使用的插件配置

### webpack.dev.js 配置要点：
* merge 上述base公用基础配置
* dev server监听端口，地址配置
* `HtmlWebpackPlugin` 插件配置，该插件用于dev server自动生成的入口index.html配置（包含挂载vue的主div元素，js文件路径注入（如果配置了插件inject为true则无需指定））

### webpack.prod.js 配置要点：
* merge 上述base公用基础配置
* 设置mode为production
* todo：公用模块拆分，css提取等

<!-- more -->

webpack.base.js:
```javascript
const path = require("path");
const HtmlWebpackPlugin = require('html-webpack-plugin');
const VueLoaderPlugin = require('vue-loader/lib/plugin');
const DeclarationPlugin = require('declaration-bundler-webpack-plugin');

module.exports = {
  entry: {
    index: path.resolve(__dirname,'../src/main.ts'),
  },
  output: {
    filename: '[name].[hash].js',
    path: path.resolve(__dirname,'../dist'),
  },
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.vue', '.json'],
    alias: {
      '@': path.resolve(__dirname,'../src'),
    }
  },
  module:{
    rules:[ // todo 拆分js，css，不打包element ui内容
      {
        test: /\.(js|vue)$/,
        loader: 'eslint-loader',
        enforce: 'pre',
        include: [path.resolve(__dirname,'../src')],
        options: {
        }
      },
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        enforce: 'pre',
        loader: 'tslint-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: Object.assign({}, {
          loaders: {
            ts: "ts-loader",
            tsx: "babel-loader!ts-loader"
          }
        })
      },
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        use: [
          "babel-loader",
          {
            loader: "ts-loader",
            options: { appendTsxSuffixTo: [/\.vue$/] }
          }
        ]
      },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        include: [path.resolve(__dirname,'../src'), path.resolve(__dirname,'../src')]
      },
      {
        test: /\.css$/,
        use: [
          "style-loader",
          "vue-style-loader",
          "css-loader"
        ]
      },{
        test: /\.(png|svg|jpg|gif|woff|ttf)$/,
        use:'url-loader',

      }
    ]
  },
  plugins:[
    new HtmlWebpackPlugin({
      title:'WebpackTest'
    }),
    new VueLoaderPlugin(),
    new DeclarationPlugin(
      {
        moduleName:'some.path.moduleName',
        out:'./builds/bundle.d.ts'
      }
    )
  ]
};

```

这里`tsx`的模块处理下`option`有一行配置：` appendTsxSuffixTo: [/\.vue$/]`非常重要，这行配置是将`vue`单页面组件文件中的<script lang="ts"></script>标签内所有ts代码关联到`tsx`类型文件，这样script标签内的所有代码才会被TypeScript所编译，不然可能在编译的时候出现没有默认module导出的情况（实际上就是只编译了模板，没有编译export default导出模块的代码）

webpack.dev.js

```javascript
const merge = require("webpack-merge");
const base = require("./webpack.base");
const webpack = require("webpack");
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = merge(base ,{
  mode: 'development',
  devtool: 'source-map',
  devServer:{
    compress: true, //启用压缩
    port: 1207,     //端口
    open: false,    //自动打开浏览器
    hot: true
  },
  plugins:[
    new HtmlWebpackPlugin({
      filename:'index.html',
      template:'template.html', // 手动创建一个template.html，里面包含一个<div id="app"></div>元素即可，更多配置可看插件文档
      inject:true
    }),
    new webpack.HotModuleReplacementPlugin()
  ]
});

```

webpack.prod.js
```javascript
const merge = require('webpack-merge');
const base = require("./webpack.base.js");
const CleanWebpackPlugin = require('clean-webpack-plugin');
const path = require("path");

// todo 自动清理dist，common模块拆分，优化等
module.exports = merge(base ,{
  mode: 'production',
  plugins: [],
})

```
tsconfig.json
```javascript
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "strict": true,
    "jsx": "preserve",
    "importHelpers": true,
    "moduleResolution": "node",
    "allowJs": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "sourceMap": true,
    "baseUrl": ".",
    "types": [
      "webpack-env",
      "jest"
    ],
    "paths": {
      "@/*": [
        "src/*"
      ]
    },
    "lib": [
      "esnext",
      "dom",
      "dom.iterable",
      "scripthost"
    ]
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "tests/**/*.ts",
    "tests/**/*.tsx"
  ],
  "exclude": [
    "node_modules"
  ]
}
```

webpack 启动dev server命令：`webpack-dev-server --config config/webpack.dev.js`
webpack 打生产包命令：`webpack --config config/webpack.prod.js`