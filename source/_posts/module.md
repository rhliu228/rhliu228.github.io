---
-- webpack构建 ES2015+ module
--

用vue-cli3搭建项目，发现该工具减少了现代前端工具在配置上的烦恼，且其基于webpack 4的预配置提供构建设置，尽可能地在工具链中加入当前和未来的最佳实践。通过vue-cli3构建项目时，它会安装Vue CLI运行时命令（vue-cli-service），选择功能插件，生成必要的配置文件，然后就可以愉快地专注于业务代码了。而且该CLI工具尊重各个第三方工具的配置文件，如果我们对这些依赖进行配置，可以很轻松地更改配置文件或者通过webpack-merge合并到最终的配置中。

当我第一次使用vue-cli3的时候，只是赞叹于它漂亮的GUI界面，并不了解vue-cli3在背后集成了什么配置。它在package.json的script入口提供了分别用于开发模式和生产模式的两个命令，当然我还采用了lint检查工具，所以一共三个命令：
```
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build --modern",
    "lint": "vue-cli-service lint"
  },
```

在开发阶段，CLI工具会充分利用webpack4在development模式下内置的调试工具以及热更新、热替换等进行构建优化。而翻看npm run build后的代码，会发现打包后的代码跟我自己之前手动配置的差别很大。于是我开始查阅相关资料，了解Vue CLI3背后的配置以及其原理。

