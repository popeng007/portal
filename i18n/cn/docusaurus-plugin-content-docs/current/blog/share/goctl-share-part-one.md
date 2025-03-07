# goctl 效能工具深度解析（一）

## 1. goctl 的诞生
   goctl 的最早功能是为了解决 GRPC 内网调试问题，大约是在 2019 年，在我们的生产环境中，rpc 是内网隔离的，不可通过外网访问，为了快速去 mock 一些线上 RPC client 的请求，就简单的实现了第一版本的代码生成，主要目的是去访问 RPC Server 做一些调试。
## 2. 为什么需要 goctl？
- 降低沟通成本

  沟通，是团队协作进行信息交换的一种形式，沟通的方式有很多种，会议沟通、文档沟通、聊天交流，相信不管是哪种方式，沟通都是团队中最难的一个环节，会议沟通需要占用大量时间，动则半小时起步，文档沟通同样，也会占据大量时间去构思和编写大篇幅的文档，最后可能还没表达出预期目标，线上聊天，需要双方都在线上才能进行信息交换，当然我们这里沟通交换的信息更多是指开发中的一些内容，如接口信息、部署信息等。

- 降低团队耦合

  有了沟通，那么团队之间的协作的耦合是避免不了的，例如：在前后端开发中，最大的耦合是接口的耦合，前端完成了规定 UI 绘制后，需要等待后端的接口部署到对应环境才能实现功能的调试，在此期间，前端的团队资源就会大大浪费，由此还会导致项目的延期等问题。

- 提高开发效率

  除了沟通成本和团队耦合以外，每个团队在进行项目开发时也有很多时间是在做重复的工作，例如：我们在开发一个新的功能时，需要去定义接口，编写接口文档，编码准备工作，业务开发，model 文件，编写 Dockerfile 文件，编写 k8s yaml 文件，在这些上面我们可以在每个环节上都有提升的空间，让用户将真正的时间集中在业务开发上。

- 降低错误率

  在之前的开发实践中，经常会出现 grpc server 实现不完全的问题，grpc server 实现类经常会出现编译不过的情况；除此之外，数据库查询层代码开发，sql 语句的编写多参，少参，参数错位，在编译过程中很难发现，一般可能到 QA 环节才能发现，更甚者会导致线上问题。

## 3. 怎么理解开发规范？
   开发规范，包括编码规范，工程规范，提交规范，代码审核规范，部署规范等等，团队内如果将开发规范统一起来，其收益可想而知，举个例子：假如你所在的公司没有统一开发规范，这时需要你去做一些跨部门协作或者支持，焕然一新的开发环境让你望而却步，还没开始就有了抵触的念头，这岂不是违背了支持的初衷。

## 4. 怎么理解工程效率？
工程效率，要理解工程效率，可以先看看『效率』的定义：

```text
效率是指单位时间内完成的工作量
```

效率的目标是快，但快并不是效率，换句话说就是在单位时间内完成的高质量工作，这才是我们要追求的目标，那么工程效率就是为了『缩短从开发到线上的距离』

# 二 、goctl 的安装及功能介绍
## 1. 介绍
- goctl 定义

    > goctl 定义，准确说是定位吧，可以理解为一个代码生成脚手架，其核心思想是通过对 DSL 进行语法解析、语义分析到数据驱动实现完成代码生成功能，旨在帮助开发者从工程效率角度提高开发者的开发效率。
  
- 解决的问题
  - 提高开发效率：goctl 的代码生成覆盖了Go、Java、Android、iOS、TS 等多门语言，通过 goctl 可以一键生成各端代码，让开发者将精力集中在业务开发上。
  - 降低沟通成本：使用 zero-api 替代 API 文档，各端以 zero-api 为中心生成各端代码，无需要在通过会议、API 文档来进行接口信息传递。
  - 降低耦合度： 在之前的开发时间中，没有 goctl 之前，我们采用的是传统的 API 文档来作为接口信息传递的载体，然后通过会议来进行二次 review，我们都知道，前端开发工作中，除了 UI 的绘制外就是 UI 联调，如果比较积极的前端开发者，可能会自己利用一些脚手架或者自己 mock 一些数据去进行 UI 联调，反之，也有原地等待后端部署后再联调的 case，原因是自己写 mock 数据太耗时，也不想去动，如果前端的开发进度在后端之前，那么在后端部署到环境中这期间，前端只能安静的原地等待，这不仅是一种资源的浪费，也会造成项目进度的延期，有了 zero-api ，前端可以利用 goctl 去生成适量的 mock 数据来进行 UI联调。

- 发展历史
![goctl-history](../../resource/goctl-history.svg)

 goctl 发展历史可以大概的过一下，挑几个标志性的功能讲讲吧！
 首先是 goctl rpc，goctl rpc 的第一版本就是 goctl 的雏形，主要是为了生成 client 代码对 rpc 做调试使用，具体就不展开了。goctl rpc 的发展经过了很多曲折，印象比较深刻的应该算 goctl rpc proto 向 goctl rpc protoc 的转变，这期间为了解决很多问题，才做了相应的变更，具体细节在分享 rpc 代码生成使用那块再详细分享，在这里面其实对我来说印象最深刻的一个功能算是 goctl model 了，其算是我开始维护 goctl 的一个入手点吧，也是我从业务开发实践中挖掘的解决方案吧，终究我还是比较 "懒"，记得在当时开发一个业务模块，由于业务比较大，需要创建的数据表也很多，在没有 goctl 前都是自己去手动编码完成，这一块我记忆犹新，花了仅一天的时间来写 model 文件，因为我没有用 orm 的习惯，所以这块 sql 完全是裸 sql 语句，服务跑起来后才发现有很多 sql 传参上的错误，基本都是多参少参问题，除此之外，每次新增索引，或者字段变更，缓存的相关逻辑维护也让我十分头疼，于是结合当时的开发最佳实践，将其按照缓存与否把这块的工作交给了 goctl，第一版本 goctl model 生成主要是网页版本，后面陆续集成到 goctl 里了，这给我后续开发带来了很大的开发效率收益。 
 
- 展望未来
  
  goctl 集成了比较多的功能，其基本是以代码生成为主，开发规范为辅，goctl 是围绕 zero-api 为中心做代码生成的，因此在将来，zero-api 的建设会成为我们的重心：
  - ast 的改造：替换原有比较重的 antlr4，可以解决很多人安装 goctl 时 内存不够，系统架构不支持的各种问题，这
  - goctl mock 的支持，最大限度解耦各端依赖耦合
## 2. 安装
   Goctl 安装有两种方式，分别是 go get 或者 go install，其次是 docker。

### go get/install
```bash 
# Go 1.16 之前版本
# $ GO111MODULE=on GOPROXY=https://goproxy.cn/,direct go get -u github.com/zeromicro/go-zero/tools/goctl@latest

# Go 1.16 及以后版本
$ GOPROXY=https://goproxy.cn/,direct go install github.com/zeromicro/go-zero/tools/goctl@latest

# macOS
# $ brew install goctl
# 验证
$ goctl --version
goctl version 1.3.6-beta darwin/amd64

# 查看 goctl
$ cd $GOPATH/bin && ls | grep "goctl"
```

:::tip
如果执行 goctl --version 提示 command not found: goctl 的错误，请先排查 $GOBIN 是否在环境变量下。
:::

### docker

```bash
# 1. pull
$ docker pull kevinwan/goctl
....

# 2. ls
$ docker image ls | grep "goctl"
kevinwan/goctl        latest    7fd46843de3d   5 weeks ago    383MB

# 3. run
$ docker run -it kevinwan/goctl /bin/sh

```

## 3. 功能介绍
   我们接下来看看功能介绍，通过指令 goctl --help 可以列出我们当前已经支持的功能，可能大家在看文档时也没有专心下来对这块指令做研究，只是用到一些高频指令，今天既然是 goctl 的专题分享，我们就花点时间来一起过过，带着大家从 0 到 1 熟悉 goctl。
```bash
$ goctl --help                                                       
A cli tool to generate api, zrpc, model code

Usage:
goctl [command]

Available Commands:
api         Generate api related files
bug         Report a bug
completion  Generate the autocompletion script for the specified shell
docker      Generate Dockerfile
env         Check or edit goctl environment
help        Help about any command
kube        Generate kubernetes files
migrate     Migrate from tal-tech to zeromicro
model       Generate model code
quickstart  quickly start a project
rpc         Generate rpc code
template    Template operation
upgrade     Upgrade goctl to latest version

Flags:
-h, --help      help for goctl
-v, --version   version for goctl

Use "goctl [command] --help" for more information about a command.
```

:::tip
goctl 的功能比较丰富，所以难免指令参数比较多，大家在使用时除了看文档外，也善用 --help 这个flag，因为他会告诉你每个指令或者子指令的用法，需要什么参数，参数类型是字符串还是布尔，是必填还是可选，切不可自己猜测他应该用什么 flag，虽然 goctl 是我自己主要维护，其实很多时候，有些指令比较文档，没有太多 feature 去更新，我也记不住，我也是通过 --help 去查看功能介绍的
:::

### 重点演示

#### goctl completion

查看 goctl 版本

```bash
$ goctl --version
goctl version 1.3.5 darwin/amd64
```

##### 1.3.5

:::tip
本节内容仅适合 goctl 版本在 1.3.5及之前的的自动补全设置参考，1.3.5版本及之前仅支持类 Unix 操作系统，不支持 Windows 的自动补全，1.3.5 之后的请跳过此节内容
:::

1. 执行 goctl completion -h 查看使用说明

```bash
$ goctl completion --help                                
   NAME:
   goctl completion - generation completion script, it only works for unix-like OS

USAGE:
goctl completion [command options] [arguments...]

OPTIONS:
--name value, -n value  the filename of auto complete script, default is [goctl_autocomplete]
```

根据帮助说明得知，goctl completion 支持通过 --name 传递一个参数生成自动补全脚本文件名，但该参数为非必填，默认不填时为goctl_autocomplete 文件，其位于 $GOCTLHOME 目录下。

2. 执行 goctl completion 来生成自动补全脚本
```bash
$ goctl completion
generation auto completion success!
executes the following script to setting shell:
echo PROG=goctl source /Users/keson/.goctl/.auto_complete/zsh/goctl_autocomplete >> ~/.zshrc && source ~/.zshrc
or
echo PROG=goctl source /Users/keson/.goctl/.auto_complete/bash/goctl_autocomplete >> ~/.bashrc && source ~/.bashrc
```
   
3. 查看当前 shell

```bash
$ echo $SHELL                                                  
/bin/zsh
```

4. 选择对应 shell 的脚本执行，我这里为 zsh，所以执行指令如下

```bash
$ echo PROG=goctl source /Users/keson/.goctl/.auto_complete/zsh/goctl_autocomplete >> ~/.zshrc && source ~/.zshrc
```

5. 重启终端验证结果

```bash
# 键入 goctl 后按下 tab 键查看自动补全提示
$ goctl # tab
api            -- generate api related files
bug            -- report a bug
completion     -- generation completion script, it only works for unix-like OS
docker         -- generate Dockerfile
env            -- check or edit goctl environment
help        h  -- Shows a list of commands or help for one command
kube           -- generate kubernetes files
migrate        -- migrate from tal-tech to zeromicro
model          -- generate model code
rpc            -- generate rpc code
template       -- template operation
upgrade        -- upgrade goctl to latest version
```

##### 1.3.6

:::tip
本节内容仅适合 goctl 版本在 1.3.6 及之后的的自动补全设置参考，1.3.6 之前的请跳过此节内容
:::

1. 如果你是从 1.3.5 升级到 1.3.6 及以后，则需要执行如下指令来清除旧版本自动补全配置，这里以 zsh 为例子，否则请跳过这一步

```bash
# 编辑 ~/.zshrc
$ vim ~/.zshrc
# 找到 PROG=goctl 所在行并删除
# source
$ source ~/.zshrc && rm .zcompdump*
# 重启终端
```

2. 执行 goctl completion -h 查看使用说明，从使用帮助中可以查看到目前支持 bash、fish 、powershell、zsh 4 种 shell 的自动补全。
```bash
$ goctl completion --help
Generate the autocompletion script for goctl for the specified shell.
See each sub-command's help for details on how to use the generated script.

Usage:
goctl completion [command]

Available Commands:
bash        Generate the autocompletion script for bash
fish        Generate the autocompletion script for fish
powershell  Generate the autocompletion script for powershell
zsh         Generate the autocompletion script for zsh

Flags:
-h, --help   help for completion

Use "goctl completion [command] --help" for more information about a command.
```

3. 查看当前使用的 shell
```bash
$ echo $SHELL                                                  
/bin/zsh
```

4. 由于当前使用的 zsh，所以我们来看一下 zsh 的自动补全设置帮助
```bash
$ goctl completion zsh --help
Generate the autocompletion script for the zsh shell.

If shell completion is not already enabled in your environment you will need
to enable it.  You can execute the following once:

        echo "autoload -U compinit; compinit" >> ~/.zshrc

To load completions for every new session, execute once:

#### Linux:

        goctl completion zsh > "${fpath[1]}/_goctl"

#### macOS:

        goctl completion zsh > /usr/local/share/zsh/site-functions/_goctl

You will need to start a new shell for this setup to take effect.

Usage:
goctl completion zsh [flags]

Flags:
-h, --help              help for zsh
--no-descriptions   disable completion descriptions
```

从上文可以看出，根据操作系统的不同，自动补全的设置方式也不一样，我这里是 macOS，我们执行一下对应的指令：
```
$ goctl completion zsh > /usr/local/share/zsh/site-functions/_goctl
```

我们先重开一个终端来试一下：
```
# 键入 goctl 后按下 tab 键查看自动补全提示
$ goctl # tab
api         -- Generate api related files
bug         -- Report a bug
completion  -- Generate the autocompletion script for the specified shell
docker      -- Generate Dockerfile
env         -- Check or edit goctl environment
help        -- Help about any command
kube        -- Generate kubernetes files
migrate     -- Migrate from tal-tech to zeromicro
model       -- Generate model code
quickstart  -- quickly start a project
rpc         -- Generate rpc code
template    -- Template operation
upgrade     -- Upgrade goctl to latest version
```

**常见错误**
1. goctl Error: unknown flag: --generate-goctl-completion
2. (eval):1: command not found: _goctl
   按照 1.3.6 自动补全配置重新行配置

#### goctl migrate
帮助开发者从1.3.0 之前版本无缝迁移到 1.3.0及后的任意版本,如果使用的 go-zero 版本本身就是 1.3.0 及之后的，则无需做此操作。

**为什么需要迁移？**
go-zero 在 1.3.0 版本做了组织变更，即用 zeromicro 替换原来的 tal-tech 组织名称。

:::tip
我现在有一个演示项目 awesomemigrate 是用旧版本 goctl 生成的，该项目的 go-zero 的 module 为
:::

```go
module awesomemigrate

go 1.18

require github.com/tal-tech/go-zero v1.2.5

...
```


假设我们需要将该项目的 go-zero 升级至截止目前最新版本 1.3.3 ，要完成项目从 tal-tech 到 zeromicro 的升级 前需要确保 goctl 版本大于 v1.3.2，然后在执行 goctl 迁移指令执行，如果 goctl 版本小于 v1.3.2，则需要升级。
```bash
# 1. 查看当前 goctl 版本
$ goctl --version
goctl version 1.2.5 darwin/amd64

# 2. 由于 goctl 版本小于 v1.3.2，则需要升级至最新，如果 goctl 版本已经是1.3.2及之后了，则可以不用升级
$ go install github.com/zeromicro/go-zero/tools/goctl@latest

# 3. 查看一下迁移指令使用帮助
$ goctl migrate --help                                  
NAME:
goctl migrate - migrate from tal-tech to zeromicro

USAGE:
goctl migrate [command options] [arguments...]

DESCRIPTION:
migrate is a transition command to help users migrate their projects from tal-tech to zeromicro version

OPTIONS:
--verbose, -v    verbose enables extra logging
--version value  the target release version of github.com/zeromicro/go-zero to migrate

# 4. 进入待迁移的项目下，执行迁移指令
$ goctl migrate --version 1.3.3

# 5. 验证依赖是否存在问题
$ go test .
?       awesomemigrate  [no test files]
```

迁移完成后，我们再来看看 module 文件

```go
module awesomemigrate

go 1.18

require github.com/zeromicro/go-zero v1.3.3
...
```

在 project 搜索一下 tal-tech的匹配结果会发现为0：

#### goctl env
goctl env 主要是用于环境检测、安装、环境参数控制等功能，除此之外还可以查看当前 goctl 的一些环境信息，以至于用户在遇到 bug 时可以根据此环境上报当前 goctl 处于的环境。

首先我们看看其使用说明
```bash
$ goctl env --help                                    
NAME:
goctl env - check or edit goctl environment

USAGE:
goctl env command [command options] [arguments...]

COMMANDS:
install  goctl env installation
check    detect goctl env and dependency tools

OPTIONS:
--write value, -w value  edit goctl environment
--help, -h               show help
```

goctl env 支持环境查看、参数修改、依赖检测安装几个功能，我们依次来看一下

##### 1. 环境查看
在不输入任何 flag 的情况下，则是查看当前 goctl 环境。

```bash
$ goctl env                                             
GOCTL_OS=darwin                        
GOCTL_ARCH=amd64
GOCTL_HOME=/Users/keson/.goctl
GOCTL_DEBUG=false
GOCTL_CACHE=/Users/keson/.goctl/cache
GOCTL_VERSION=1.3.5
PROTOC_VERSION=3.19.4
PROTOC_GEN_GO_VERSION=v1.27.1
PROTO_GEN_GO_GRPC_VERSION=1.2.0
```

以上参数的大概说明为

|参数名称 | 参数类型 | 参数说明 |
|---|---|---|
| GOCTL_OS | STRING |当前操作系统，常见值有 darwin、windows、linux 等|
|GOCTL_ARCH|STRING|当前系统架构，常见值有 amd64, 386 等   |
|GOCTL_HOME|STRING|默认为 ~/.goctl |
|GOCTL_DEBUG|BOOLEAN|是否开启 debug 模式，默认为 false，目前暂未使用，主要是 goctl 开发中可能用于控制调试模式  |
|GOCTL_CACHE|STRING|goctl 缓存目录，主要缓存下载的依赖 protoc、protoc-gen-go、protoc-gen-go-grpc等|
|GOCTL_VERSION|STRING|当前 goctl 版本   |
|PROTOC_VERSION|STRING|当前 protoc 版本，如未安装则为空字符串|
|PROTOC_GEN_GO_VERSION|STRING|当前 protoc-gen-go 版本，如未安装或者从github.com/golang/protobuf 安装版本为 v1.3.2 之前的则为空字符串|
|PROTO_GEN_GO_GRPC_VERSION|STRING|当前 protoc-gen-go-grpc版本，如未安装则为空字符串 |

##### 2. 修改参数
比如我们将 GOCTL_DEBUG 修改为 true

```bash
# 修改前
$ goctl env | grep 'GOCTL_DEBUG'
GOCTL_DEBUG=false

# 修改 GOCTL_DEBUG 为 true
$ goctl env -w GOCTL_DEBUG=true

# 修改后
goctl env | grep 'GOCTL_DEBUG'
GOCTL_DEBUG=true
```

##### 3. 依赖检测/安装
我们来检测一下我当前的依赖都是否安装好，目前依赖检测内容为protoc、protoc-gen-go、protoc-gen-go-grpc

```bash
# 我们来检查一下依赖安装
$ goctl env check --verbose                              
[goctl-env]: preparing to check env

[goctl-env]: looking up "protoc"
[goctl-env]: "protoc" is not found in PATH

[goctl-env]: looking up "protoc-gen-go"
[goctl-env]: "protoc-gen-go" is not found in PATH

[goctl-env]: looking up "protoc-gen-go-grpc"
[goctl-env]: "protoc-gen-go-grpc" is not found in PATH

[goctl-env]: check env finish, some dependencies is not found in PATH, you can execute
command 'goctl env check --install' to install it, for details, please execute command
'goctl env check --help'

# 安装依赖，安装依赖有2中方式
# 方式一
# $ goctl env check --install --force --verbose

# 方式二
$ goctl env install --verbose -f                       
[goctl-env]: preparing to check env

[goctl-env]: looking up "protoc"
[goctl-env]: "protoc" is not found in PATH
[goctl-env]: preparing to install "protoc"
[goctl-env]: "protoc" is already installed in "/Users/keson/go/bin/protoc"

[goctl-env]: looking up "protoc-gen-go"
[goctl-env]: "protoc-gen-go" is not found in PATH
[goctl-env]: preparing to install "protoc-gen-go"
[goctl-env]: "protoc-gen-go" is already installed in "/Users/keson/go/bin/protoc-gen-go"

[goctl-env]: looking up "protoc-gen-go-grpc"
[goctl-env]: "protoc-gen-go-grpc" is not found in PATH
[goctl-env]: preparing to install "protoc-gen-go-grpc"
[goctl-env]: "protoc-gen-go-grpc" is already installed in "/Users/keson/go/bin/protoc-gen-go-grpc"

[goctl-env]: congratulations! your goctl environment is ready!
```

#### goctl rpc
首先我们来看一下该指令的使用帮助

```bash
goctl rpc -h                                                       
Generate rpc code

Usage:
goctl rpc [flags]
goctl rpc [command]

Available Commands:
new         Generate rpc demo service
protoc      Generate grpc code
template    Generate proto template

Flags:
--branch string   The branch of the remote repo, it does work with --remote
-h, --help            help for rpc
--home string     The goctl home path of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
--o string        Output a sample proto file
--remote string   The remote git repo of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
The git repo directory must be consistent with the https://github.com/zeromicro/go-zero-template directory structure

Use "goctl rpc [command] --help" for more information about a command.
```

该指令提供了三个子指令，new 是快速创建一个 zrpc 服务，protoc 是根据 proto 描述文件生成 zrpc 代码，而 template 则是快速生成一个 proto 模板，我们将重点围绕 goctl rpc protoc 指令来展开，看看 goctl rpc protoc 的使用说明:

```bash
goctl rpc protoc -h                                              
Generate grpc code

Usage:
goctl rpc protoc [flags]

Examples:
goctl rpc protoc xx.proto --go_out=./pb --go-grpc_out=./pb --zrpc_out=.

Flags:
--branch string     The branch of the remote repo, it does work with --remote
-h, --help              help for protoc
--home string       The goctl home path of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
--remote string     The remote git repo of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
The git repo directory must be consistent with the https://github.com/zeromicro/go-zero-template directory structure
--style string      The file naming format, see [https://github.com/zeromicro/go-zero/tree/master/tools/goctl/config/readme.md] (default "gozero")
-v, --verbose           Enable log output
--zrpc_out string   The zrpc output directory
```

goctl rpc protoc 到底怎么使用，为什么示例里面有 --go_out、--go-grpc_out 参数，而查看 help 的时候却没有看到介绍？目前我们先这样理解，我们先抛开 goctl，根据官方 protoc 生成 grpc 代码时的指令是什么？会用到哪些 flag，对于 zrpc 的代码生成，你可以理解为 goctl 生成 zrpc 只是在生成 grpc 代码指令的基础上加了 goctl rpc 前缀和一些 goctl 生成 zrpc 需要的 flag，而对于 protoc 及 插件 protoc-gen-go 、 protoc-gen-grpc-go 的相关参数 goctl 只是做继承，没有必要显示的在 help 里面再描述一遍，后面在分析源码时可以给大家详细讲解一些这块的设计和考虑，接下来我们用一个例子来演示一下，假设我们有一个 greet.proto 文件，抛开 goctl ，我们生成 grpc 代码需要执行的指令如下：

:::tip
如果不知道 grpc 代码是怎么生成的，这块建议参考官方文档 https://grpc.io/
:::

```bash
# 进入 greet.proto 所在目录
$ protoc greet.proto --go_out . --go-grpc_out .
$ tree
.
├── greet.proto
└── pb
├── greet.pb.go
└── greet_grpc.pb.go
```

那么按照上文对 goctl rpc protoc 的介绍，我们生成 zrpc 代码的指令则应该为

```bash
# 这里多了一个 --zrpc_out 用于指定 zrpc 代码的输出目录
$ goctl rpc protoc greet.proto --go_out=. --go-grpc_out=. --zrpc_out=. --verbose                               23:02:39  羽2.625s
[goctl-env]: preparing to check env

[goctl-env]: looking up "protoc"
[goctl-env]: "protoc" is installed

[goctl-env]: looking up "protoc-gen-go"
[goctl-env]: "protoc-gen-go" is installed

[goctl-env]: looking up "protoc-gen-go-grpc"
[goctl-env]: "protoc-gen-go-grpc" is installed

[goctl-env]: congratulations! your goctl environment is ready!
[command]: protoc greet.proto --go_out . --go-grpc_out .
Done.
$ tree                                                                                                          23:02:52  羽819ms
.
├── etc
│   └── greet.yaml
├── go.mod
├── greet
│   └── greet.go
├── greet.go
├── greet.proto
├── internal
│   ├── config
│   │   └── config.go
│   ├── logic
│   │   └── pinglogic.go
│   ├── server
│   │   └── greetserver.go
│   └── svc
│       └── servicecontext.go
└── pb
├── greet.pb.go
└── greet_grpc.pb.go

8 directories, 11 files
```

:::tip
如果在生成代码时需要输出日志，可以加 --verbose 来显示日志。
goctl rpc protoc 在生成 zrpc 代码时会先对你的环境依赖进行检测，如果没有安装则会自动安装依赖再生成代码。
:::


#### 4. 编辑器插件
为了提升大家对 zero-api 文件的编写效率，我们分别对 intellij 和 vscode 提供了相应的编辑器插件, intellij 插件介绍及使用请参考 https://github.com/zeromicro/goctl-intellij， vscode 插件介绍及使用请参考 https://github.com/zeromicro/goctl-vscode/blob/main/README-cn.md
   
# 三、goctl 使用中遇到的问题
goctl 从最初的一个功能 rpc proxy 到当前版本(v1.3.5) 已经拥有13个一级指令和近30个二级指令, 期间 goctl 做了一些调整，而且，gcotl 本身的前进不发也非常快，他更像是在摸索中前进，朝着更快，更好用的方向发展，因此在迭代的路上，goctl 会显得有些不稳定，大家兴许也遇到很多问题，这里大概总结一下大家在社区中反馈比较多的一些问题来分享一下。

## 1. 386 架构上安装 goctl 失败！
   描述：antlr 生成的源码中，没有对 386 架构的 uint 的边界值进行很好的处理，导致边界溢出
   修复版本：[v1.3.3](https://github.com/zeromicro/go-zero/releases/tag/tools%2Fgoctl%2Fv1.3.3)

## 2. grpc 到底安装什么版本插件？
描述：熟悉 grpc 的人应该都知道，生成 grpc 代码的插件 protoc-gen-go 有两个仓库在维护，有3 个安装来源，2个维护仓库分别是：

```bash
# 1. golang
# github.com/golang/protobuf/protoc-gen-go
# 2. protocolbuffers
# github.com/protocolbuffers/protobuf-go/cmd/protoc-gen-go
```

3个安装来源是

```bash
# 1. golang 维护的仓库，目前已不推荐使用
# github.com/golang/protobuf/protoc-gen-go
# 2. protocolbuffers 维护的，目前推荐使用的
# github.com/protocolbuffers/protobuf-go/cmd/protoc-gen-go
# 3. goolge 安装，这其实和第二种安装的是一个二进制，他的源码就是protocolbuffers 维护的相同仓库
# google.golang.org/protobuf/cmd/protoc-gen-go
```

在 v.1.3.4前，如果使用 goctl rpc proto 生成 zrpc 代码则建议安装旧版本的插件,即 golang 维护的，因此该指令生成 zrpc 代码是 goctl 为了用户生成指令的简单，做了很厚的封装，但随之带来的问题就是难兼容，因此该指令已不推荐使用。

在v.1.3.4及以后，没有对 protoc-gen-go 做限制，用户可以自由选择不同的来源进行安装，但建议还是安装 protocolbuffer 维护的版本（官方文档已经替换为这个），而且必须要安装protoc-gen-grpc-go 插件，这样做的目的是跟着 grpc 官方走，不过用户不用有太大的心理负担，一下安装这么多依赖，很麻烦，goctl 都已经帮你实现了，你只要使用 goctl rpc protoc 生成代码时会自动检测依赖并安装。

综上所述，建议大家还在使用该指令的用户尽早用 goctl rpc protoc 替代。

## 3. 为什么 Windows 上生成 api/zrpc 代码时，总是提示 `The system cannot find the path specified` 类似错误？
```text
goctl.exe api go -api test.api -dir .
Stat : The system cannot find the path specified.
```

描述：产生该原因是因为 `go list -json -m` 获取 go module 时拿到的是一个固定值command-line-arguments，该问题已经在 https://github.com/zeromicro/go-zero/pull/1897 进行修复，版本将在 `v1.3.6` 生效。

## 4. No help topic 'proto'
描述：该指令在 `v1.3.4` 已经移除，用 `goctl rpc protoc` 替代

## 5. zrpc 生成代码时报 go_package 错误
```
protoc-gen-go: unable to determine Go import path for "greet.proto"

Please specify either:
• a "go_package" option in the .proto source file, or
• a "M" argument on the command line.

See https://developers.google.com/protocol-buffers/docs/reference/go-generated#package for more information.

--go_out: protoc-gen-go: Plugin failed with status code 1.
```

## 6. zrpc 生成代码时指定了 pb 的输出目录为当前服务 main 目录
```
the output of pb.go and _grpc.pb.go must not be the same with --zrpc_out:
pb output: /Users/keson/workspace/awesome-goctl/zwesome-zrpc/pb_in_main
zrpc out: /Users/keson/workspace/awesome-goctl/zwesome-zrpc/pb_in_main
```

## 7. 为什么我的代码生成的 pb client（不）带 client 标志？
这是旧版本 `goctl rpc proto` 生成 zrpc 代码时才有这个问题，目前已经移除了该指令。

## 8. 为什么我生成的目录结构和文档演示不一样？
对于 `goctl rpc protoc` 生成 zrpc 代码的目录结构，总的结构上是不会存在差异的，唯一存在差异的是 pb 的输出目录，这个取决于你指定的参数来控制，控制 pb 输出目录的因素有 `go_opt`、`go-grpc_opt`、`go_package` ，详情可参考 《[Protocol Buffers: Go Generated Code](https://developers.google.com/protocol-buffers/docs/reference/go-generated)》。

## 9. 为什么我安装了 goctl 编辑器插件还是不能高亮
打开 Goland 的设置，搜索 FileTypes 设置 api 文件后缀即可。

# 扩展阅读
1. [Cobra: A Framework for Modern CLI Apps in Go](https://cobra.dev/)