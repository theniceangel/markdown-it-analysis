# markdown-it-emoji

插件扩展了在 md 文件里面识别 emoji 的能力。一般 emoji 的语法是 `:名称:`。名称一般是指定的英文或者数字，同时还支持一些 shortcuts。例如

```js
:100: => 💯
:stuck_out_tongue: => 😛
// shortcuts
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

    渲染 rule，是在 MarkdownIt.renderer.render 之后调用的。也就是所有的 Parser 生成不同 type 的 token 之后，开始渲染输出的。正如上面 `emoji_html` 函数一样简单，就是返回 token 的 content 就行。content 这个时候已经是 emoji 了。

  4. **ParserCore 添加 emoji rule**

      ```js
      module.exports = function create_rule(md, emojies, shortcuts, scanRE, replaceRE) {
        var arrayReplaceAt = md.utils.arrayReplaceAt,
            ucm = md.utils.lib.ucmicro,
            ZPCc = new RegExp([ ucm.Z.source, ucm.P.source, ucm.Cc.source ].join('|'));

        function splitTextToken(text, level, Token) {
          ......
        }

        return function emoji_replace(state) {
          var i, j, l, tokens, token,
              blockTokens = state.tokens,
              autolinkLevel = 0;

          for (j = 0, l = blockTokens.length; j < l; j++) {
            if (blockTokens[j].type !== 'inline') { continue; }
            tokens = blockTokens[j].children;

            // We scan from the end, to keep position when new tags added.
            // Use reversed logic in links start/end match
            for (i = tokens.length - 1; i >= 0; i--) {
              token = tokens[i];

              if (token.type === 'link_open' || token.type === 'link_close') {
                if (token.info === 'auto') { autolinkLevel -= token.nesting; }
              }

              if (token.type === 'text' && autolinkLevel === 0 && scanRE.test(token.content)) {
                // replace current node
                blockTokens[j].children = tokens = arrayReplaceAt(
                  tokens, i, splitTextToken(token.content, token.level, state.Token)
                );
              }
            }
          }
        };
      };
      ```

      步骤 4 的执行时间是发生在步骤 3 之前的，因为步骤 4 的 rule 是在 ParserCore.parse 的时候调用，而步骤 3 是在 render 的过程中调用。

      emoji_replace 的逻辑很清晰，给 ParserCore 的 ParserBlock 处理完成之后，这个时候，会生成 type 为 inline 的 token。而 emoji_replace 函数先过滤出 type 为 inline 的 token。再拿到 `token.children`，从后往前扫描存在 children 里的 token。如果命中了以下逻辑，就开始准备生成 type 为 emoji 的 token并且调用 `arrayReplaceAt` 插入到 token.children 当中，最后再经过步骤 3 的 `md.renderer.rules.emoji` 处理，生成对应的 emoji。

      ```js
      if (token.type === 'text' && autolinkLevel === 0 && scanRE.test(token.content)) {
        // replace current node
        blockTokens[j].children = tokens = arrayReplaceAt(
          tokens, i, splitTextToken(token.content, token.level, state.Token)
        );
      }
      ```

      我们再来看下 `splitTextToken` 是怎么处理 `token.content`，最终生成 type 为 emoji 的 token的。

      ```js
      function splitTextToken(text, level, Token) {
        var token, last_pos = 0, nodes = [];

        text.replace(replaceRE, function (match, offset, src) {
          var emoji_name;
          if (shortcuts.hasOwnProperty(match)) {
            emoji_name = shortcuts[match];

            if (offset > 0 && !ZPCc.test(src[offset - 1])) {
              return;
            }

            if (offset + match.length < src.length && !ZPCc.test(src[offset + match.length])) {
              return;
            }
          } else {
            emoji_name = match.slice(1, -1);
          }

          if (offset > last_pos) {
            token         = new Token('text', '', 0);
            token.content = text.slice(last_pos, offset);
            nodes.push(token);
          }

          token         = new Token('emoji', '', 0);
          token.markup  = emoji_name;
          token.content = emojies[emoji_name];
          nodes.push(token);

          last_pos = offset + match.length;
        });

        if (last_pos < text.length) {
          token         = new Token('text', '', 0);
          token.content = text.slice(last_pos);
          nodes.push(token);
        }

        return nodes;
      }
      ```

      第一个参数 `text`，就是 `token.content`。它是一个含有 emoji 语法，但还未生成 emoji 的字符串，比如 `":smile:"`，接着调用 `text.replace` 函数，并且传入 `replaceRE` 这个正则，`replaceRE` 是拥有解析 `lib/data/full.json` 以及 `lib/data/shortcuts.js` emoji 语法的正则，它是一个全局匹配模式，会逐步的将 `text` 内符合对应 emoji 语法的字符串转化为 emoji。举个例子：

      ```js
      const text = ":D,:100:,:-1:"

      // 经过 splitTextToken 处理，最后输出 😄,💯,👎
      ```

## 小结

经过 markdown-it-emoji 的插件的处理之后，最后 md 文件里面的 `emoji` 语法，都将被识别并且渲染成 `emoji`。

从这个插件来看，MarkdownIt 的扩展性是非常优秀的。你总是能在不同的阶段去触及到 tokens，甚至还可以更改 render rule 来定制化自己的需求。