## 作用

`markdown-it` 是一个 parser。它接收一些字符串，并且按照指定的规则（Rule）输出一些 HTML 字符串。既然是接受字符串，那么

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

从上可以看到，传入一定格式的字符串给 md，它会返回 HTML 字符串，只要将其 append 到 DOM Tree 中，即可完成渲染。那么内部的 parse 机制到底是怎样的呢，先从入口文件 `markdown-it-lib/index.js` 入手。

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

  1. MarkdownIt 支持传入两个参数 presetName 和 options