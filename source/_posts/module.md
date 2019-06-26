---
-- webpack构建 ES2015+ module
--

用vue CLI3搭建项目，发现该工具减少了现代前端工具在配置上的烦恼，且其基于webpack 4的预配置提供构建设置，尽可能地在工具链中加入当前和未来的最佳实践。通过vue-cli3构建项目时，它会安装Vue CLI运行时命令（vue-cli-service），选择功能插件，生成必要的配置文件，然后就可以愉快地专注于业务代码了。而且该CLI工具尊重各个第三方工具的配置文件，如果我们对这些依赖进行配置，可以很轻松地更改配置文件或者通过webpack-merge合并到最终的配置中。

当我第一次使用vue-cli3的时候，只是赞叹于它漂亮的GUI界面，并不了解vue-cli3在背后集成了什么配置。它在package.json的script入口提供了分别用于开发模式和生产模式的两个命令，当然我还采用了lint检查工具，所以一共三个命令：
```
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build --modern",
    "lint": "vue-cli-service lint"
  },
```

在开发阶段，CLI工具会充分利用webpack4在development模式下内置的调试工具以及热更新、热替换等进行构建优化。而翻看npm run build后的代码，发现打包后的代码跟我自己之前手动配置的差别很大。于是我开始查阅相关资料，了解Vue CLI3背后的配置以及其原理。

观察打包后的js文件，首先会发现每个chunk都会有两个副本，其中一个副本在文件名后面添加了legacy后缀名，另一个则没有。而在html文件中，给引入带有legacy后缀js文件的script标签添加了nomodule属性，另一个script标签则添加了type=module属性。
通过查阅type=module属性的作用可以了解到，在当下，对于大部分用户而言，我们根本不需要把代码编译到 ES5，不仅体积大，而且运行速度慢。我们需要做的，就是把代码编译到 ES2015+，然后为少数使用老旧浏览器的用户保留一个 ES5 标准的备胎即可。其核心原理在于依赖 <script type="module">的支持来分辨浏览器对 ES2015+ 代码的支持，并且可以用<script nomodule>进行优雅降级（get到新技能的欢喜^^）
支持 <script type="module"> 的浏览器，必然支持下面的特性：
* async/await
* Promise
* Class
* 箭头函数、Map/Set、fetch 等等...
  
而不支持 <script type="module"> 的老旧浏览器，会因为无法识别这个标签，而不去加载 ES2015+ 的代码。另外老旧的浏览器同样无法识别 nomodule 属性，会自动忽略它，从而加载 ES5 标准的代码。

```
<script type="module" src="app.js"></script>

<script nomodule src="app-legacy.js"></script>   // legacy 是遗产的意思，在这里面是老旧的意思，理解成老旧的语法
```
想要支持 module 和 nomodule 核心就是 Babel7的插件预设babel-preset-env。babel-preset-env将基于实际浏览器以及运行环境，自动确定babel插件以及polyfill，转义ES2015以及此版本以上的语法。而该preset的esmodules属性可以让我们直接编译到 ES2015+ 的语法。改造一下webpack，构建两次，分别用不同的 babel 配置，就可以编译出两份文件。
