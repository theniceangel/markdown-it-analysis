# ParserInline

我们在 ParserCore 讲到了，经过 ParserCore 处理之后，生成了 type 为 inline 的 token。下一步就是交给 ParserInline 处理。而这个 rule 函数的代码如下：

```js
module.exports = function inline(state) {
  var tokens = state.tokens, tok, i, l;

  // Parse inlines
  for (i = 0, l = tokens.length; i < l; i++) {
    tok = tokens[i];
    if (tok.type === 'inline') {
      state.md.inline.parse(tok.content, state.md, state.env, tok.children);
    }
  }
};
```

也就是拿到 type 为 inline 的 token，调用 ParserInline 的 parse 方法。ParserInline 位于 `lib/parser_inline.js`。

```js
var _rules = [
  [ 'text',            require('./rules_inline/text') ],
  [ 'newline',         require('./rules_inline/newline') ],
  [ 'escape',          require('./rules_inline/escape') ],
  [ 'backticks',       require('./rules_inline/backticks') ],
  [ 'strikethrough',   require('./rules_inline/strikethrough').tokenize ],
  [ 'emphasis',        require('./rules_inline/emphasis').tokenize ],
  [ 'link',            require('./rules_inline/link') ],
  [ 'image',           require('./rules_inline/image') ],
  [ 'autolink',        require('./rules_inline/autolink') ],
  [ 'html_inline',     require('./rules_inline/html_inline') ],
  [ 'entity',          require('./rules_inline/entity') ]
];

var _rules2 = [
  [ 'balance_pairs',   require('./rules_inline/balance_pairs') ],
  [ 'strikethrough',   require('./rules_inline/strikethrough').postProcess ],
  [ 'emphasis',        require('./rules_inline/emphasis').postProcess ],
  [ 'text_collapse',   require('./rules_inline/text_collapse') ]
];

function ParserInline() {
  var i;

  this.ruler = new Ruler();

  for (i = 0; i < _rules.length; i++) {
    this.ruler.push(_rules[i][0], _rules[i][1]);
  }

  this.ruler2 = new Ruler();

  for (i = 0; i < _rules2.length; i++) {
    this.ruler2.push(_rules2[i][0], _rules2[i][1]);
  }
}
```

从构造函数看出，ParserInline 不同于 ParserBlock，它是有两个 Ruler 实例的。他们都是在 parse 方法里面用到的， ruler 是在 tokenize 调用的，ruler2 是在 tokenize 之后再使用的。

```js
ParserInline.prototype.tokenize = function (state) {
  var ok, i,
      rules = this.ruler.getRules(''),
      len = rules.length,
      end = state.posMax,
      maxNesting = state.md.options.maxNesting;

  while (state.pos < end) {
    if (state.level < maxNesting) {
      for (i = 0; i < len; i++) {
        ok = rules[i](state, false);
        if (ok) { break; }
      }
    }

    if (ok) {
      if (state.pos >= end) { break; }
      continue;
    }

    state.pending += state.src[state.pos++];
  }

  if (state.pending) {
    state.pushPending();
  }
};

ParserInline.prototype.parse = function (str, md, env, outTokens) {
  var i, rules, len;
  var state = new this.State(str, md, env, outTokens);

  this.tokenize(state);

  rules = this.ruler2.getRules('');
  len = rules.length;

  for (i = 0; i < len; i++) {
    rules[i](state);
  }
};
```

文章的开头说到将 type 为 inline 的 token 传给 md.inline.parse 方法，这样就走进了 parse 的函数内部，首先生成属于 ParserInline 的 state，还记得 ParserCore 与 ParserBlock 的 state 么？它们的作用都是存放不同 parser 在 parse 过程中的依赖信息。

我们先来看下 State 类，它位于 `lib/rules_inline/state_inline.js`。

```js
function StateInline(src, md, env, outTokens) {
  this.src = src;
  this.env = env;
  this.md = md;
  this.tokens = outTokens;

  this.pos = 0;
  this.posMax = this.src.length;
  this.level = 0;
  this.pending = '';
  this.pendingLevel = 0;

  this.cache = {};

  this.delimiters = [];
}
```

列举一些比较有用的字段信息：

1. **pos**

当前 token 的 content 的第几个字符串索引

2. **posMax**

当前 token 的 content 的最大索引

3. **pending**

存放一段完整的字符串，比如

```js
let src = "**emphasis**"
let state = new StateInline(src)

// state.pending 就是 'emphasis'
```

4. **delimiters**

存放一些特殊标记的分隔符，比如 `*`、`~` 等。元素格式如下:

```js
{
  close:false
  end:-1
  jump:0
  length:2
  level:0
  marker:42
  open:true
  token:0
}
// marker 表示字符串对应的 ascii 码
```

生成 state 之后，然后调用 tokenize 方法。

```js
ParserInline.prototype.tokenize = function (state) {
  var ok, i,
      rules = this.ruler.getRules(''),
      len = rules.length,
      end = state.posMax,
      maxNesting = state.md.options.maxNesting;

  while (state.pos < end) {
    if (state.level < maxNesting) {
      for (i = 0; i < len; i++) {
        ok = rules[i](state, false);
        if (ok) { break; }
      }
    }

    if (ok) {
      if (state.pos >= end) { break; }
      continue;
    }

    state.pending += state.src[state.pos++];
  }

  if (state.pending) {
    state.pushPending();
  }
};
```

首先获取默认的 rule chain，然后扫描 token 的 content 字段，从第一个字符扫描至尾部，每一个字符依次调用 ruler 的 rule 函数。它们位于 `lib/rules_inline/` 文件夹下面。调用顺序依次如下：

- **text.js**

  ```js
  module.exports = function text(state, silent) {
    var pos = state.pos;

    while (pos < state.posMax && !isTerminatorChar(state.src.charCodeAt(pos))) {
      pos++;
    }

    if (pos === state.pos) { return false; }

    if (!silent) { state.pending += state.src.slice(state.pos, pos); }

    state.pos = pos;

    return true;
  };
  ```

  作用是提取连续的非 isTerminatorChar 字符。isTerminatorChar 字符的规定如下：

  ```js
  function isTerminatorChar(ch) {
    switch (ch) {
      case 0x0A/* \n */:
      case 0x21/* ! */:
      case 0x23/* # */:
      case 0x24/* $ */:
      case 0x25/* % */:
      case 0x26/* & */:
      case 0x2A/* * */:
      case 0x2B/* + */:
      case 0x2D/* - */:
      case 0x3A/* : */:
      case 0x3C/* < */:
      case 0x3D/* = */:
      case 0x3E/* > */:
      case 0x40/* @ */:
      case 0x5B/* [ */:
      case 0x5C/* \ */:
      case 0x5D/* ] */:
      case 0x5E/* ^ */:
      case 0x5F/* _ */:
      case 0x60/* ` */:
      case 0x7B/* { */:
      case 0x7D/* } */:
      case 0x7E/* ~ */:
        return true;
      default:
        return false;
    }
  }
  ```

  假如输入是 "\_\_ad\_\_"，那么这个 rule 就能提取 "ad" 字符串出来。

- **newline.js**

  ```js
  module.exports = function newline(state, silent) {
    var pmax, max, pos = state.pos;

    if (state.src.charCodeAt(pos) !== 0x0A/* \n */) { return false; }

    pmax = state.pending.length - 1;
    max = state.posMax;

    if (!silent) {
      if (pmax >= 0 && state.pending.charCodeAt(pmax) === 0x20) {
        if (pmax >= 1 && state.pending.charCodeAt(pmax - 1) === 0x20) {
          state.pending = state.pending.replace(/ +$/, '');
          state.push('hardbreak', 'br', 0);
        } else {
          state.pending = state.pending.slice(0, -1);
          state.push('softbreak', 'br', 0);
        }

      } else {
        state.push('softbreak', 'br', 0);
      }
    }

    pos++;

    while (pos < max && isSpace(state.src.charCodeAt(pos))) { pos++; }

    state.pos = pos;
    return true;
  };
  ```

  处理换行符（`\n`）。

- escape.js

  ```js
  module.exports = function escape(state, silent) {
    var ch, pos = state.pos, max = state.posMax;

    if (state.src.charCodeAt(pos) !== 0x5C/* \ */) { return false; }

    pos++;

    if (pos < max) {
      ch = state.src.charCodeAt(pos);

      if (ch < 256 && ESCAPED[ch] !== 0) {
        if (!silent) { state.pending += state.src[pos]; }
        state.pos += 2;
        return true;
      }

      if (ch === 0x0A) {
        if (!silent) {
          state.push('hardbreak', 'br', 0);
        }

        pos++;
        // skip leading whitespaces from next line
        while (pos < max) {
          ch = state.src.charCodeAt(pos);
          if (!isSpace(ch)) { break; }
          pos++;
        }

        state.pos = pos;
        return true;
      }
    }

    if (!silent) { state.pending += '\\'; }
    state.pos++;
    return true;
  };
  ```

  处理转义字符（`\`）。


- **backtick.js**

  ```js
  module.exports = function backtick(state, silent) {
    var start, max, marker, matchStart, matchEnd, token,
        pos = state.pos,
        ch = state.src.charCodeAt(pos);

    if (ch !== 0x60/* ` */) { return false; }

    start = pos;
    pos++;
    max = state.posMax;

    while (pos < max && state.src.charCodeAt(pos) === 0x60/* ` */) { pos++; }

    marker = state.src.slice(start, pos);

    matchStart = matchEnd = pos;

    while ((matchStart = state.src.indexOf('`', matchEnd)) !== -1) {
      matchEnd = matchStart + 1;

      while (matchEnd < max && state.src.charCodeAt(matchEnd) === 0x60/* ` */) { matchEnd++; }

      if (matchEnd - matchStart === marker.length) {
        if (!silent) {
          token         = state.push('code_inline', 'code', 0);
          token.markup  = marker;
          token.content = state.src.slice(pos, matchStart)
                                  .replace(/[ \n]+/g, ' ')
                                  .trim();
        }
        state.pos = matchEnd;
        return true;
      }
    }

    if (!silent) { state.pending += marker; }
    state.pos += marker.length;
    return true;
  };

  ```

  处理反引号字符（`）。

  markdown 语法： \`这是反引号\`。

- **strikethrough.js**

  代码太长，就不粘贴了，作用是处理删除字符（`~`）。

  markdown 语法： `~~strike~~`。

- **emphasis.js**

  作用是处理加粗文字的字符（`*` 或者 `_`）。

  markdown 语法： `**strong**`。

- **link.js**

  作用是解析超链接。

  markdown 语法： `[text](href)`。

- **image.js**

  作用是解析图片。

  markdown 语法： `![image](<src> "title")`。
