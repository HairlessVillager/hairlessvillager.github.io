---
title: 使用 Docker 搭建 miniob 开发环境
date: 2024-09-24 20:33
---

## 背景

学校的一门数据库课程要求在 [miniob](https://github.com/oceanbase/miniob) 上实现一些功能，评测方式和算法竞赛差不多，提交仓库地址和 commit hash 后评测系统会输入若干组数据，把程序输出和答案输出比较，一致则得分。

miniob 的开发环境设置比较繁琐，这里记录一下我自己的开发环境，以备不时之需。

## 准备工作

### 配置 Gitee 仓库

国外的 GitHub 由于众所周知的原因时常无妨访问，这里使用 Gitee 作为代码托管。在 Gitee 创建仓库时选择从 GitHub 克隆，地址：https://github.com/oceanbase/miniob.git。

Gitee 仓库创建好后，使用`git clone`命令把仓库克隆到本地，保证 `push`/`pop`/`fetch` 操作无报错。

### 配置 VSCode

安装 C/C++ Extension Pack 插件，主要为了实现代码跳转功能。这里亦可使用自己喜欢的 IDE。

### 配置 Docker

Docker 安装好后，使用`docker run -d --name miniob-dev --privileged -e REPO_ADDR=<your_repo_addr> oceanbase/miniob`命令创建 miniob 的开发容器。这里的`<your_repo_addr>`要换成自己的仓库地址，形如`https://gitee.com/xxx/miniob.git`。容器创建时会克隆仓库到容器内。

### 编译

容器创建后，使用`docker exec -it miniob-dev bash`进入容器的 Bash 环境。

1. 查看一下当前的位置。输入`pwd`显示当前目录是`/root`。输入`ls`查看目录下的文件/子目录，应该有`docker`和`source`两个文件夹。
2. 输入`cd source/miniob`进入 miniob 的仓库目录。
3. 输入`bash build.sh`运行编译脚本。编译过程需要大约两分钟，无报错则说明编译成功。
4. 输入`./build_debug/bin/observer -f ./etc/observer.ini -P cli`启动 miniob 的 CLI。这里可以自由输入 SQL 测试。

## 开发流程

这里最好创建两个 Terminal，一个在宿主机，一个在容器内。

### 在宿主机上修改源代码

略。

### 源代码复制到容器内

使用`docker cp ./src miniob-dev:/root/source/miniob/`命令复制。复制大小大概为1.8MB。

### 快速构建

使用`sudo apt-get install dos2unix`安装格式转换工具。

把以下内容写入`easy-build.sh`：

```bash
#!/bin/bash
rm observer.log.*
rm -r miniob

cd src/observer/sql/parser
bash gen_parser.sh

cd ../../../..
find ./src -type f -exec dos2unix {} \;
find ./src -type f -exec chmod 644 {} \;
bash build.sh --make -j12
echo "RUN ./build_debug/bin/observer -f ./etc/observer.ini -P cli"
```

然后使用`bash easy-build.sh`快速构建。

### 本地测试

`test/case`目录下有测试用例，可自行测试。

### 提交评测

在[训练营页面](https://open.oceanbase.com/train?questionId=600004)点击“立即评测”，输入仓库地址、commit hash后提交，评测会需要很长时间，可在此页面查看评测结果。