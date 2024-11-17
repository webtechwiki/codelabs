summary: 使用 Google Codelab 编写技术教程
id: publish-technical-tutorials-in-google-godelab-format
categories: codelab
tags: codelab
status: Published
authors: panhy
Feedback Link: https://github.com/webtechwiki/codelabs/issues

# 使用 Google Codelab 格式编写技术教程

## 概述  

Duration: 1

### Codelabs是什么？

Google Developers Codelab 提供了一种引导式编码实践教程体验。大部分 Codelab 会逐步介绍开发小应用或在现有应用中新增功能的过程。其友好的交互体验，以及丰富的资源，使得 Codelab 成为学习新技能的好方法。Google Codelabs 网站可以通过 <https://codelabs.developers.google.cn/?hl=zh-cn> 网址访问。当然，因为 Google 开源了这个网站构建工具，我们也可以基于该工具搭建我们自己的 Codelabs。

---

## 安装基础环境

需要的环境如下

- `Golang`: <https://golang.google.cn/>，需要使用到 Golang 语言插件，因此需要安装 Golang 语言环境；
- `claat`: <https://github.com/googlecodelabs/tools/tree/main/claat#install>，这是由 Google 维护的开源 Golang 命令行工具；
- `Nodejs`: <https://nodejs.org>，在编写文章的这个时间节点，虽然 `nodejs` 更新到了20，但推荐使用12，可以使用 `nvm` 或者 `fnm` 来管理你的本地 nodejs 版本；
- `gulp-cli`: 用于运行 `codelabs` 项目的 cli 工具。

将 Golang 语言二进制安装包解压到 `/usr/local/go` 目录，并且在用户目录下的 `.bash_profie`（或者 `.zsh_profile`） 添加以下内容

```bash
## 指定go语言的安装路径和go语言的项目路径
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
## 将go语言二进制可执行文件设置到PATH环境变量
export PATH=$PATH:$HOME/go/bin:$GOROOT/bin
```

安装好 `Golang` 之后，使用以下命令安装 `Claat`

```bash
go install github.com/googlecodelabs/tools/claat@latest
```

安装完成 `Nodejs` 之后，直接使用以下命令全局安装 `gulp-cli`

```bash
npm config set registry https://registry.npmmirror.com
```

## 运行 codelabs 网站
