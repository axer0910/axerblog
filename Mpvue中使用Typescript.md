---
title: Mpvue中使用Typescript
tag: frontend
date: 2019-04-23
updated: 2019-04-23
---


参考了 https://github.com/WingGao/mpvue-ts-demo.git 的结构，
Npm需要的包：
* `awesome-typescript-loader` ts文件的loader
* `typescript` typescript支持
* `vue` 原版Vue 2.5以上，为了支持ts中各种vue类型接口的支持
* `vue-property-decorator` Vue特性的装饰器
* `vuex-class` 提供Vuex装饰器支持

Webpack中MpvueEntry对ts支持的不是很好，所以小程序每个page下需要一个main.ts的入口，在webpack.base.conf.js添加到entry中去：
```javascript
function getEntry (rootSrc, pattern) {
  var files = glob.sync(path.resolve(rootSrc, pattern))
  return files.reduce((res, file) => {
    var info = path.parse(file)
    var key = info.dir.slice(rootSrc.length + 1) + '/' + info.name
    res[key] = path.resolve(file)
    return res
  }, {})
}

const appEntry = { app: resolve('./src/main.ts') } // 小程序app入口
const pagesEntry = getEntry(resolve('./src'), 'pages/**/main.ts') // 小程序每个page的入口
const entry = Object.assign({}, appEntry, pagesEntry)
```
添加ts和tsx支持：
```javascript
resolve: {
    extensions: ['.js', '.vue', '.json', '.ts'],
    alias: {
      '@': resolve('src'),
      vue: 'mpvue',
      flyio: 'flyio/dist/npm/wx',
      wx: resolve('src/utils/wx')
    },
    symlinks: false
  },
  module: {
    rules: [
      {
        test: /\.(js|vue)$/,
        loader: 'eslint-loader',
        enforce: 'pre',
        include: [resolve('src'), resolve('test')],
        options: {
          formatter: require('eslint-friendly-formatter')
        }
      },
      {
        test: /\.vue$/,
        loader: 'mpvue-loader',
        options: vueLoaderConfig
      },
      {
        test: /\.tsx?$/,
        // include: [resolve('src'), resolve('test')],
        exclude: /node_modules/,
        use: [
          'babel-loader',
          {
            loader: 'mpvue-loader',
            options: {
              checkMPEntry: true
            }
          },
          {
            // loader: 'ts-loader',
            loader: 'awesome-typescript-loader',
            options: {
              // errorsAsWarnings: true,
              useCache: true,
            }
          }
        ]
      },
      ...
```

vue-loader.conf.js 添加lang='ts'支持
```javascript
module.exports = {
  loaders: Object.assign(utils.cssLoaders({
    sourceMap: isProduction
      ? config.build.productionSourceMap
      : config.dev.cssSourceMap,
    extract: isProduction
  }), {
    ts: [
      'babel-loader',
      {
        // loader: 'ts-loader',
        loader: 'awesome-typescript-loader',
        options: {
          // errorsAsWarnings: true,
          useCache: true,
        }
      }
    ]
  }),
  transformToRequire: {
    video: 'src',
    source: 'src',
    img: 'src',
    image: 'xlink:href'
  }
}
```