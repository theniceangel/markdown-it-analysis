# markdown-it-emoji

插件扩展了在 md 文件里面识别 emoji 的能力。一般 emoji 的语法是 `:名称:`。名称一般是指定的英文或者数字，同时还支持一些 shortcuts。例如

```js
:100: => 💯
:stuck_out_tongue: => 😛
:D => 😄
```

注册插件的逻辑如下：

```js
var md = require('markdown-it')();
var emoji = require('markdown-it-emoji');

md.use(emoji [, options]);
```

而 MarkdownIt 的 `use` 的逻辑很简单，就是调用 `use` 传入的第一个参数，它是一个函数，这函数会被调用，并且入参是从第二个参数开始的所有参数。

```js
MarkdownIt.prototype.use = function (plugin /*, params, ... */) {
  var args = [ this ].concat(Array.prototype.slice.call(arguments, 1));
  plugin.apply(plugin, args);
  return this;
};
```

而我们更加关注的是 markdown-it-emoji 的这个函数，它位于 `markdown-it-emoji/index.js`。

```js
var emojies_defs      = require('./lib/data/full.json');
var emojies_shortcuts = require('./lib/data/shortcuts');
var emoji_html        = require('./lib/render');
var emoji_replace     = require('./lib/replace');
var normalize_opts    = require('./lib/normalize_opts');


module.exports = function emoji_plugin(md, options) {

  // 步骤一
  var defaults = {
    defs: emojies_defs,
    shortcuts: emojies_shortcuts,
    enabled: []
  };

  // 步骤二
  var opts = normalize_opts(md.utils.assign({}, defaults, options || {}));

  // 步骤三
  md.renderer.rules.emoji = emoji_html;

  // 步骤四
  md.core.ruler.push('emoji', emoji_replace(md, opts.defs, opts.shortcuts, opts.scanRE, opts.replaceRE));
};
```

emoji_plugin 这个函数看起来也是非常的简单，首先它有 `md` 和 `options` 两个参数。 `options` 会与内置的 `defaults` 做一次 assign 操作。我们根据函数的执行，大致分为 4 个步骤。

1. **defaults**

```js
// defs 属性值是 emoji 的映射。

defs = {
  "100": "💯",
  "1234": "🔢",
  "grinning": "😀",
  "smiley": "😃",
  "smile": "😄",
  "grin": "😁",
  "laughing": "😆",
  ......
  // 所有的配置在 `lib/data/full.json`
}

// shortcuts 属性值是一些短名称的映射配置。
// 比如你可以用 ":smile:"，也可以用 ":D"

shortcuts = [
  angry:            [ '>:(', '>:-(' ],
  blush:            [ ':")', ':-")' ],
  broken_heart:     [ '</3', '<\\3' ],
  ......
  // 所有的配置在 `lib/data/shortcuts.js`
]

// 开启的 emoji 规则。仅仅只开启 eabled 配置的 emoji，会把其他的默认 emoji 规则关闭
enabled = []
```

2. **normalize_opts**

```js
module.exports = function normalize_opts(options) {
  var emojies = options.defs,
      shortcuts;

  // Filter emojies by whitelist, if needed
  if (options.enabled.length) {
    emojies = Object.keys(emojies).reduce(function (acc, key) {
      if (options.enabled.indexOf(key) >= 0) {
        acc[key] = emojies[key];
      }
      return acc;
    }, {});
  }

  // Flatten shortcuts to simple object: { alias: emoji_name }
  shortcuts = Object.keys(options.shortcuts).reduce(function (acc, key) {
    // Skip aliases for filtered emojies, to reduce regexp
    if (!emojies[key]) { return acc; }

    if (Array.isArray(options.shortcuts[key])) {
      options.shortcuts[key].forEach(function (alias) {
        acc[alias] = key;
      });
      return acc;
    }

    acc[options.shortcuts[key]] = key;
    return acc;
  }, {});

  // Compile regexp
  var names = Object.keys(emojies)
                .map(function (name) { return ':' + name + ':'; })
                .concat(Object.keys(shortcuts))
                .sort()
                .reverse()
                .map(function (name) { return quoteRE(name); })
                .join('|');
  var scanRE = RegExp(names);
  var replaceRE = RegExp(names, 'g');

  return {
    defs: emojies,
    shortcuts: shortcuts,
    scanRE: scanRE,
    replaceRE: replaceRE
  };
};
```

函数逻辑很简单，就是处理用户输入的 options。首先处理 `enabled` 白名单校验，然后再支持 `shortcuts` 的语法，最后生成 `scanRE` 正则，这个是用来识别 emoji 语法。它是以 `|` 为分割，并且拥有校验 `full.json` 和 `shortcuts.js` 所有的 emoji 语法的能力。

3. **添加渲染 emoji 的 rule**

  ```js

  md.renderer.rules.emoji = emoji_html;

  module.exports = function emoji_html(tokens, idx /*, options, env */) {
    return tokens[idx].content;
  };
  ```

  渲染 rule，是在 MarkdownIt.renderer.render 之后调用的。也就是所有的 Parser 生成不同 type 的 token 之后，开始做渲染输出的。正如上面 `emoji_html` 函数一样简单，就是返回 token 的 content 就行。content 这个时候已经是 emoji 了。

  4. **ParserCore 添加 emoji rule**

