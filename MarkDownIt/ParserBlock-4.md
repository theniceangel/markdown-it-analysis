# ParserBlock

我们在上一篇 ParserCore 讲到了，首先先调用 normalize 这个 rule 函数将 Linux(`"\n"`) 与 Windows(`"\r\n"`) 的换行符统一处理成 `"\n"`。接着就走到 ParseBlock.parse 的流程。这一步主要是产出 block 为 true 的 token。如下图所示：

```js
module.exports = function block(state) {
  var token;

  if (state.inlineMode) {
    token          = new state.Token('inline', '', 0);
    token.content  = state.src;
    token.map      = [ 0, 1 ];
    token.children = [];
    state.tokens.push(token);
  } else {
    state.md.block.parse(state.src, state.md, state.env, state.tokens);
  }
};
```

parse 函数传入了四个参数：
1. state.src 代表用户传入的字符串
2. state.md 是指当前 md 实例，主要是为了方便拿到 md 上的属性与方法
3. state.env 是在调用 md.parse 注入的一些额外的数据，默认是 `{}`，一般不会需要它，除非你要做一些定制化的开发
4. tokens 引用。**注意**：不能在 rule 函数里面更改 tokens 引用，必须保证所有的 rule 函数都是在操作统一份 tokens。

我们再来聚焦 ParserBlock 内部的逻辑。位于 `lib/parser_block.js`。

```js
var _rules = [
  [ 'table',      require('./rules_block/table'),      [ 'paragraph', 'reference' ] ],
  [ 'code',       require('./rules_block/code') ],
  [ 'fence',      require('./rules_block/fence'),      [ 'paragraph', 'reference', 'blockquote', 'list' ] ],
  [ 'blockquote', require('./rules_block/blockquote'), [ 'paragraph', 'reference', 'blockquote', 'list' ] ],
  [ 'hr',         require('./rules_block/hr'),         [ 'paragraph', 'reference', 'blockquote', 'list' ] ],
  [ 'list',       require('./rules_block/list'),       [ 'paragraph', 'reference', 'blockquote' ] ],
  [ 'reference',  require('./rules_block/reference') ],
  [ 'heading',    require('./rules_block/heading'),    [ 'paragraph', 'reference', 'blockquote' ] ],
  [ 'lheading',   require('./rules_block/lheading') ],
  [ 'html_block', require('./rules_block/html_block'), [ 'paragraph', 'reference', 'blockquote' ] ],
  [ 'paragraph',  require('./rules_block/paragraph') ]
];

function ParserBlock() {

  this.ruler = new Ruler();

  for (var i = 0; i < _rules.length; i++) {
    this.ruler.push(_rules[i][0], _rules[i][1], { alt: (_rules[i][2] || []).slice() });
  }
}

ParserBlock.prototype.tokenize = function (state, startLine, endLine) {
  var ok, i,
      rules = this.ruler.getRules(''),
      len = rules.length,
      line = startLine,
      hasEmptyLines = false,
      maxNesting = state.md.options.maxNesting;

  while (line < endLine) {
    state.line = line = state.skipEmptyLines(line);
    if (line >= endLine) { break; }

    if (state.sCount[line] < state.blkIndent) { break; }

    if (state.level >= maxNesting) {
      state.line = endLine;
      break;
    }
    for (i = 0; i < len; i++) {
      ok = rules[i](state, line, endLine, false);
      if (ok) { break; }
    }
    state.tight = !hasEmptyLines;
    if (state.isEmpty(state.line - 1)) {
      hasEmptyLines = true;
    }

    line = state.line;

    if (line < endLine && state.isEmpty(line)) {
      hasEmptyLines = true;
      line++;
      state.line = line;
    }
  }
};

ParserBlock.prototype.parse = function (src, md, env, outTokens) {
  var state;

  if (!src) { return; }

  state = new this.State(src, md, env, outTokens);

  this.tokenize(state, state.line, state.lineMax);
};


ParserBlock.prototype.State = require('./rules_block/state_block');
```
从构造函数可以看出，ParserBlock 有 11 种 rule，分别为 `table`、`code`、`fence`、`blockquote`、`hr`、`list`、`reference`、`heading`、`lheading`、`html_block`、`paragraph`。经过由这些 rule 组成的 rules chain 之后，就能输出 type 为对应类型的 tokens，这也就是 ParserBlock 的作用所在。ruler 用来管理所有的 rule 以及 rule 所属于的 chain。

```js
for (var i = 0; i < _rules.length; i++) {
  this.ruler.push(_rules[i][0], _rules[i][1], { alt: (_rules[i][2] || []).slice() });
}
```

_rules 是一个二维数组，它的元素也是一个数组，先暂称之为 ruleConfig。ruleConfig 的第一个元素是 rule 的 name。第二个是 rule 的 fn，第三个是 rule 的 alt，也就是所属的职责链。假如 alt 为 `['paragraph', 'reference']`，那么如果调用 `ruler.getRules('paragraph')` 就能返回 `[fn]`，同时调用 `ruler.getRules('reference')` 也能返回 `[fn]`，因为它属于这两种职责链。

再来看 parse 方法。

```js
ParserBlock.prototype.parse = function (src, md, env, outTokens) {
  var state;

  if (!src) { return; }

  state = new this.State(src, md, env, outTokens);

  this.tokenize(state, state.line, state.lineMax);
};
ParserBlock.prototype.State = require('./rules_block/state_block');
```

先来看属于 ParserBlock 的 State，记得之前 ParserCore 的 State么？也就是在每一个 Parser 的过程中都有一个 State 实例，用来管理他们在 parse 的一些状态。ParserBlock 的 State 是位于 `lib/rules_block/state_block.js`。

```js
function StateBlock(src, md, env, tokens) {
  var ch, s, start, pos, len, indent, offset, indent_found;
  this.src = src;
  this.md     = md;

  this.env = env;

  this.tokens = tokens

  this.bMarks = []
  this.eMarks = []
  this.tShift = []
  this.sCount = []

  this.bsCount = []

  this.blkIndent  = 0

  this.line       = 0
  this.lineMax    = 0
  this.tight      = false
  this.ddIndent   = -1
  this.parentType = 'root'

  this.level = 0

  this.result = ''
  s = this.src
  indent_found = false

  for (start = pos = indent = offset = 0, len = s.length; pos < len; pos++) {
    ch = s.charCodeAt(pos);

    if (!indent_found) {
      if (isSpace(ch)) {
        indent++;

        if (ch === 0x09) {
          offset += 4 - offset % 4;
        } else {
          offset++;
        }
        continue;
      } else {
        indent_found = true;
      }
    }

    if (ch === 0x0A || pos === len - 1) {
      if (ch !== 0x0A) { pos++; }
      this.bMarks.push(start);
      this.eMarks.push(pos);
      this.tShift.push(indent);
      this.sCount.push(offset);
      this.bsCount.push(0);

      indent_found = false;
      indent = 0;
      offset = 0;
      start = pos + 1;
    }
  }

  this.bMarks.push(s.length);
  this.eMarks.push(s.length);
  this.tShift.push(0);
  this.sCount.push(0);
  this.bsCount.push(0);

  this.lineMax = this.bMarks.length - 1;
}
```

理解 State 上的属性的作用，是很关键的。因为这些属性都是接下来 tokenize 所依赖的信息。重点关注如下的属性：

- **tokens**

  tokenize 之后的 token 组成的数组

- **bMarks**

  存储每一行的起始位置，因为 parse 的过程是根据换行符逐行扫描

- **eMarks**

  存储每一行的终止位置

- **tShift**

  存储每一行第一个非空格的字符的位置（制表符长度只算做1）

- **sCount**

  存储每一行第一个非空格的字符串的位置（制表符长度为4）

- **bsCount**

  一般为 0

- **blkIndent**

  一般为 0

- **line**

  当前所在行数。tokenize 的时候逐行扫描会用到

- **lineMax**

  src 被分割成了多少行

以上都是在 tokenize 过程中非常有用的属性。接来下看一下 tokenize 的过程，之后就生成了 block 为 true 的 token。

```js
ParserBlock.prototype.tokenize = function (state, startLine, endLine) {
  var ok, i,
      rules = this.ruler.getRules(''),
      len = rules.length,
      line = startLine,
      hasEmptyLines = false,
      maxNesting = state.md.options.maxNesting;

  while (line < endLine) {
    state.line = line = state.skipEmptyLines(line);
    if (line >= endLine) { break; }

    if (state.sCount[line] < state.blkIndent) { break; }

    if (state.level >= maxNesting) {
      state.line = endLine;
      break;
    }

    for (i = 0; i < len; i++) {
      ok = rules[i](state, line, endLine, false);
      if (ok) { break; }
    }

    state.tight = !hasEmptyLines;
    if (state.isEmpty(state.line - 1)) {
      hasEmptyLines = true;
    }

    line = state.line;

    if (line < endLine && state.isEmpty(line)) {
      hasEmptyLines = true;
      line++;
      state.line = line;
    }
  }
}
```
函数的执行流程如下：

1. 获取 ParserBlock 构造函数声明的所有 rule 函数，因为在 Ruler 类里面规定，内部的 rule 函数一定属于名字为空字符串的rule chain。当然构造函数还有很多其他的 rule chain。比如 `paragraph`、`reference`、`blockquote`、 `list`，暂时还未用到。同时，声明了很多初始变量。

2. 然后走到一个 while 循环，因为 state_block 存放的信息都是以 src 每一行作为维度区分的，比如每一行的起始位置，每一行的终止位置，每一行第一个字符的位置。这些信息都是特定 rule 所需要的。while 语句的前面部分就是跳过空行、是否达到最大嵌套等级的判断，重点关注这行代码。

```js
for (i = 0; i < len; i++) {
  ok = rules[i](state, line, endLine, false);
  if (ok) { break; }
}
```

这里的循环，也就是会对 src 的每一行都执行 rule chain，进而产出 token，如果其中一个 rule 返回 true，就跳出循环，准备 tokenize src 的下一行。
那我们来看下这些 rules 的作用。它们都位于 `lib/rules_block` 文件夹下面。

- **table.js**

```js
module.exports = function table(state, startLine, endLine, silent) {
  var ch, lineText, pos, i, nextLine, columns, columnCount, token,
      aligns, t, tableLines, tbodyLines;

if (startLine + 2 > endLine) { return false; }

  nextLine = startLine + 1;

  if (state.sCount[nextLine] < state.blkIndent) { return false; }

  if (state.sCount[nextLine] - state.blkIndent >= 4) { return false; }

  pos = state.bMarks[nextLine] + state.tShift[nextLine];
  if (pos >= state.eMarks[nextLine]) { return false; }

  ch = state.src.charCodeAt(pos++);
  if (ch !== 0x7C/* | */ && ch !== 0x2D/* - */ && ch !== 0x3A/* : */) { return false; }

  while (pos < state.eMarks[nextLine]) {
    ch = state.src.charCodeAt(pos);

    if (ch !== 0x7C/* | */ && ch !== 0x2D/* - */ && ch !== 0x3A/* : */ && !isSpace(ch)) { return false; }

    pos++;
  }

  lineText = getLine(state, startLine + 1);

  columns = lineText.split('|');
  aligns = [];
  for (i = 0; i < columns.length; i++) {
    t = columns[i].trim();
    if (!t) {
      if (i === 0 || i === columns.length - 1) {
        continue;
      } else {
        return false;
      }
    }

    if (!/^:?-+:?$/.test(t)) { return false; }
    if (t.charCodeAt(t.length - 1) === 0x3A/* : */) {
      aligns.push(t.charCodeAt(0) === 0x3A/* : */ ? 'center' : 'right');
    } else if (t.charCodeAt(0) === 0x3A/* : */) {
      aligns.push('left');
    } else {
      aligns.push('');
    }
  }

  lineText = getLine(state, startLine).trim();
  if (lineText.indexOf('|') === -1) { return false; }
  if (state.sCount[startLine] - state.blkIndent >= 4) { return false; }
  columns = escapedSplit(lineText.replace(/^\||\|$/g, ''));

  columnCount = columns.length;
  if (columnCount > aligns.length) { return false; }

  if (silent) { return true; }

  token     = state.push('table_open', 'table', 1);
  token.map = tableLines = [ startLine, 0 ];

  token     = state.push('thead_open', 'thead', 1);
  token.map = [ startLine, startLine + 1 ];

  token     = state.push('tr_open', 'tr', 1);
  token.map = [ startLine, startLine + 1 ];

  for (i = 0; i < columns.length; i++) {
    token          = state.push('th_open', 'th', 1);
    token.map      = [ startLine, startLine + 1 ];
    if (aligns[i]) {
      token.attrs  = [ [ 'style', 'text-align:' + aligns[i] ] ];
    }

    token          = state.push('inline', '', 0);
    token.content  = columns[i].trim();
    token.map      = [ startLine, startLine + 1 ];
    token.children = [];

    token          = state.push('th_close', 'th', -1);
  }

  token     = state.push('tr_close', 'tr', -1);
  token     = state.push('thead_close', 'thead', -1);

  token     = state.push('tbody_open', 'tbody', 1);
  token.map = tbodyLines = [ startLine + 2, 0 ];

  for (nextLine = startLine + 2; nextLine < endLine; nextLine++) {
    if (state.sCount[nextLine] < state.blkIndent) { break; }

    lineText = getLine(state, nextLine).trim();
    if (lineText.indexOf('|') === -1) { break; }
    if (state.sCount[nextLine] - state.blkIndent >= 4) { break; }
    columns = escapedSplit(lineText.replace(/^\||\|$/g, ''));

    token = state.push('tr_open', 'tr', 1);
    for (i = 0; i < columnCount; i++) {
      token          = state.push('td_open', 'td', 1);
      if (aligns[i]) {
        token.attrs  = [ [ 'style', 'text-align:' + aligns[i] ] ];
      }

      token          = state.push('inline', '', 0);
      token.content  = columns[i] ? columns[i].trim() : '';
      token.children = [];

      token          = state.push('td_close', 'td', -1);
    }
    token = state.push('tr_close', 'tr', -1);
  }
  token = state.push('tbody_close', 'tbody', -1);
  token = state.push('table_close', 'table', -1);

  tableLines[1] = tbodyLines[1] = nextLine;
  state.line = nextLine;
  return true;
}
```

table 这个 rule 就是用来生成 table HMTL 字符串的。内部的解析都是根据 markdown 写 table 的规范而来的，详细逻辑这里就不展开，如果感兴趣，可以写个 demo 自己打断点试试。

- **code.js**

```js
module.exports = function code(state, startLine, endLine/*, silent*/) {
  var nextLine, last, token;

  if (state.sCount[startLine] - state.blkIndent < 4) { return false; }

  last = nextLine = startLine + 1;

  while (nextLine < endLine) {
    if (state.isEmpty(nextLine)) {
      nextLine++;
      continue;
    }

    if (state.sCount[nextLine] - state.blkIndent >= 4) {
      nextLine++;
      last = nextLine;
      continue;
    }
    break;
  }

  state.line = last;

  token         = state.push('code_block', 'code', 0);
  token.content = state.getLines(startLine, last, 4 + state.blkIndent, true);
  token.map     = [ startLine, state.line ];

  return true;
};
```
code rule 的作用也是很简单，它认为只要你每行的起始位置多于 3 个空格，那就是一个 code_block。比如下面的。

    我现在就是一个 code_block

- **fence.js**

```js
module.exports = function fence(state, startLine, endLine, silent) {
  var marker, len, params, nextLine, mem, token, markup,
      haveEndMarker = false,
      pos = state.bMarks[startLine] + state.tShift[startLine],
      max = state.eMarks[startLine];

  if (state.sCount[startLine] - state.blkIndent >= 4) { return false; }

  if (pos + 3 > max) { return false; }

  marker = state.src.charCodeAt(pos);

  if (marker !== 0x7E/* ~ */ && marker !== 0x60 /* ` */) {
    return false;
  }

  mem = pos;
  pos = state.skipChars(pos, marker);

  len = pos - mem;

  if (len < 3) { return false; }

  markup = state.src.slice(mem, pos);
  params = state.src.slice(pos, max);

  if (params.indexOf(String.fromCharCode(marker)) >= 0) { return false; }

  // Since start is found, we can report success here in validation mode
  if (silent) { return true; }

  // search end of block
  nextLine = startLine;

  for (;;) {
    nextLine++;
    if (nextLine >= endLine) {
      break;
    }

    pos = mem = state.bMarks[nextLine] + state.tShift[nextLine];
    max = state.eMarks[nextLine];

    if (pos < max && state.sCount[nextLine] < state.blkIndent) {
      break;
    }

    if (state.src.charCodeAt(pos) !== marker) { continue; }

    if (state.sCount[nextLine] - state.blkIndent >= 4) {
      continue;
    }

    pos = state.skipChars(pos, marker);

    if (pos - mem < len) { continue; }

    pos = state.skipSpaces(pos);

    if (pos < max) { continue; }

    haveEndMarker = true;
    // found!
    break;
  }

  len = state.sCount[startLine];

  state.line = nextLine + (haveEndMarker ? 1 : 0);

  token         = state.push('fence', 'code', 0);
  token.info    = params;
  token.content = state.getLines(startLine + 1, nextLine, len, true);
  token.markup  = markup;
  token.map     = [ startLine, state.line ];

  return true;
};

```

fence rule 类似于 code rule。它代表具有语言类型的 code_block。比如 javascript、shell、css、stylus等等。举个栗子：

```shell
echo 'done'
```

上面就是会解析生成一个 type 为 fence，info 为 shell，markup 为 `"```"` 的token。

- **blockquote.js**

代码太长，就不贴代码了，blockquote 的作用就是生成 markup 为 > 的 token。下面就是一个 blockquote。

> i am a blockquote

- **hr.js**

```js
module.exports = function hr(state, startLine, endLine, silent) {
  var marker, cnt, ch, token,
      pos = state.bMarks[startLine] + state.tShift[startLine],
      max = state.eMarks[startLine];

  // if it's indented more than 3 spaces, it should be a code block
  if (state.sCount[startLine] - state.blkIndent >= 4) { return false; }

  marker = state.src.charCodeAt(pos++);

  // Check hr marker
  if (marker !== 0x2A/* * */ &&
      marker !== 0x2D/* - */ &&
      marker !== 0x5F/* _ */) {
    return false;
  }

  // markers can be mixed with spaces, but there should be at least 3 of them

  cnt = 1;
  while (pos < max) {
    ch = state.src.charCodeAt(pos++);
    if (ch !== marker && !isSpace(ch)) { return false; }
    if (ch === marker) { cnt++; }
  }

  if (cnt < 3) { return false; }

  if (silent) { return true; }

  state.line = startLine + 1;

  token        = state.push('hr', 'hr', 0);
  token.map    = [ startLine, state.line ];
  token.markup = Array(cnt + 1).join(String.fromCharCode(marker));

  return true;
};
```

hr rule 也很简单，就是生成 type 为 hr 的 token。它的 markup 是 `***`、`---`、`___`，也就是在 md 文件写这三种语法，都能解析出 `<hr>` 标签。

- **list.js**

list 作用是为了解析有序列表以及无序列表的。详细的逻辑比较复杂，需要了解的可以自己通过 demo 断点调试。

- **reference.js**

reference 作用是为了解析超链接。我们在 md 的语法就是类似于 `[reference](http://www.baidu.con)` 这种。

- **heading.js**

```js
module.exports = function heading(state, startLine, endLine, silent) {
  var ch, level, tmp, token,
      pos = state.bMarks[startLine] + state.tShift[startLine],
      max = state.eMarks[startLine];

  // if it's indented more than 3 spaces, it should be a code block
  if (state.sCount[startLine] - state.blkIndent >= 4) { return false; }

  ch  = state.src.charCodeAt(pos);

  if (ch !== 0x23/* # */ || pos >= max) { return false; }

  // count heading level
  level = 1;
  ch = state.src.charCodeAt(++pos);
  while (ch === 0x23/* # */ && pos < max && level <= 6) {
    level++;
    ch = state.src.charCodeAt(++pos);
  }

  if (level > 6 || (pos < max && !isSpace(ch))) { return false; }

  if (silent) { return true; }

  // Let's cut tails like '    ###  ' from the end of string

  max = state.skipSpacesBack(max, pos);
  tmp = state.skipCharsBack(max, 0x23, pos); // #
  if (tmp > pos && isSpace(state.src.charCodeAt(tmp - 1))) {
    max = tmp;
  }

  state.line = startLine + 1;

  token        = state.push('heading_open', 'h' + String(level), 1);
  token.markup = '########'.slice(0, level);
  token.map    = [ startLine, state.line ];

  token          = state.push('inline', '', 0);
  token.content  = state.src.slice(pos, max).trim();
  token.map      = [ startLine, state.line ];
  token.children = [];

  token        = state.push('heading_close', 'h' + String(level), -1);
  token.markup = '########'.slice(0, level);

  return true;
};
```

heading 作用是解析标题标签(h1 - h6)。它的语法主要是 #, ## 等等。

- **lheading.js**

lheading 是解析自带分隔符的标签，比如下面

```markup

这是一个标题
========

// 上面会渲染成

<h1>这是一个标题</h1>
```

- **html_block.js**

html_block 是解析 HTML，如果你在 md 里面写 HTML 标签，那么最后还是会得到 HTML 字符串，比如你写如下字符串：

```js
let src = "<p>234</p>"

// 得到如下token

let token = [
  {
    "type": "html_block",
    "tag": "",
    "attrs": null,
    "map": [
      0,
      1
    ],
    "nesting": 0,
    "level": 0,
    "children": null,
    "content": "<p>234</p>",
    "markup": "",
    "info": "",
    "meta": null,
    "block": true,
    "hidden": false
  }
]

最后输出的字符串也是 `<p>234</p>`
```

- **paragraph.js**

```js
module.exports = function paragraph(state, startLine/*, endLine*/) {
  var content, terminate, i, l, token, oldParentType,
      nextLine = startLine + 1,
      terminatorRules = state.md.block.ruler.getRules('paragraph'),
      endLine = state.lineMax;

  oldParentType = state.parentType;
  state.parentType = 'paragraph';

  // jump line-by-line until empty one or EOF
  for (; nextLine < endLine && !state.isEmpty(nextLine); nextLine++) {
    // this would be a code block normally, but after paragraph
    // it's considered a lazy continuation regardless of what's there
    if (state.sCount[nextLine] - state.blkIndent > 3) { continue; }

    // quirk for blockquotes, this line should already be checked by that rule
    if (state.sCount[nextLine] < 0) { continue; }

    // Some tags can terminate paragraph without empty line.
    terminate = false;
    for (i = 0, l = terminatorRules.length; i < l; i++) {
      if (terminatorRules[i](state, nextLine, endLine, true)) {
        terminate = true;
        break;
      }
    }
    if (terminate) { break; }
  }

  content = state.getLines(startLine, nextLine, state.blkIndent, false).trim();

  state.line = nextLine;

  token          = state.push('paragraph_open', 'p', 1);
  token.map      = [ startLine, state.line ];

  token          = state.push('inline', '', 0);
  token.content  = content;
  token.map      = [ startLine, state.line ];
  token.children = [];

  token          = state.push('paragraph_close', 'p', -1);

  state.parentType = oldParentType;

  return true;
};
```

paragraph 那就很简单也是经常用到的，就是生成 p 标签。

## 总结

综上，可以看出 ParserBlock 的流程还是非常的复杂与繁琐的。首先它拥有自己的 block_state，block_state 存储了 ParserBlock 在 tokenize 过程中需要的很多信息，它将 src 字符串按照换行符分割成了以行作为维度的字符串。在 tokenize 的过程中逐行对字符串运用不同的 rule 函数，生成对应类型的 token，这样就完成了 ParserBlock 的 parse 过程。

![ParseerBlock](https://github.com/theniceangel/markdown-it-analysis/blob/master/images/parser-block.png?raw=true)

在 ParserBlock 处理之后，生成了一种 type 为 inline 的 token。这种 token 属于未完全解析的 token，因为它有一个 children 属性，会存放通过 ParserInline 处理之后的更细粒度的 token。下一张，我们进一步探讨 ParserInline 的整体流程。