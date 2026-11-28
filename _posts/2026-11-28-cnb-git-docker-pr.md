---
layout: post
title: "CNB现代开发者实践课复盘（上）：Git底层、Docker分层与PR流水线"
date: 2026-11-28 23:30:00 +0800
categories: 深研自述
tags: [Git, Docker]
---

## 1. 问题与背景

空闲时间把CNB上那门《现代开发者实践课程：从Git到AI编程》从头到尾跑了一遍，配套的`ai-engineering-exercise`作业仓库和黑客松模板也都过完了。这门课前两章压根没碰大模型，全在讲Git和Docker。刚开始看还有点纳闷，等跑完第4章的NPC多智能体才反应过来——后面AI能在云端容器里自动切分支、跑测试、提PR，靠的全是这两块底座。

整门课跨度不小，合在一篇写太散，我按工程底座和AI实战拆成上下两篇。这篇记第1章Git（9个lab任务）、第2章Docker（lab+Go/scratch示例）和`docker-exercises`的3道作业。所有任务都在CNB云原生开发环境里跑通并过了验收。

---

## 2. Git：看懂对象和指针，命令不用死背

以前写C++矩阵乘法和脉动阵列仓库，Git日常也就用`add`、`commit`、`push`那几下，真遇到要拆提交或者追上游仓库就得现查。这次把底层存储模型理了一遍，顺畅很多。

### 2.1 三区、四类对象与分支指针

Git在本地维护三个区域：工作区→`git add`→暂存区→`git commit`→版本库（`.git/`）。这里有个之前我没太在意的细节：`git add`不是保存文件，而是挑改动。一次改了好几个文件，靠`git add`分批挑进暂存区，才能整理成几个逻辑独立的提交，不用把一天的改动全糊进一条commit。

落到底层，Git就是一个用内容哈希做索引的键值数据库，数据存成四类对象：
* **`blob`**：只存文件内容，不存文件名和路径。
* **`tree`**：存目录结构、文件名和权限，向下指向具体的`blob`或子`tree`。
* **`commit`**：一次完整快照，指向根`tree`和父提交`parent`，外加作者、时间戳和提交说明。
* **`tag`**：给某个`commit`起固定名字，一般拿来标版本（如`v1.0.0`）。

每次提交存的都是当前目录的完整快照，但没改过的文件会直接复用旧的`blob`哈希，所以并不浪费空间。第1章lab里用`git cat-file`顺着最新提交往下查了一遍，整条引用链一眼就能对上：

```bash
git cat-file -p HEAD          # 查看最新commit对象，拿到tree和parent哈希
git cat-file -p <tree-hash>   # 查看tree对象，列出目录里的文件名和blob哈希
git cat-file -p <blob-hash>   # 直接输出文件原始内容
git cat-file -t <hash>        # 验证对象类型（blob / tree / commit / tag）
```

顺着对象模型往下看，分支也没什么神秘的：分支根本不是把代码复制一份，它只是一个指向某个`commit`的可移动指针，`HEAD`指向当前所在的分支。所以建分支几乎零成本，每次新提交也只是把当前分支指针往前推一格。

### 2.2 两种远程协作流

本地跑顺后，多人协作主要分两种情况，区别在于对目标仓库有没有写权限：

| 协作场景 | 权限前提 | 远程配置 | 同步与合入路径 |
|---|---|---|---|
| 同仓库分支协同 | 有写权限（团队/私有库） | 单个`origin` | `git pull origin main`→开`feature`分支→`git push -u origin feature`→提PR→Review后合并 |
| 开源Fork协同 | 无写权限（外部开源库） | `origin`（个人Fork，可读写）+`upstream`（原仓库，只读） | `git fetch upstream`→`git rebase upstream/main`→推到自己的`origin`→跨仓向原仓库提PR |

开源协作里经常要追上游更新，一般用`git fetch upstream`配`git rebase upstream/main`，把自己的提交重演到上游最新提交后面，保持历史一条直线。但`rebase`只该用在还没推送或者只留在自己Fork上的提交；如果`rebase`后推送被拒，确认没问题用`--force-with-lease`，别对公共分支直接`git push --force`，否则会把别人已经合进去的提交覆盖掉。

### 2.3 用`.cnb.yml`给PR加自动拦截与AI初审

开分支提PR只能防人直接往`main`里乱推，真要保证合进去的代码不坏，还是得在PR上挂流水线自动跑检查。CNB的做法是在仓库根目录放`.cnb.yml`，监听`pull_request`事件自动拉容器执行：

```yaml
$:
  pull_request:
    - docker:
        image: python:3.12-slim
      stages:
        - name: lint check
          script: |
            pip install ruff
            ruff check .
        - name: unit test
          script: |
            pip install pytest
            pytest --tb=short
```

配置里`$`匹配所有分支，`docker.image`指定检查容器，`stages`按顺序执行。只要哪一步退出码非0，PR页面就直接拦住不让合并。第1章lab里我照这套给`feature-hello`的PR挂了`py_compile`语法检查，也在PR评论区试了召唤内置NPC做AI初审（`@npc/code-security(安全审计员)`和`@npc/hok(妲己)`）。

如果不想每次手动`@`，可以把`cnb pulls comment`写进`.cnb.yml`的`pull_request`步骤里自动触发。这类审查NPC是只读的，只看diff发评论、不改代码；用起来挺省事，但最终合不合并还是得自己拍板。

---

## 3. Docker：进程边界、缓存分层与多阶段构建

Git管住了代码版本，但换台机器配环境还是麻烦。第2章和`docker-exercises`干的事很直接：把代码连带运行环境一起打成镜像交付。

### 3.1 镜像命名与`PID=1`进程边界

Docker镜像格式是`<repository>/<image>:<tag>`。省略`repository`默认走Docker Hub，省略`tag`默认是`latest`；像CNB制品库里的私有镜像（如`docker.cnb.cool/looc/private-git-cnb`），必须先`docker login`才能拉。

练习和实际部署里经常用Alpine Linux打底，体积只有5MB左右，还自带`apk`包管理器。拿它跑容器时有个挺反直觉的地方：容器活不活着，全看主进程`PID=1`退没退。

跑一句`docker inspect alpine --format='{{.Config.Cmd}}'`就能看到，它的默认启动命令是`["/bin/sh"]`。直接执行`docker run alpine`时，容器拉起`/bin/sh`，发现没交互输入就立刻退出，容器也跟着停了；所以交互要加`-it`，挂后台要加`-d`。

明白`PID=1`之后，`docker attach`和`docker exec`的区别也就一眼看清了：
* **`docker attach <container_id>`**：直接接管容器主进程`PID=1`的标准输入输出，一旦退出，主进程结束，容器跟着停。
* **`docker exec -it <container_id> /bin/sh`**：在运行中的容器里另起一个进程，进去敲`ps -a`能看到两个`/bin/sh`，退出来完全不影响`PID=1`。

平时进后台容器排查问题，基本一律用`exec`，少碰`attach`。

### 3.2 `Dockerfile`、分层缓存与非root运行

在容器里装完软件再用`docker commit`也能打包出新镜像（比如往Alpine里装个`figlet`），但这种做法是黑盒的——下次要补装`git`或者升级依赖，只能重起容器手动重敲一遍，没法追踪。正经交付还是写声明式的`Dockerfile`，所有步骤留在文本里，能走Git评审，随时能重编。

几组容易搞混的指令放在一起对照更清楚：

| 指令组 | 核心区别与用法 |
|---|---|
| `COPY` vs `ADD` | 都能把文件拷进镜像；`ADD`额外支持URL和自动解压归档包，拷本地文件一般直接用`COPY` |
| `CMD` vs `ENTRYPOINT` | `CMD`是默认启动命令，`docker run`后面传参会覆盖它；`ENTRYPOINT`是固定入口点，外部参数会追加给它 |
| `ARG` vs `ENV` | `ARG`只在`docker build`构建阶段生效；`ENV`会保留到容器运行时环境变量里 |
| `EXPOSE` | 只声明容器监听端口，不会自动对外开放；真正做端口映射靠运行时的`-p 宿主机端口:容器端口` |

做`02-docker/lab`的任务三和`docker-exercises/02-python-flask-app`时，有两个习惯挺管用。一个是**利用分层缓存**：Docker每条指令都会生成一层，前面层没变，重新`docker build`会直接复用`CACHED`，一旦某层改了，它后面全得重跑。所以写Python应用镜像别一上来就`COPY . .`，先单独拷`requirements.txt`装完依赖，再拷业务代码，这样改代码重编时不用每次重跑`pip install`。另一个是用`USER`切到非root用户跑服务：

```dockerfile
FROM python:3.11-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

ENV FLASK_APP=app.py
EXPOSE 5000
CMD ["python", "app.py"]
```

再配一份`.dockerignore`把`__pycache__`、`*.pyc`和`.env`挡在构建上下文外面，既不破坏缓存，也避免把本地密钥打进镜像层。

### 3.3 多阶段构建：把GB级工具链从生产镜像里摘掉

Go、Rust这类编译型语言，如果走单阶段构建非常臃肿——最后跑的明明只是一个二进制文件，镜像里却背着整套编译器和工具链。

`02-docker/lab/golang_sample/`给了`Dockerfile.single`和`Dockerfile.multi`两份对照。单阶段直接拿`golang:1.23`编译运行，最终镜像超过1GB；换成多阶段构建，第一阶段负责静态编译，第二阶段切到`alpine:latest`只把编译出来的二进制拷过去，体积直接压到几十MB：

```dockerfile
# 第一阶段：在完整Go工具链镜像中静态编译
FROM golang:1.23 AS builder
WORKDIR /app
COPY go.mod .
COPY main.go .
ENV CGO_ENABLED=0
ENV GOOS=linux
RUN go mod tidy && go build -ldflags="-w -s" -o server .

# 第二阶段：只留极简运行环境和二进制产物
FROM alpine:latest
ARG PORT=8081
ENV PORT=${PORT}
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/server .
EXPOSE ${PORT}
CMD ["./server"]
```

这里有两个小细节：`CGO_ENABLED=0 GOOS=linux`关掉CGO确保编出静态二进制，`-ldflags="-w -s"`去掉调试信息和符号表进一步压体积；另外Alpine默认不带根证书，程序要发HTTPS请求的话得像上面那样补装`ca-certificates`。

配套作业`docker-exercises/03-rust-multistage-build`是同一套思路，要求最终镜像小于20MB且输出包含`"Hello from Rust"`。第一阶段用`FROM rust:1.75-slim as builder`，先装MUSL目标`rustup target add x86_64-unknown-linux-musl`，再跑`cargo build --target x86_64-unknown-linux-musl --release`编静态二进制；第二阶段同样切回`alpine:latest`，用`COPY --from=builder`把产物拿过来跑。生产镜像里既没源码也没编译器，传输快，攻击面也小。

### 3.4 镜像的起点：`scratch`空镜像

顺着`FROM`一路往上追，所有镜像的最底层是Docker内置的`scratch`。它是0字节的空镜像，里面没有文件系统、没有shell，连`tar`和`rm`都没有。所以在`FROM scratch`阶段不能写任何`RUN`指令，只能用`COPY`、`ADD`、`ENV`、`WORKDIR`、`CMD`这类不依赖容器内shell的指令。`02-docker/lab/scratch_sample`演示的就是从`scratch`拼出自定义Ubuntu环境，跑完这一步，镜像分层最底下那层也就透了。

---

## 4. 踩坑、边界与下篇

把前两章实操里容易踩的几个坑和边界留个底：
* **密钥进Git历史删不掉**：`.env`、API Key、访问令牌绝不能提交进仓库。哪怕下一条commit把它删了，旧的`blob`对象依然留在`.git/`历史里，动手前先配好`.gitignore`和`.dockerignore`。
* **公共分支别强推**：`rebase`只整理自己私有的未合并提交，对协作主干动`git push --force`会抹掉别人代码。
* **多阶段构建注意静态链接**：在Debian/glibc镜像里编出来的动态链接Go或Rust二进制，直接拷进基于musl的Alpine里会报找不到动态库；Go记得设`CGO_ENABLED=0`，Rust记得切`x86_64-unknown-linux-musl`。
* **课程范围边界**：第2章只覆盖单体镜像和端口映射，多容器网络、Volume数据持久化、`docker-compose`编排和监控运维不在这门课的主线里。

说到底，Git和Docker看着都是老工具，但真跑完才知道为什么课要把它们放最前面——一个管变更留痕和自动拦截，一个管环境隔离和随时复现。底座立住之后，下篇再写大模型接口、手写Agent Loop，以及AI怎么直接住进仓库和容器里干活。

---

## 5. 参考来源

* **课程主仓库（第1–2章讲义、lab与示例源码）**：`ai-engineering-course`（`01-git/lecture/git-guide.md`、`01-git/lab/README.md`、`02-docker/lecture/01-docker-foundation.md`、`02-docker/lecture/02-dockerfile.md`、`02-docker/lab/README.md`、`02-docker/lab/golang_sample/Dockerfile.single`与`Dockerfile.multi`）。
* **配套作业仓库（Docker练习）**：`ai-engineering-exercise`（`docker-exercises/README.md`、`02-python-flask-app/Dockerfile`、`03-rust-multistage-build/Dockerfile`）。