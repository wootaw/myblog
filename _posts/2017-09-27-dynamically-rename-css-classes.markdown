---
layout: post
title:  "动态地重命名 CSS 类"
date:   2017-09-27 15:21:23 +0800
categories: css
---
你有没有看过[Google](https://www.google.com/)的源代码？您会注意到的第一件事是CSS类名称长度不超过几个字符。

![google.com HTML](https://cdn-images-1.medium.com/max/1600/1*mGuDYFM56iyLi1MgZPC8bw.png)

但是这个怎么做呢？

## CSS minifiers的缺点

minifier无法做到的一件事 - 更改选择器名称。这是因为CSS minifier不控制HTML输出。同时，CSS名称可能会变长。

如果您使用的是CSS模块，那么您的CSS模块可能会包含样式表文件名，本地标识符名称和随机哈希。使用css-loader localIdentName 配置描述类名模板，例如， [name]_[local]_[hash:base64:5]。因此，生成的类名称将如下所示 .MovieView_movie-title_yvKVV;如果你喜欢描述性的名字，它会变得更长，例如 .MovieView_movie-description-with-summary-paragraph_yvKVV。

## 在编译时重命名CSS类名

但是，如果您使用 webpack 和 babel-plugin-react-css-modules，你很幸运🍀 - 你可以使用 css-loader getLocalIdent 配置和等效的 babel-plugin-react-css-modules generateScopedName 配置在编译时重命名类名。

```js
const generateScopedName = (
  localName: string,
  resourcePath: string
) => {
  const componentName = resourcePath.split('/').slice(-2, -1);
  return componentName + '_' + localName;
};
```

关于generateScopedName的一个很酷的事情是该函数的同一个实例可以用于Babel和webpack构建过程：

```js
/**
 * @file Webpack configuration.
 */
const path = require('path');

const generateScopedName = (localName, resourcePath) => {
  const componentName = resourcePath.split('/').slice(-2, -1);

  return componentName + '_' + localName;
};

module.exports = {
  module: {
    rules: [
      {
        include: path.resolve(__dirname, '../app'),
        loader: 'babel-loader',
        options: {
          babelrc: false,
          extends: path.resolve(__dirname, '../app/webpack.production.babelrc'),
          plugins: [
            [
              'react-css-modules',
              {
                context: common.context,
                filetypes: {
                  '.scss': {
                    syntax: 'postcss-scss'
                  }
                },
                generateScopedName,
                webpackHotModuleReloading: false
              }
            ]
          ]
        },
        test: /\.js$/
      },
      {
        test: /\.scss$/,
        use: [
          {
            loader: 'css-loader',
            options: {
              camelCase: true,
              getLocalIdent: (context, localIdentName, localName) => {
                return generateScopedName(localName, context.resourcePath);
              },
              importLoaders: 1,
              minimize: true,
              modules: true
            }
          },
          'resolve-url-loader'
        ]
      }
    ]
  },
  output: {
    filename: '[name].[chunkhash].js',
    path: path.join(__dirname, './.dist'),
    publicPath: '/static/'
  },
  stats: 'minimal'
};
```

## 缩短名称

感谢 babel-plugin-react-css-modules 和 css-loader 共享相同的逻辑来生成CSS类名，我们可以将类名更改为我们喜欢的任何名称，甚至是随机哈希。但是，我想要最短的类名，而不是随机哈希。

为了生成最短的类名，我创建了类名索引，并使用 incstr 模块为索引中的每个条目生成增量ID。

```js
const incstr = require('incstr');

const createUniqueIdGenerator = () => {
  const index = {};

  const generateNextId = incstr.idGenerator({
    // Removed "d" letter to avoid accidental "ad" construct.
    // @see https://medium.com/@mbrevda/just-make-sure-ad-isnt-being-used-as-a-class-name-prefix-or-you-might-suffer-the-wrath-of-the-558d65502793
    alphabet: 'abcefghijklmnopqrstuvwxyz0123456789'
  });

  return (name) => {
    if (index[name]) {
      return index[name];
    }

    let nextId;

    do {
      // Class name cannot start with a number.
      nextId = generateNextId();
    } while (/^[0-9]/.test(nextId));

    index[name] = generateNextId();

    return index[name];
  };
};

const uniqueIdGenerator = createUniqueIdGenerator();

const generateScopedName = (localName, resourcePath) => {
  const componentName = resourcePath.split('/').slice(-2, -1);

  return uniqueIdGenerator(componentName) + '_' + uniqueIdGenerator(localName);
};
```

这保证了短而独特的类名。现在，而不是 .MovieView_movie-title_yvKVV 和 .MovieView_movie-description-with-summary-paragraph_yvKVV 我们的类名已经变成 .a_a，.b_a等。

## 使用Scope隔离来进一步减少bundle大小

我有一个很好的理由将_添加到CSS类名中，将组件名称和本地标识符名称分开 - 这种区别对于缩小很有用。

csso（CSS minifier）具有范围配置。范围定义了在某些标记上专门使用的类名列表，即来自不同范围的选择器与同一元素不匹配。此信息允许优化程序更积极地移动规则。

要利用这一点，请使用 csso-webpack-plugin 对CSS包进行后处理：

```js
const getScopes = (ast) => {
  const scopes = {};

  const getModuleID = (className) => {
    const tokens = className.split('_')[0];
  
    if (tokens.length !== 2) {
      return 'default';
    }

    return tokens[0];
  };

  csso.syntax.walk(ast, node => {
    if (node.type === 'ClassSelector') {
      const moduleId = getModuleID(node.name);

      if (moduleId) {
        if (!scopes[moduleId]) {
          scopes[moduleId] = [];
        }

        if (!scopes[moduleId].includes(node.name)) {
          scopes[moduleId].push(node.name);
        }
      }
    }
  });

  return Object.values(scopes);
};
```

## 这值得么？

反对这种缩小的第一个论点是压缩算法会为你做。与具有长类名的原始包相比，使用Brotli算法压缩的GO2CINEMA CSS包仅节省1 KB。

另一方面，设置此缩小是一次性投资，它减少了需要解析的文档的大小。它还有其他好处，例如阻止依赖CSS类名称的scapers导航或意外匹配广告拦截器黑名单的CSS选择器。

