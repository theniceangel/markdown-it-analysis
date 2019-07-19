# markdown-it-container

相信很多人都用过 `:::warning 这样的语法吧 :::`。这种能力在 MarkdownIt 的源码是不具备的，是 `markdown-it-container` 插件赋予的。

插件的逻辑全部在 `index.js` 里面。先忽略一些细节，从整体上来感受下大致的逻辑

```js
module.exports = function container_plugin(md, name, options) {

  function validateDefault(params) {
    // ....
  }

  function renderDefault(tokens, idx, _options, env, self) {
    // ....
  }

  options = options || {};

  var min_markers = 3,
      marker_str  = options.marker || ':',
      marker_char = marker_str.charCodeAt(0),
      marker_len  = marker_str.length,
      validate    = options.validate || validateDefault,
      render      = options.render || renderDefault;

  function container(state, startLine, endLine, silent) {
    // ....
  }

  md.block.ruler.before('fence', 'container_' + name, container, {
    alt: [ 'paragraph', 'reference', 'blockquote', 'list' ]
  });
  md.renderer.rules['container_' + name + '_open'] = render;
  md.renderer.rules['container_' + name + '_close'] = render;
};
```

函数内部先处理一些 options 的逻辑。可以自定义一些属性或者方法，比如 `marker`、`validate`、`render` 等。之后，在 ParserBlock 的 fence rule 后面追加了一个 container rule，并且在 render rule 后面加了对相应 type 的 token 的 rule。如果你传入的 name 为 `md`，那么生成的 token 的 type 就有 `container_md_open` 和 `container_md_close`，最后调用 `renderDefault` 或者 传入的 `options.render` 函数输出 HTML 字符串。

- **container函数**

  先分析最核心的 container 函数。

  ```js
  function container(state, startLine, endLine, silent) {
    var pos, nextLine, marker_count, markup, params, token,
        old_parent, old_line_max,
        auto_closed = false,
        start = state.bMarks[startLine] + state.tShift[startLine],
        max = state.eMarks[startLine];

    // Check out the first character quickly,
    // this should filter out most of non-containers
    //
    if (marker_char !== state.src.charCodeAt(start)) { return false; }

    // Check out the rest of the marker string
    //
    for (pos = start + 1; pos <= max; pos++) {
      if (marker_str[(pos - start) % marker_len] !== state.src[pos]) {
        break;
      }
    }

    marker_count = Math.floor((pos - start) / marker_len);
    if (marker_count < min_markers) { return false; }
    pos -= (pos - start) % marker_len;

    markup = state.src.slice(start, pos);
    params = state.src.slice(pos, max);
    if (!validate(params)) { return false; }

    // Since start is found, we can report success here in validation mode
    //
    if (silent) { return true; }

    // Search for the end of the block
    //
    nextLine = startLine;

    for (;;) {
      nextLine++;
      if (nextLine >= endLine) {
        // unclosed block should be autoclosed by end of document.
        // also block seems to be autoclosed by end of parent
        break;
      }

      start = state.bMarks[nextLine] + state.tShift[nextLine];
      max = state.eMarks[nextLine];

      if (start < max && state.sCount[nextLine] < state.blkIndent) {
        // non-empty line with negative indent should stop the list:
        // - ```
        //  test
        break;
      }

      if (marker_char !== state.src.charCodeAt(start)) { continue; }

      if (state.sCount[nextLine] - state.blkIndent >= 4) {
        // closing fence should be indented less than 4 spaces
        continue;
      }

      for (pos = start + 1; pos <= max; pos++) {
        if (marker_str[(pos - start) % marker_len] !== state.src[pos]) {
          break;
        }
      }

      // closing code fence must be at least as long as the opening one
      if (Math.floor((pos - start) / marker_len) < marker_count) { continue; }

      // make sure tail has spaces only
      pos -= (pos - start) % marker_len;
      pos = state.skipSpaces(pos);

      if (pos < max) { continue; }

      // found!
      auto_closed = true;
      break;
    }

    old_parent = state.parentType;
    old_line_max = state.lineMax;
    state.parentType = 'container';

    // this will prevent lazy continuations from ever going past our end marker
    state.lineMax = nextLine;

    token        = state.push('container_' + name + '_open', 'div', 1);
    token.markup = markup;
    token.block  = true;
    token.info   = params;
    token.map    = [ startLine, nextLine ];

    state.md.block.tokenize(state, startLine + 1, nextLine);

    token        = state.push('container_' + name + '_close', 'div', -1);
    token.markup = state.src.slice(start, pos);
    token.block  = true;

    state.parentType = old_parent;
    state.lineMax = old_line_max;
    state.line = nextLine + (auto_closed ? 1 : 0);

    return true;
  }
  ```

  `container` 函数的原则就是**尽早 return**。

  第一步：先判断第一个字符是不是你传入的 `options.marker` 或者默认的 `:`。

  第二步：校验 marker 的数量，是不是大于 3。

  第三步：拿着 `markup` 和 `params` 做校验。举个栗子： `:::warning 这是一个测试 :::`。其中，`markup` 就是 `:::`，`params` 就是 `warning`。

  第四步：先生成 type 为 `'container_' + name + '_open'` 的 token，再调用默认的 `tokenize` 方法解析 `:::warning` 内部的字符串，最后再生成 type 为 `'container_' + name + '_close'` 的 token。

- **renderDefault函数**

  ```js
  function renderDefault(tokens, idx, _options, env, self) {

    // add a class to the opening tag
    if (tokens[idx].nesting === 1) {
      tokens[idx].attrPush([ 'class', name ]);
    }

    return self.renderToken(tokens, idx, _options, env, self);
  }
  ```

  生成完 tokens 之后，最后如果是 type 为 `'container_' + name + '_open'` 或 `'container_' + name + '_close'` 的 token，都会走到 `renderDefault` 渲染函数。

  `renderDefault` 函数的处理逻辑很简单，就是给最外面的元素加上对应的 className。这样只要在 css 文件里面写了对应 className 的样式，就能渲染出很好看的 container 样式了，最后交由 md.renderToken 去生成对应的开标签或者闭字符串。举个栗子

  ```js
  const src = `
    :::warning
    这是一个 markdown-container
    :::
  `
  // 加载 warning 规则
  md.use(require('markdown-it-container'), 'warning', {})
  // output
  const output = md.render(src)
  /**
   `<div class="warning">
      <p>这是一个 markdown-container</p>
    </div>
   `
   */
  ```

  你也可以通过 `options.render` 去覆盖它，来实现更多的细节。