最近做 better-scroll 2.0 的重构，我们内部敲定 [ VuePress ](https://v1.vuepress.vuejs.org/zh/)来写 API 文档。VuePress 对 Markdown 文件的处理是通过 [markdown-it](https://github.com/markdown-it/markdown-it)。它内部的原理就是将 Markdown 渲染出来的字符串模板再交给 Vue-Loader 处理，而且能在文件里面写 Vue 组件，如此便能利用工程化的手段去解决写文档时候遇到的难题。

而这篇源码分析的起因，是因为我们在做 better-scroll 1.0 的时候收到过一个吐槽：

```markup
只有一个光秃秃的 API 文档，能不能在文档的里面提供所有的示例代码，这样我粘贴就行了！
```

有了社区的需求，我们就得想办法去解决。

当前，我们的示例 demo 都是基于 Vue 来实现的（后期会加入 React demo），一般实现方式都是采用 Vue SFC（单文件组件）。即以下的方式：

```html
<template>
    ...
</template>

<script>
  ...
</script>

<style>
  ...
</style>
```

我们希望能在 Markdown 文件里面利用类似于 `>>> example/demo.vue` 的这种语法，在 markdown-it 编译的时候去抽取对应路径文件的源代码，想要拓展以上“抽取代码”的能力，就必须要通过 `markdown-it` 的插件来实现。

这也就是研究 `markdown-it` 源码的契机！