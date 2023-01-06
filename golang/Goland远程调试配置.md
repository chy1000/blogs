# Goland远程调试配置

#### 前言

> 平常我是在 windows 系统下工作的，但很多时候写的代码是要放到 linux 服务器使用的。按照常规流程，我要在 windows 写好程序，然后再编译好上传到 linux 服务器进行测试。整个流程十分繁琐，还好 Goland 编辑器提供了远程调试的功能，我们只需要设置好，就可以在 windows 下编写代码，然后在 linux 服务器进行测试。

#### 远端服务器配置

> ##### １. 安装 Golang
>
> 下载安装好`Golang`并设置好环境变量
>
> ```shell
> # 下载安装
> wget https://golang.org/dl/go1.17.linux-amd64.tar.gz
> rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.linux-amd64.tar.gz
> # 执行下面的四行，设置环境变量
> export GO111MODULE=on
> export GOPROXY=https://goproxy.cn
> export GOROOT=/usr/local/go
> export PATH=$PATH:/usr/local/go/bin
> vi /etc/profile # 上面的设置，退出登录就失效了。所以要把它们写入 /etc/profile，复制上面五行加到文件最后
> source /etc/profile
> # 查看环境变量设置是否正确
> go env
> ```
>
> ##### ２. 安装 Delve 调试器
> 要实现远程调试，要先安装 Delve
>
> ```shell
> # 下载安装
> go get github.com/go-delve/delve/cmd/dlv
> # 开放防火墙 2345 端口(delve需要用到)
> firewall-cmd --permanent --add-port=2345/tcp;
> firewall-cmd --reload;
> ```
>
> 当我们安装好 Delve 后，假设我们已经配置好本地服务器并同步代码到远端服务器上面，执行如下代码开启监听：
>
> ```shell
> # 进入go代码项目目录
> cd /home/{你的go项目目录}
> go build -gcflags="all=-N -l" -o {你的go代码生成的执行程序}; /root/go/bin/dlv --listen=:2345 --headless=true --api-version=2 exec ./{你的go代码生成的执行程序}
> ```
>
> 到这里远端服务器就配置完成并开启了监听，等待本地工作机有调试请求过来。

#### 本地电脑配置

> ##### １. 安装 Goland 编辑器
>
> 访问 https://www.jetbrains.com/go/download/other.html 下载相关的版本，并一步步安装完成。由于 Goland 是有版权的，只能免费使用 30 天，如果我们想继续使用，需要安装“无限试用插件”，请自行网上搜索下载插件。下载好插件后，请按链接中教程安装和使用： https://mp.weixin.qq.com/s/7VvUbhWWBnnPnbfWdtxcyw
>
> ##### ２.  本地 Goland 编辑器配置
>
> 配置好 SFTP 服务器和自动同步，这样我们写完代码每次保存，代码就会自动同步到远端的服务器。
>
> ```shell
> # 配置 module
> 菜单：File > Settings > Go > Go Modules 
> 把 "Enable Go modules integration" 打上勾。如果不打勾配置，意味着依赖包都使用　vendor 目录里面的
> 
> # 配置 SFTP 服务器：
> 菜单： Tools > Deployment > Configurations 按提示配置 SFTP 服务器
> 
> # 配置远程调试：
> 菜单：Run > Edit Configurations...
> 新加 Go Remote
> 填写 Host (填写 远端服务器IP)
> 
> # 配置自动同步保存：
> 打开：Tools->Deployment->Options... 选择：On explicit save action（Ctrl+S）
> ```
>
> #####  3. 本地 go 项目创建 mod 文件
>
> 如果不创建 mod 文件，会导致设置不了断点。
>
> ```
> cd {go项目}
> go mod init xxx/xxx
> go mod tidy
> ```

配置完成后，我们在 Goland 编辑器点击 debug 并设置好断点，就可以跟踪调试了。