`markdown-it` 是一个 parser。它接收一些字符串，并且经过内部的 rule 函数处理之后，调用 render 之后输出 HTML 字符串。既然是接受字符串，那么如下所见

```js
// node.js
var md = require('markdown-it')();
var result = md.render('# markdown-it rulezz!');

// output "<h1>markdown-it rulezz!</h1>"

// browser
var md = require('markdown-it')();
var result = md.render('# markdown-it rulezz!');

// output "<h1>markdown-it rulezz!</h1>"
```

输入一定格式的字符串给 md，输出 HTML 字符串，只要将其 append 到 DOM Tree 中，即可完成渲染。简单易懂。

那么内部的 parse 机制到底是怎样的呢？先从入口文件 `markdown-it/lib/index.js` 入手。

```js
function MarkdownIt(presetName, options) {
  if (!(this instanceof MarkdownIt)) {
    return new MarkdownIt(presetName, options);
  }

  if (!options) {
    if (!utils.isString(presetName)) {
      options = presetName || {};
      presetName = 'default';
    }
  }

  this.inline = new ParserInline();

  this.block = new ParserBlock();

  this.core = new ParserCore();

  this.renderer = new Renderer();

  this.linkify = new LinkifyIt();

  this.validateLink = validateLink;

  this.normalizeLink = normalizeLink;

  this.normalizeLinkText = normalizeLinkText;

  this.utils = utils;

  this.helpers = utils.assign({}, helpers);

  this.options = {};
  this.configure(presetName);

  if (options) { this.set(options); }
}
```

- MarkdownIt 支持传入两个参数 presetName 和 options.

  首先，构造函数内部先对调用方是否用了 new 操作符来调用 MarkdownIt 做了兼容。并且对 presetName 与 options 做了缺省值的一些操作。先来看下 presetName。

  ```js
  // 构造函数调用 configure
  this.configure(presetName)

  var config = {
    'default': require('./presets/default'),
    zero: require('./presets/zero'),
    commonmark: require('./presets/commonmark')
  };

  MarkdownIt.prototype.configure = function (presets) {
    var self = this, presetName;

    if (utils.isString(presets)) {
      presetName = presets;
      presets = config[presetName];
      if (!presets) { throw new Error('Wrong `markdown-it` preset "' + presetName + '", check name'); }
    }

    if (!presets) { throw new Error('Wrong `markdown-it` preset, can\'t be empty'); }

    if (presets.options) { self.set(presets.options); }

    if (presets.components) {
      Object.keys(presets.components).forEach(function (name) {
        if (presets.components[name].rules) {
          self[name].ruler.enableOnly(presets.components[name].rules);
        }
        if (presets.components[name].rules2) {
          self[name].ruler2.enableOnly(presets.components[name].rules2);
        }
      });
    }
    return this;
  }
  ```

  configure 方法接收 presets 作为参数，内部会对其做一些适配的工作。当 presets 为 `commonmark | default | zero` 的时候，就赋值为不同对象。这三种配置对象位于 `lib/presets` 下面，就拿 `default` 来说吧。

  ```js
  module.exports = {
    options: {
      html:         false,        // Enable HTML tags in source
      xhtmlOut:     false,        // Use '/' to close single tags (<br />)
      breaks:       false,        // Convert '\n' in paragraphs into <br>
      langPrefix:   'language-',  // CSS language prefix for fenced blocks
      linkify:      false,        // autoconvert URL-like texts to links

      // Enable some language-neutral replacements + quotes beautification
      typographer:  false,

      // Double + single quotes replacement pairs, when typographer enabled,
      // and smartquotes on. Could be either a String or an Array.
      //
      // For example, you can use '«»„“' for Russian, '„“‚‘' for German,
      // and ['«\xA0', '\xA0»', '‹\xA0', '\xA0›'] for French (including nbsp).
      quotes: '\u201c\u201d\u2018\u2019', /* “”‘’ */

      // Highlighter function. Should return escaped HTML,
      // or '' if the source string is not changed and should be escaped externaly.
      // If result starts with <pre... internal wrapper is skipped.
      //
      // function (/*str, lang*/) { return ''; }
      //
      highlight: null,

      maxNesting:   100            // Internal protection, recursion limit
    },

    components: {

      core: {},
      block: {},
      inline: {}
    }
  };
  ```

  导出了拥有 options 与 components 属性的对象。接下来就走到 configure 内部判断 options 与 components 的逻辑来。components 下面的顶级属性 core & block & inline 分别对应了以下三个实例，其目的是为了禁用它们内部的一些 rule 函数，让 Markdown 的 parse 更快，更纯粹。这也体现出 MarkdownIt 的灵活性以及高度定制的特性。

  ```js
  this.inline = new ParserInline();

  this.block = new ParserBlock();

  this.core = new ParserCore();

  this.renderer = new Renderer();
  ```

  这四个类，是整个 MarkdownIt 的灵魂。它们贯穿 MarkdownIt 的 parse、tokenize、render 的全流程，这四个类，我会在接下来的系列来一一分析。

- 接着初始化跟 url 相关的三个功能函数

  ```js
  this.validateLink = validateLink;

  this.normalizeLink = normalizeLink;

  this.normalizeLinkText = normalizeLinkText;
  ```

  主要是用来校验 url 的合法以及 decode 与 encode，感兴趣的同学，可以研究内部用到的 mdurl 以及 punycode 的逻辑。

- 最后部分就是一些 util 与 helpers 函数的挂载以及对 options 的配置。

从上面来看，整个初始化工作的逻辑非常清晰、明了，而且源码内部有大量的用法介绍，对开发者是非常的友好。

大致了解了构造函数的逻辑之后，我们看下 MarkdownIt 原型上的一些方法。

### set

+ 作用：合并 options

```js
MarkdownIt.prototype.set = function (options) {
  utils.assign(this.options, options);
  return this;
};
```

### configure

+ 作用：禁用 ParserInline、ParserBlock、ParserCore 的某些规则并且获取特定的 options

```js
MarkdownIt.prototype.configure = function (presets) {
  var self = this, presetName;

  if (utils.isString(presets)) {
    presetName = presets;
    presets = config[presetName];
    if (!presets) { throw new Error('Wrong `markdown-it` preset "' + presetName + '", check name'); }
  }

  ... // 省略部分代码
  return this;
};
```

### enable

+ 作用：开启 ParserInline、ParserBlock、ParserCore 的某些规则

```js
MarkdownIt.prototype.enable = function (list, ignoreInvalid) {
  var result = [];

  if (!Array.isArray(list)) { list = [ list ]; }

  [ 'core', 'block', 'inline' ].forEach(function (chain) {
    result = result.concat(this[chain].ruler.enable(list, true));
  }, this);

  ... // 省略部分代码
  return this;
};
```

### disable

+ 作用：关闭 ParserInline、ParserBlock、ParserCore 的某些规则

```js
MarkdownIt.prototype.disable = function (list, ignoreInvalid) {
  var result = [];

  if (!Array.isArray(list)) { list = [ list ]; }

  [ 'core', 'block', 'inline' ].forEach(function (chain) {
    result = result.concat(this[chain].ruler.disable(list, true));
  }, this);

  ... // 省略部分代码
  return this;
};
```

### use

+ 作用：注入插件

```js
// plugin 是一个函数，会将后面的参数传入 plugin 并且执行。
MarkdownIt.prototype.use = function (plugin /*, params, ... */) {
  var args = [ this ].concat(Array.prototype.slice.call(arguments, 1));
  plugin.apply(plugin, args);
  return this;
};
```

### parse

+ 作用：MarkdownIt 编译的入口

```js
// 1.，输入必须是字符串
// 2. State 是 CoreParser 的状态管理类，保存了 md 单例、src 编译字符串、tokens 词法单元等重要信息。
// 3.每种 Parser(InlineParser & BlockParser) 都有对应的状态管理类，内部的实现有一定区别。
MarkdownIt.prototype.parse = function (src, env) {
  if (typeof src !== 'string') {
    throw new Error('Input data should be a String');
  }

  var state = new this.core.State(src, this, env);

  this.core.process(state);

  return state.tokens;
};
```

### render

+ 作用：MarkdownIt 编译的出口，吐出 HTML 字符串

```js
// md 实例上是有 renderer 属性，可以理解为渲染器，对外暴露一些特定的 API，接收 tokens 来生成 HTML 字符串。
MarkdownIt.prototype.render = function (src, env) {
  env = env || {};

  return this.renderer.render(this.parse(src, env), this.options, env);
};
```

### parseInline

+ 作用：仅仅编译类型为 inline 的 token

```js
// md 实例上的 renderer，可以理解为渲染器，对外暴露一些特定的 API，接收 tokens 来生成 HTML 字符串。
MarkdownIt.prototype.parseInline = function (src, env) {
var state = new this.core.State(src, this, env);

  state.inlineMode = true;
  this.core.process(state);

  return state.tokens;
};
```

### renderInline

+ 作用：接收 parseInline 输出的 tokens，最终生成 HTML 字符串，不会被 p 标签包裹。

```js
MarkdownIt.prototype.renderInline = function (src, env) {
  env = env || {};

  return this.renderer.render(this.parseInline(src, env), this.options, env);
};

```

## 小结

从整体来看，MarkdownIt 的流程如下图：

![全流程](https://github.com/theniceangel/markdown-it-analysis/blob/master/images/markdown-it-flowchart.png?raw=true)

随着一步步对 MarkdownIt 的剖析之后，是不是惊叹于 Markdown 的架构设计，至少对于我来说，很多信息通过代码的组织以及变量命名就已经传达给我了，这也体现了阅读优秀源码的好处。

下一篇文章，将讲解 Ruler 以及 Token 这两个基础类。