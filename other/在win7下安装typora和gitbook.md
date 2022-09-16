# 在win7下安装typora和gitbook

`Typora`是一款比较好用`Markdown`文档编辑器，我现在写`github`文章都是用它来写的。好用是好用，但最近要收费的，囊中羞涩只能另想办法了[笑哭]。

### Typora 过期解决方法 （转自网上教程）

`Typora`的安装比较简单，上官网下载安装包一键安装，然后使用下面的教程解决过期的问题。安装完`Typora`，接下来我们需要安装`gitbook`来生成电子书。

```bash
1. 按 Windows+R 打开运行窗口
2. 输入regedit，点确定，打开注册表
3. 依次展开计算机\HKEY_CURRENT_USER\Software\Typora
4. 然后在 Typora 上右键，点权限，选中 Administrtors ，把权限全部设置为拒绝。
```




### gitbook 安装

`gitbook`安装需要用到`nodejs`，所以首先要安装`nodejs`。很可惜，不知从什么版本开始，`nodejs`已经不再支持`win7`系统了。所以只能选择旧的版本。

https://nodejs.org/download/release/v10.14.1/node-v10.14.1-x64.msi

从上面的链接地址下载安装好`nodejs`。

接着通过`nodejs`安装`cnpm`。`npm`是`nodejs`官方的包管理器，但由于墙的关系，经常很多包或者软件安装不上，这时就需要用到`cnmp`了。`cnmp`是淘宝写的一个包管理器，当`npm`安装软件安装不上时，可以用`cnpm`来替代它。

```bash
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

使用`cnpm`来安装`gitbook`

```bash
cnpm install -g gitbook-cli
gitbook -V
```

这里有可能安装出错：

```bash
cnpm install -g gitbook-cli
internal/modules/cjs/loader.js:582
    throw err;
    ^
Error: Cannot find module 'fs/promises'
    at Function.Module._resolveFilename (internal/modules/cjs/loader.js:580:15)
    at Function.Module._load (internal/modules/cjs/loader.js:506:25)
    at Module.require (internal/modules/cjs/loader.js:636:17)
    at require (internal/modules/cjs/helpers.js:20:18)
    at Object.<anonymous> (C:\Users\chy\AppData\Roaming\npm\node_modules\cnpm\n
de_modules\npminstall\bin\install.js:10:12)
    at Module._compile (internal/modules/cjs/loader.js:688:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:699:10)
    at Module.load (internal/modules/cjs/loader.js:598:32)
    at tryModuleLoad (internal/modules/cjs/loader.js:537:12)
    at Function.Module._load (internal/modules/cjs/loader.js:529:3)
```

网上搜索得知可能是版本问题，于是删除`cnpm`，重新安装旧的版本，安装`gitbook`成功。

```bash
npm uninstall -g cnpm
npm install -g cnpm@7.1.0 --registry=https://registry.npm.taobao.org
cnpm install -g gitbook-cli
```

