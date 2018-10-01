+++
title = "[译] Docker for devs：创建一个开发版镜像"
date = 2018-06-30T22:37:19+08:00
tags = ["Docker"]
categories = ["DevOps"]
draft = false
+++

# Docker for devs：创建一个开发版镜像

> 在腾讯云+翻译社的第一期活动中翻译了这篇文章，因为被官方采纳了，所以在云+社区的云计算专栏里也能看到[这篇译文](https://cloud.tencent.com/developer/article/1015260)。

_Docker for Devs 系列包括以下5篇文章，这是第二篇。_

- [容器化您的应用程序环境](http://www.summa.com/blog/docker-for-devs-creating-an-application-development-image)
- **创建一个开发版镜像（这篇文章）**
- [容器中的模块热重载和代码更新](http://www.summa.com/blog/docker-for-developers-hot-module-reloading-live-editing-in-containers)
- [组成多容器网络](http://www.summa.com/blog/docker-for-developers-composing-multi-container-networks)
- [与你的团队分享镜像](http://www.summa.com/blog/docker-for-developers-sharing-images-with-your-team)

在这个系列教程的第一部分中，我们为应用程序创建了一个的 基础 Docker 镜像并运行了该镜像的实例（称为容器）。我们也见证了 [Grayskull 的力量](https://www.youtube.com/watch?v=V8h8snfYidg)......我的意思是，Docker！_（PS: Grayskull 出自《He-Man and the Masters of the Universe》，上个世纪八十年代的一部动画）_

我们完成了所有典型应用程序的配置和运行，但不是从我们的本地主机，而是在一个容器内运行。接下来的 Docker for Developers 系列中，我们将试着配置一个可编辑的应用程序开发环境镜像。

## Docker for Developers：入门

我们在本教程的这一部分中的目标是生成一个代表我们应用程序开发版本的镜像，并为它配置一个（可运行）容器所需的必要组件，这样我们就能对文件系统进行更改并将其反映在容器中。

![Live editing in container](https://ask.qcloudimg.com/draft/1038922/jhxbf1vxxf.gif)

### 步骤1：创建一个开发版镜像

让我们在我们的应用程序的根目录中创建一个新的 Docker 镜像文件。以下是派生镜像的说明：

```bash
# dev.dockerfile

FROM express-prod-i

ENV NODE_ENV=development

CMD [“./initialize.sh”]
```

**我们做了什么？**

我们创建了一个新的docker镜像文件：

- 从我们的生产环境镜像 **express-prod-i** [获得](https://docs.docker.com/engine/reference/builder/#from)了基本镜像...
- ...并创建了值为 "development" 的容器本地 [ENV](https://docs.docker.com/engine/reference/builder/#env)变量 NODE\_ENV 。
- 最后，我们指定从 [WORKDIR](https://docs.docker.com/engine/reference/builder/#/workdir)运行名为 "initialized.sh" 的 bash shell 脚本。

### **步骤2：创建我们的初始化 Bash Shell 脚本**

我们不会在创建镜像时初始化应用程序，而是将其移至容器中。因此，应用程序启动步骤（例如，"npm install"）将在每次容器启动时执行。

- 在项目根目录下创建一个名为 "initialize.sh" 的文件

- 将以下内容粘贴到 "initialize.sh" 中

```bash
npm install
node bin/www
```

- 从终端/命令提示符进入项目根目录并运行以下命令，以使 bash shell 脚本可执行：

```bash
chmod +x initialize.sh
```

> 注意：请记住，这些容器正在基于 Linux 的环境中运行，因此运行 chmod system 命令会将指定文件的权限设置为可执行文件（本例中为initialize.sh）。

**我们做了什么？**

还记得吗，我们在基本的 **express-prod-i** 镜像中指定了运行 "npm install"  命令，该命令将安装 NPM 软件包作为容器的一部分。但在这里，我们：

- 创建一个文件，该文件将包含每次从此镜像生成的容器启动时要运行的命令。
- 设置权限，以便可以从容器内执行文件，并在容器启动时执行初始化步骤（如  "npm install"）。

### 步骤3：创建应用程序开发版镜像

现在，我们拥有了一个新的 Docker 镜像文件，我们已经准备好创建一个镜像了。

### **步骤3a：构建开发版镜像**

就像我们在[上一篇教程](https://dzone.com/articles/docker-for-devs-containerizing-your-application)中所做的那样，让我们创建一个新的镜像：

- 从终端/命令提示符进入我们的项目根目录。
- 在项目根目录的下执行以下命令：（PS：不要忘记最后的 **空格** 和 **"."** ）

```bash
docker build -t express-dev -i -f dev.dockerfile。
```

**我们做了什么？**

- 我们使用 Docker [build](https://docs.docker.com/engine/reference/commandline/build/) 命令创建了一个新的镜像。
- 需要注意的是，我们使用了一个新的标志 (-f) 代表文件，以指定我们希望它使用哪个
 Docker 文件。记住，默认文件名是 Dockerfile。
- 像之前一样用标志 (-t) 标记指定镜像名称，并为其命名 "express-dev-i"。

### **步骤3b：列出镜像**

运行 `docker images`，我们可以看到所有运行着的新旧镜像：

![镜像列表](https://ask.qcloudimg.com/draft/1038922/0srdiz6j63.png)

### **步骤4：生成并运行挂载数据卷（Volume）的容器**

我们现在有一个镜像，代表我们的应用程序的开发版本，它以我们的生产环境版本为基础。现在，我们想在运行那个容器的同时，挂载数据卷（Volume）。

一直以来，您可能一直在想如何编辑源代码，并且如果源代码驻留在容器中，它会反映在正在运行的容器中，对吗？那也是我们要完成的主要目标之一，不是吗？

我之前提到，镜像是一堆不同的只读分层文件系统。每层添加或替换下面的层。我也提到容器是镜像的一个运行实例。但事实上不止于此，容器为镜像的底层只读文件系统提供了一个读写层。

![Picture of Docker Container Write Layer over Image Read-only layer](https://ask.qcloudimg.com/draft/1038922/tgfpnm1ijx.png)

为了将这些只读层和读写层合并在一起，Docker 使用了 [Union File System](https://docs.docker.com/engine/understanding-docker/#/how-does-a-docker-image-work)（联合文件系统）。但通过容器的状态变化并不会反映在镜像中，任何文件更改都严格保存在容器中。这就带来了一个问题：当一个容器脱机时，在容器实例化的底层镜像中任何改变都不会被保存。

因此，为了持久化容器所做的更改（也有其他好处），Docker 开发了 [Volume](https://docs.docker.com/engine/tutorials/dockervolumes/)，通常被称作数据卷。简而言之，数据卷是存在于 Union File System 之外的目录或文件，通常位于主机文件系统上。

### 步骤4a：使用数据卷创建开发版镜像

现在我们有了一个表示应用程序开发版本的镜像，我们准备在主机上创建一个容器，其中包含指向应用程序源代码本地目录的
数据卷： 

> **重要提示：**如果你已经在容器外运行了应用程序（例如，`node bin/www`），与我们在 shell 脚本 _initialization.sh_ 中设置的命令相同，并且你的文件夹根目录下有一个本地的 node\_modules 目录，请现在删除他们。

1. 从终端/命令提示符进入 express 应用程序根目录。
2. `docker run –name express-dev-app -p 7000:3000 -v $(pwd):/var/app express-dev-i`

> **注意**：赋给 volume -v 标志的值被拆分为由 **(:)** 分隔的_主机目录_，后面跟着容器 WORKDIR 工作目录。这只与我们的设置有关，但指定的容器目录不一定是 WORKDIR 目录。  

**我们做了什么？**

1. 使用 Docker [RUN](https://docs.docker.com/engine/reference/run/) 命令，我们生成并启动了一个容器（一个镜像的实例）。
2. 并使用 -name 标志给我们的容器提供了 _"express-dev-app"_ 的名字。
3. 将我们的主机上7000的本地端口映射到我们使用 -p 标志公开的3000内部容器端口（与Dockerfile EXPOSE命令一起使用）。
4. 使用 volume -v 标志，我们在主机上挂载了一个数据卷，`$(pwd)` 代表主机上的“当前工作目录”到容器 _"/var/app"_ 中的一个目录（指定为 Dockerfile 中的 WORKDIR）。
5. 最后，指定要生成的镜像"express-dev-i" ，并将其作为容器运行

> **提示：**当容器被移除时，默认情况下不会删除数据卷。但是，您可以使用 docker remove（_rm）_指定 _-v_ 标志来删除关联卷： `docker rm -v [容器的名称或ID]`。

### 步骤4b：验证容器是否正在运行

如果一切按计划进行，您应该能在终端/命令提示符中看到 `npm install` 的结果和正在安装的 node modules 列表。我特意遗漏了这个被分开的 -d 标志，这样就可以观察到了。

我们可以通过运行 `docker ps`命令列出正在运行的容器，来验证是否有问题导致容器停止运行。如果没有列出，可以将
 ALL -a 标志添加到上述命令中，以显示所有容器，并查看是否有“express-dev-app”容器列出的退出错误。

### 步骤4c：检查容器的挂载信息

在我们继续之前，我们可以通过使用下面的 `INSPECT` 命令来查看有关装载量的信息，这个命令会向我们显示大量的容器信息：

```bash
docker inspect express-dev-app
```

​

![Mount Information](https://ask.qcloudimg.com/draft/1038922/pmzef0ocn7.png)

**我们做了什么？**

- 我们使用 Docker [INSPECT](https://docs.docker.com/engine/reference/commandline/inspect/) 命令查看有关容器信息的 JSON 格式输出。
- 它包含一个 "Mounts"  部分，列出了数据卷的来源。
- 它指向我们在本地主机上指定的项目根目录，以及指向容器中的 WORKDIR 目录的目的地。

### **步骤5：在本地编辑源代码**

这大概你一直在等待的时刻吧！我们将单刀直入，看看我们如何在本地进行源代码更改，并将其反映在容器中。

> **重要提示**：请务必查看第6步，了解关于安装的本地源代码和容器的一些重要提示，命令和解释。

#### 步骤5a：验证正在运行的 Express App

1. 在浏览器中进入http://localhost:7000

![浏览器运行结果](https://ask.qcloudimg.com/draft/1038922/cqvaknj98z.png)

1. 在根目录中，导航到 /views 目录并打开 index.jade 文件

1. 找到行

```dockerfile
p Welcome to ＃{title}
```

1. 修改为

```javascript
p Welcome to #{title} running in a container
```

    5. 回到浏览器中，刷新URL

![刷新后](https://ask.qcloudimg.com/draft/1038922/8fkc5alvrn.png)

​

**我们做了什么？**

我们不需要重建，甚至无需重新启动容器，就能看到我们对这个 express 应用的前端进行的简单而重要的改动被反映在了容器中。

### 步骤6：Node\_Modules 驻留本地

还记得吗，我们在创建最后一个容器之前删除了本地应用程序根目录中可能存在的任何 node\_modules 文件夹。但是，如果你再查看一下，会发现 node\_modules 文件夹依然存在。

这是因为托管运行 node.js 应用程序所需的更改（例如安装所有依赖的 node 模块），会通过我们挂载的卷在本地反映出来。

#### 步骤6a：与容器进行交互

我们可以通过连接到正在运行的容器来验证。在容器上打开一个 bash shell 并检查有关工作目录的信息。

我们没有以脱机模式启动容器，因此您需要停止正在运行的容器，并使用`docker start`命令重启，如上一个教程中所示。

或者您需要打开一个新的终端/命令提示符并通过：

1. `docker exec -it express-dev-app /bin/sh`
1. 在提示符下输入命令： `ls -l`

![Image of bash shell interaction on Docker container](https://ask.qcloudimg.com/draft/1038922/h1q2leak21.png)

**我们做了什么？**

- 我们使用 [EXEC](https://docs.docker.com/engine/reference/commandline/exec/) 命令连接正在运行的容器，使用 -it 标志提供交互式终端，并指定我们想要使用 /bin/sh 参数连接到bash shell。
- 你应该注意到，当我们连接到容器时，我们将自动连接到正在工作的 WORKDIR 目录。
- 我们使用 list 命令`ls -l`来显示目录内容实际上显示了本地卷挂载主机目录的内容。

## 结论

我们在 Docker for Developer 教程中完成的看起来很简单，但是非常高效。我们将我们的应用程序设置模块化，到一个包含应用程序必要设置的容器，同时保持对我们运行在容器中的应用程序源代码的控制。

本篇教程中，我们只是初步地在应用程序开发中应用 Docker 容器化技术。在下一个教程中，我们将抛开这些简单的例子，通过在容器中使用和运行支持热重载的通用（同构）React.js 应用程序，进行更深入的实践。

> 原文链接：https://dzone.com/articles/docker-for-devs-creating-a-developer-image
> 原文作者：Max McCarty