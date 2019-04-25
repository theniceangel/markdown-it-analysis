最近做 better-scroll 2.0 的重构，我们内部敲定 [ VuePress ](https://v1.vuepress.vuejs.org/zh/)来写 API 文档。VuePress 的好处在于我们可以在 Markdown 里面写 Vue 组件，VuePress 也支持开发自定义主题和插件来增强 Markdown 的一些能力。如此我们就能享受到 Vue + Webpack 的开发体验。我们便能利用工程化的手段去解决写文档时候遇到的难题。

而这篇源码分析的起因，是因为我们在做 better-scroll 1.0 的时候收到过一个吐槽：

```markup
只有一个光秃秃的 API 文档，能不能在文档的里面提供所有的示例代码，这样我粘贴就行了！
```

有了社区的需求，我们就得想办法去解决。

当前，我们的示例 demo 都是基于 Vue 来实现的（后期会加入 React demo），一般实现方式都是采用 Vue SFC（单文件组件）。即以下的方式：

```markup
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

我们希望能在 Markdown 文件里面利用类似于 `>>> example/demo.vue` 的这种语法，在 Markdown 编译的时候去抽取文件的源代码。因此，我们需要深挖 VuePress 灵活的插件化的能力。

VuePress 是基于 [markdown-it](https://github.com/markdown-it/markdown-it) 来渲染 Markdown 的。我们想要拓展以上“抽取代码”的能力，就必须要通过 `markdown-it` 的插件来实现。

这也就是研究 `markdown-it` 源码及其周边插件原理的契机。