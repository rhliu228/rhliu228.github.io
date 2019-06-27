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
通过查阅type=module属性的作用可以了解到，在当下，对于大部分用户而言，我们根本不需要把代码编译到 ES5，不仅体积大，而且运行速度慢。我们需要做的，就是把代码编译到 ES2015+，然后为少数使用老旧浏览器的用户保留一个 ES5 标准的备胎即可。其核心原理在于依赖 <script type="module">的支持来分辨浏览器对 ES2015+ 代码的支持，并且可以用<script nomodule>进行优雅降级。
支持 <script type="module"> 的浏览器，必然支持下面的特性：
* async/await
* Promise
* Class
* 箭头函数、Map/Set、fetch 等等...
  
而不支持 <script type="module"> 的老旧浏览器，会因为无法识别这个标签，而不去加载 ES2015+ 的代码。另外老旧的浏览器同样无法识别nomodule 属性，会自动忽略它，从而加载 ES5 标准的代码。

```
<script type="module" src="app.js"></script>

<script nomodule src="app-legacy.js"></script>   // legacy 是遗产的意思，在这里面是老旧的意思，理解成老旧的语法
```
想要支持module和nomodule的核心就是 Babel7的插件预设babel-preset-env。babel-preset-env将基于实际浏览器以及运行环境，自动确定babel插件以及polyfill，转义ES2015以及此版本以上的语法。而该preset的esmodules属性可以让我们直接编译到 ES2015+ 的语法。
改造一下webpack，构建两次，分别用不同的 babel 配置，就可以编译出两份文件。
CLI引入了@vue/babel-preset-app插件来提供babel-preset-env的功能，插件支持用户构建应用程序、UMD或原生web组件，其配置如下：
```
// @vue/babel-preset-app/index.js
// resolve targets
  let targets
  if (process.env.VUE_CLI_BABEL_TARGET_NODE) {
    // running tests in Node.js
    targets = { node: 'current' }
  } else if (process.env.VUE_CLI_BUILD_TARGET === 'wc' || process.env.VUE_CLI_BUILD_TARGET === 'wc-async') {
    // targeting browsers that at least support ES2015 classes
    // https://github.com/babel/babel/blob/master/packages/babel-preset-env/data/plugins.json#L52-L61
    targets = {
      browsers: [
        'Chrome >= 49',
        'Firefox >= 45',
        'Safari >= 10',
        'Edge >= 13',
        'iOS >= 10',
        'Electron >= 0.36'
      ]
    }
  } else if (process.env.VUE_CLI_MODERN_BUILD) {
    // targeting browsers that support <script type="module">
    targets = { esmodules: true }
  } else {
    targets = rawTargets
  }

  // included-by-default polyfills. These are common polyfills that 3rd party
  // dependencies may rely on (e.g. Vuex relies on Promise), but since with
  // useBuiltIns: 'usage' we won't be running Babel on these deps, they need to
  // be force-included.
  let polyfills
  const buildTarget = process.env.VUE_CLI_BUILD_TARGET || 'app'
  if (
    buildTarget === 'app' &&
    useBuiltIns === 'usage' &&
    !process.env.VUE_CLI_BABEL_TARGET_NODE &&
    !process.env.VUE_CLI_MODERN_BUILD
  ) {
    polyfills = getPolyfills(targets, userPolyfills || defaultPolyfills, {
      ignoreBrowserslistConfig,
      configPath
    })
    plugins.push([
      require('./polyfillsPlugin'),
      { polyfills, entryFiles, useAbsolutePath: !!absoluteRuntime }
    ])
  } else {
    polyfills = []
  }
```
翻看@vue/cli-service模块的源码， 我在里面找到CLI为了支持处理模板中的 module 和 nomodule 属性而引入的webpack插件：ModernModePlugin。该插件暴露了一个es6类，在该类的prototype属性上的apply方法定义如下：
```
  apply (compiler) {
    if (!this.isModernBuild) {
      this.applyLegacy(compiler)
    } else {
      this.applyModern(compiler)
    }
  }
```
isModernBuild属性表示当前构建是否生成ES2015+版本的代码。若是为false，则调用applyLegacy方法，并把编译器对象作为参数传递过去：
```
applyLegacy (compiler) {
  const ID = `vue-cli-legacy-bundle`
  compiler.hooks.compilation.tap(ID, compilation => {
    compilation.hooks.htmlWebpackPluginAlterAssetTags.tapAsync(ID, async (data, cb) => {
      // get stats, write to disk
      await fs.ensureDir(this.targetDir)
      const htmlName = path.basename(data.plugin.options.filename)
      // Watch out for output files in sub directories
      const htmlPath = path.dirname(data.plugin.options.filename)
      const tempFilename = path.join(this.targetDir, htmlPath, `legacy-assets-${htmlName}.json`)
      await fs.mkdirp(path.dirname(tempFilename))
      await fs.writeFile(tempFilename, JSON.stringify(data.body))
      cb()
    })
  })
}
```

若为false，则调用applyModern方法：
```
  // use <script type="module"> for modern assets
  data.body.forEach(tag => {
    if (tag.tagName === 'script' && tag.attributes) {
      tag.attributes.type = 'module'
    }
  })

  // use <link rel="modulepreload"> instead of <link rel="preload">
  // for modern assets
  data.head.forEach(tag => {
    if (tag.tagName === 'link' &&
        tag.attributes.rel === 'preload' &&
        tag.attributes.as === 'script') {
      tag.attributes.rel = 'modulepreload'
    }
  })

  // inject links for legacy assets as <script nomodule>
  const htmlName = path.basename(data.plugin.options.filename)
  // Watch out for output files in sub directories
  const htmlPath = path.dirname(data.plugin.options.filename)
  const tempFilename = path.join(this.targetDir, htmlPath, `legacy-assets-${htmlName}.json`)
  const legacyAssets = JSON.parse(await fs.readFile(tempFilename, 'utf-8'))
    .filter(a => a.tagName === 'script' && a.attributes)
  legacyAssets.forEach(a => { a.attributes.nomodule = '' })

  if (this.unsafeInline) {
    // inject inline Safari 10 nomodule fix
    data.body.push({
      tagName: 'script',
      closeTag: true,
      innerHTML: safariFix
    })
  } else {
    // inject the fix as an external script
    const safariFixPath = path.join(this.jsDirectory, 'safari-nomodule-fix.js')
    const fullSafariFixPath = path.join(compilation.options.output.publicPath, safariFixPath)
    compilation.assets[safariFixPath] = {
      source: function () {
        return new Buffer(safariFix)
      },
      size: function () {
        return Buffer.byteLength(safariFix)
      }
    }
    data.body.push({
      tagName: 'script',
      closeTag: true,
      attributes: {
        src: fullSafariFixPath
      }
    })
  }

  data.body.push(...legacyAssets)
  await fs.remove(tempFilename)
  cb()
  // 在 htmlWebpackPlugin 处理好模板的时候再处理下，把页面上 <script nomudule=""> 处理成 <script nomudule>
  compilation.hooks.htmlWebpackPluginAfterHtmlProcessing.tap(ID, data => {
    data.html = data.html.replace(/\snomodule="">/g, ' nomodule>')
  })
```

ios10.3版本有个bug，不支持 nomodule 属性，这样带来的后果就是 10.3 版本的 IOS 同时执行两份 JS 文件。有个hack写法可以解决这个问题：
```
// 这个会解决 10.3 版本同时加载 nomodule 脚本的 bug，但是仅限于外部脚本，对于内联的是没用的
// fix 的核心就是利用 document 的 beforeload 事件来阻止 nomodule 标签的脚本加载
(function() {
  var check = document.createElement('script');
  if (!('noModule' in check) && 'onbeforeload' in check) {
    var support = false;
    document.addEventListener('beforeload', function(e) {
      if (e.target === check) {
        support = true;
      } else if (!e.target.hasAttribute('nomodule') || !support) {
        return;
      }
      e.preventDefault();
    }, true);

    check.type = 'module';
    check.src = '.';
    document.head.appendChild(check);
    check.remove();
  }
}());
```

这段代码被ModernModePlugin引入并定义在常量safariFix中。
参考链接：
* [Webpack 构建策略 module 和 nomodule](https://github.com/shaodahong/dahong/issues/18)
