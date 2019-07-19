# 前言

要想理清 MarkdownIt 源码的来龙去脉，必须要清楚两个基础类—— Ruler & Token。

## Token

俗称词法单元。

md 接收一个字符串，经过一系列的 parser 的处理，变成了一个个 token，接着调用 render 对应的rule，将 token 作为输入，最后输出 HTML 字符串。

先来看下 Token 的定义，位于 `lib/token.js`。

```js
function Token(type, tag, nesting) {

  this.type     = type;

  this.tag      = tag;

  this.attrs    = null;

  this.map      = null;

  this.nesting  = nesting;

  this.level    = 0;

  this.children = null;

  this.content  = '';

  this.markup   = '';

  this.info     = '';

  this.meta     = null;

  this.block    = false;

  this.hidden   = false;
}
```

- **type**

  token 的类型，比如 `paragraph_open` 、`paragraph_close`、`hr`，分别会渲染成 `<p>`、`</p>`、`<hr>`。

- **tag**

  标签名称，比如 `p`、`strong`、`''`(空字符串。代表是文字)等等。

- **attrs**

  HTML 标签元素的特性，如果存在，则是一个二维数组，比如 `[["href", "http://dev.nodeca.com"]]`

- **map**

  token 的位置信息，数组只有两个元素，前者是起始行、后者是结束行。

- **nesting**

  标签的类型，1 是开标签，0 是自闭合标签，-1 是关标签。例如 `<p>`、`<hr>`、`</p>`。

- **level**

  缩紧的层级。

- **children**

  子token。只有 type 为 inline 或者 image 的 token 会有 children。因为 inline 类型的 token 还会经历一次 parser，提取出更详细的 token，比如以下的场景。

  ```js
  const src = '__advertisement__'
  const result = md.render(src)

  // 首先得到如下的一个 token
  {
    ...,
    content:"__Advertisement :)__",
    children: [Token, ...]
  }
  // 看出 content 是需要解析并提取出 "__"， "__" 需要被渲染成 <strong> 标签。因此 inline 类型的 children 是用来存放子 token的。
  ```

- **content**

  放置标签之间的内容。

- **markup**

  一些特定语法的标记。比如 "```" 表明是一个 code block。"**" 是强调的语法。"-" 或者 "+" 是一个列表。

- **info**

  type 为 fence 的 token 会有 info 属性。什么是 fence 呢，如下：

  ````js
  /**
  ```js
  let md = new MarkdownIt()
  ```
  **/
  ````

  上述的注释内部就是 fence token。它的 info 就是 `js`，`markup` 是 "```"。

- **meta**

  一般插件用来放任意数据的。

- **block**

  ParserCore 生成的 token 的 block 为 true，ParserInline 生成的 token 的 block 为 true。

- **hidden**

  如果为 true，该 token 不会被 render。

接下来看一下原型上的方法。

- **attrIndex()**

  ```js
  Token.prototype.attrIndex = function attrIndex(name) {
    var attrs, i, len;

    if (!this.attrs) { return -1; }

    attrs = this.attrs;

    for (i = 0, len = attrs.length; i < len; i++) {
      if (attrs[i][0] === name) { return i; }
    }
    return -1;
  };
  ```
  根据 attribute name 返回索引。

- **attrPush()**

  ```js
  Token.prototype.attrPush = function attrPush(attrData) {
    if (this.attrs) {
      this.attrs.push(attrData);
    } else {
      this.attrs = [ attrData ];
    }
  };
  ```

  添加一个 [name, value] 对。

- **attrSet**

  ```js
  Token.prototype.attrSet = function attrSet(name, value) {
    var idx = this.attrIndex(name),
        attrData = [ name, value ];

    if (idx < 0) {
      this.attrPush(attrData);
    } else {
      this.attrs[idx] = attrData;
    }
  };
  ```

  覆盖或添加一个 [name, value] 对。

- **attrGet**

  ```js
  Token.prototype.attrGet = function attrGet(name) {
    var idx = this.attrIndex(name), value = null;
    if (idx >= 0) {
      value = this.attrs[idx][1];
    }
    return value;
  };
  ```

  根据 name 返回属性值

- **attrJoin**

  ```js
  Token.prototype.attrJoin = function attrJoin(name, value) {
    var idx = this.attrIndex(name);

    if (idx < 0) {
      this.attrPush([ name, value ]);
    } else {
      this.attrs[idx][1] = this.attrs[idx][1] + ' ' + value;
    }
  };
  ```

  根据 name 将当前的 value 拼接到以前的 value 上去。

## Token 小结

Token 是 MarkdownIt 内部最基础的类，也是最小的分割单元。它是 parse 的产物，也是 output 的依据。

## Ruler

再来看下 MarkdownIt 另外的一个类 —— Ruler，可以认为它是职责链函数的管理器。因为它内部存储了很多 rule 函数，rule 的职能分为两种，一种是 parse rule，用来解析用户传入的字符串，生成 token，另一种是 render rule，在产出 token 之后，再根据 token 的类型调用不同的 render rule，最终吐出 HTML 字符串。

先从 constructor 说起。

```js
function Ruler() {
  this.__rules__ = [];

  this.__cache__ = null;
}
```

- **\_\_rules\_\_**

  用来放所有的 rule 对象，它的结构如下：

  ```js
  {
    name: XXX,
    enabled: Boolean, // 是否开启
    fn: Function(), // 处理函数
    alt: [ name2, name3 ] // 所属的职责链名称
  }
  ```

  有些人会对 alt 疑惑，这个先留个坑，在分析 `__compile__` 方法的时候会细说。

- **__cache__**

  用来存放 rule chain 的信息，它的结构如下：

  ```js
  {
    职责链名称: [rule1.fn, rule2.fn, ...]
  }
  ```

  > 注意: 默认有个名称为空字符串('')的 rule chain，它的 value 是一个囊括所有 rule.fn 的数组。

再来分析一下原型上各个方法的作用。

- **\_\_find\_\_**

  ```js
  Ruler.prototype.__find__ = function (name) {
    for (var i = 0; i < this.__rules__.length; i++) {
      if (this.__rules__[i].name === name) {
        return i;
      }
    }
    return -1;
  };
  ```

  根据 rule name 查找它在 \_\_rules\_\_ 的索引。

- **\_\_compile\_\_**

  ```js
  Ruler.prototype.__compile__ = function () {
    var self = this;
    var chains = [ '' ];

    // collect unique names
    self.__rules__.forEach(function (rule) {
      if (!rule.enabled) { return; }

      rule.alt.forEach(function (altName) {
        if (chains.indexOf(altName) < 0) {
          chains.push(altName);
        }
      });
    });

    self.__cache__ = {};

    chains.forEach(function (chain) {
      self.__cache__[chain] = [];
      self.__rules__.forEach(function (rule) {
        if (!rule.enabled) { return; }

        if (chain && rule.alt.indexOf(chain) < 0) { return; }

        self.__cache__[chain].push(rule.fn);
      });
    });
  };
  ```

  生成职责链信息。

  1. 先通过 \_\_rules\_\_ 的 rule 查找所有的 rule chain 对应的 key 名称。这个时候 rule 的 alt 属性就显得尤为重要，因为它表示除了属于默认的职责链之外，还属于 alt 所对应的职责链。默认存在一个 key 为空字符串('') 的职责链，任何 rule.fn 都属于这个职责链。
  2. 再将 rule.fn 映射到对应的 key 属性上，缓存在 \_\_cache\_\_ 属性上。

  举个栗子：

    ```js
    let ruler = new Ruler()
    ruler.push('rule1', rule1Fn, {
      alt: 'chainA'
    })
    ruler.push('rule2', rule2Fn, {
      alt: 'chainB'
    })
    ruler.push('rule3', rule3Fn, {
      alt: 'chainB'
    })
    ruler.__compile__()

    // 我们能得到如下的结构
    ruler.__cache__ = {
      '': [rule1Fn, rule2Fn, rule3Fn],
      'chainA': [rule1Fn],
      'chainB': [rule2Fn, rule3Fn],
    }
    // 得到了三个 rule chain,分别为 '', 'chainA', 'chainB'.
    ```

- **at**

  ```js
  Ruler.prototype.at = function (name, fn, options) {
    var index = this.__find__(name);
    var opt = options || {};

    if (index === -1) { throw new Error('Parser rule not found: ' + name); }

    this.__rules__[index].fn = fn;
    this.__rules__[index].alt = opt.alt || [];
    this.__cache__ = null;
  };
  ```

  用来替换某一个 rule 的 fn 或者更改它所属的 chain name。

- **before**

  ```js
  Ruler.prototype.before = function (beforeName, ruleName, fn, options) {
    var index = this.__find__(beforeName);
    var opt = options || {};

    if (index === -1) { throw new Error('Parser rule not found: ' + beforeName); }

    this.__rules__.splice(index, 0, {
      name: ruleName,
      enabled: true,
      fn: fn,
      alt: opt.alt || []
    });

    this.__cache__ = null;
  };
  ```

  在某个 rule 之前插入一个新 rule。

- **after**

  ```js
  Ruler.prototype.after = function (afterName, ruleName, fn, options) {
    var index = this.__find__(afterName);
    var opt = options || {};

    if (index === -1) { throw new Error('Parser rule not found: ' + afterName); }

    this.__rules__.splice(index + 1, 0, {
      name: ruleName,
      enabled: true,
      fn: fn,
      alt: opt.alt || []
    });

    this.__cache__ = null;
  };
  ```

  在某个 rule 之后插入一个新 rule。

- **push**

  ```js
  Ruler.prototype.push = function (ruleName, fn, options) {
    var opt = options || {};

    this.__rules__.push({
      name: ruleName,
      enabled: true,
      fn: fn,
      alt: opt.alt || []
    });

    this.__cache__ = null;
  };
  ```

  增加 rule。

- **enable**

  ```js
  Ruler.prototype.enable = function (list, ignoreInvalid) {
    if (!Array.isArray(list)) { list = [ list ]; }

    var result = [];

    // Search by name and enable
    list.forEach(function (name) {
      var idx = this.__find__(name);

      if (idx < 0) {
        if (ignoreInvalid) { return; }
        throw new Error('Rules manager: invalid rule name ' + name);
      }
      this.__rules__[idx].enabled = true;
      result.push(name);
    }, this);

    this.__cache__ = null;
    return result;
  };
  ```

  开启 list 列出的 rule，不影响其他 rule。

- **enableOnly**

  ```js
  Ruler.prototype.enableOnly = function (list, ignoreInvalid) {
    if (!Array.isArray(list)) { list = [ list ]; }

    this.__rules__.forEach(function (rule) { rule.enabled = false; });

    this.enable(list, ignoreInvalid);
  };
  ```

  先将其他 rule 都禁用，仅仅只开启 list 对应的 rule。

- **getRules**

  ```js
  Ruler.prototype.getRules = function (chainName) {
    if (this.__cache__ === null) {
      this.__compile__();
    }
    return this.__cache__[chainName] || [];
  };
  ```

  根据 rule chain 的 key，获取对应的 fn 函数队列。

  ## Ruler 小结

  可以看出，Ruler 是相当的灵活，不管是 `at`、`before`、`after`、`enable` 还是其他方法，都赋予了 Ruler 极大的灵活性与扩展性，作为使用方，可以利用这些优秀的架构设计满足特定需求。

  ## 总结

  分析完了 Token 与 Ruler 这些基础类，我们将进一步揭开 MarkdownIt 源码的面纱。后面的文章，再分析怎么从 src 字符串 parse 生成 token 的，token 又是怎么被 renderer.render 输出成最后的字符串。下一篇，我们将进入 MarkdownIt 的入口 parser —— CoreParser 的分析。