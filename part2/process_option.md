# 应用上云的操作流程
## 2.1 Dockerfile编写
### 2.1.1基本指令

    ①FROM：FROM <name:tag>

解释：指定一个基础镜像，它一定是首个非注释指令，如果不指定tag，latest将会被指定为要使用的基础镜像版本。

    ②MAINTAINER：MAINTAINER <example@goodrain.com>
    
解释：用于指定镜像制作者的信息。

    ③RUN：RUN <command>

解释：在当前镜像中执行任意合法命令并提交执行结果，命令执行提交后，会自动执行Dockerfile中的下一个指令。
    
    ④ENV：ENV <key>=<value>
解释：用于为docker容器设置环境变量，用它设置的环境变量，可以使用docker inspect命令来查看，也可使用docker run –env \<key\>=\<value\>来修改环境变量。

    ⑤USER：USER <username>
解释：用于切换运行属主身份。默认使用的是root用户，非特殊情况，一般使用普通用户操作，因为root权限太大，使用上有安全风险。

    ⑥WORKDIR：WORKDIR <path>
解释：用于切换工作目录。默认工作目录是/，只有RUN能执行cd命令切换目录，而且还只作用与当下的RUN命令，每个RUN都是独立进行的。如果想让其他指令在指定的目录下执行，就得靠WORKDIR。WORKDIR指令的目录改变是持久的，不用每个指令前都使用一次WORKDIR。
    
    ⑦COPY/ADD：COPY/ADD <src> <dest>
解释：COPY和ADD作用相同，都是将文件从路径\<src\>复制添加到容器内部路径\<dest>。\<src>是相对于原文件夹的一个文件或目录，也可以是一个远程的url；\<dest>是目标容器中的绝对路径。但又有不同，COPY功能更纯粹，就是为了拷贝本地文件到镜像中；ADD支持添加本地的tar压缩包到容器中指定目录，压缩包会被自动解压为目录，也可以自动下载URL并拷贝到镜像。非特殊情况，推荐使用COPY。

    ⑧CMD/ENTRYPOINT：CMD/ENTRYPOINT ["ls", "-l"]/CMD/ENTRYPOINT ls -l
解释：用于指定容器创建时执行的默认命令，一个Dockerfile中只能有一个CMD指令，如果指定了多个，则只有最后一个生效。一般使用前一种语法，对于后一种语法，docker会自动加入“/bin/sh -c”到命令中，这样有可能导致意想不到的行为。一般使用只使用一种，如果两个同时使用，请确定他们的含义没有错误，一般来说，需要两个同时使用的情况只有ENTRYPOINT指定需要运行的binary，CMD给出运行的默认参数。

    ⑨VOLUME：VOLUME <path>
解释：用于创建一个可以从本地主机或其他容器挂载的挂载点，一般用来存放数据库和需要保存的数据等。

    ⑩EXPOSE：EXPOSE <port>
解释：用于指定在容器内部可开放的端口。注意，在容器运行时要加上容器内端口到外部宿主机器端口的映射。

    ⑪ONBUILD：ONBUILD <command>
解释：用于让指令延迟执行，延迟到下一个使用FROM的Dockerfile在建立镜像时执行，只限延迟一次。它一般使用在建立镜像时取得最新的源码（搭配RUN）与限定系统框架。

    ⑫ARG：ARG <key>=<value>
解释：用于指定临时变量，只在建立镜像时有效，建立完成后变量立刻失效消失，它是在Docker1.9版本新加入的指令。

    ⑬LABEL：LABEL <key>=<value>
解释：定义镜像的标签，并为其赋值。如定义镜像的拥有者owner为Jason，则使用LABEL owner=Jason。

### 2.1.2 编写Dockerfile进阶

**1.容器只运行单个应用**

从技术角度讲，你可以在Docker容器中运行多个进程。你可以将数据库，前端，后端，ssh，supervisor都运行在同一个Docker容器中。但是，这会让你非常痛苦：  
①非常长的构建时间(修改前端之后，整个后端也需要重新构建)
②非常大的镜像大小  
③多个应用的日志难以处理(不能直接使用stdout，否则多个应用的日志会混合到一起)  
④横向扩展时非常浪费资源(不同的应用需要运行的容器数并不相同)  
⑤僵尸进程问题 —— 需要选择合适的init进程  
因此，我建议大家为每个应用构建单独的Docker镜像，然后使用 Docker Compose 运行多个Docker容器。

**2.选择合适的基础镜像(alpine版本最好)**

一般，在包含必要功能前提下，使用的基础镜像越小越好。如，如果我们只需要运行node程序，则没必要选用通用的Ubuntu镜像，可以使用alpine版本的node镜像作为基础镜像。

**3.基础镜像的标签不要用latest**  

当镜像没有指定标签时，将默认使用latest 标签。因此， FROM ubuntu 指令等同于FROM ubuntu:latest。当时，当镜像更新时，latest标签会指向不同的镜像，这时构建镜像有可能失败。如果你的确需要使用最新版的基础镜像，可以使用latest标签，否则的话，最好指定确定的镜像标签。

**4.将多个RUN指令合并为一个**

Docker镜像是分层的，Dockerfile中的每个指令都会创建一个新的镜像层。镜像层将被缓存和复用，当Dockerfile的指令修改了，复制的文件变化了，或者构建镜像时指定的变量不同了，对应的镜像层缓存就会失效。某一层的镜像缓存失效之后，它之后的镜像层缓存都会失效。  
镜像层是不可变的，如果我们再某一层中添加一个文件，然后在下一层中删除它，则镜像中依然会包含该文件(只是这个文件在Docker容器中不可见了)。例如：

    RUN apt-get update && apt-get install -y nodejs


**5.每个RUN指令后删除多余文件**

假设我们更新了apt-get源，下载、解压并安装了一些软件包，它们都保存在/var/lib/apt/lists/目录中。但是，运行应用时Docker镜像中并不需要这些文件。我们最好将它们删除，因为它会使Docker镜像变大。例如，我们可以删除/var/lib/apt/lists/目录中的文件(它们是由apt-get update生成的)。

    FROM ubuntu:16.04
    RUN apt-get update \ 
  		&& apt-get install -y nodejs \
  		# added lines
  		&& rm -rf /var/lib/apt/lists/*
**6.在entrypoint脚本中使用exec**

不使用exec的话，我们则不能顺利地关闭容器，因为SIGTERM信号会被bash脚本进程吞没。exec命令启动的进程可以取代脚本进程，因此所有的信号都会正常工作。

**7.合理调整COPY与RUN的顺序**

我们应该把变化最少的部分放在Dockerfile的前面，这样可以充分利用镜像缓存。例如，在运行node应用中，需要下载npm模块，而源代码会经常变化，则每次构建镜像时都需要重新安装NPM模块，这显然不是我们希望看到的。因此我们可以先拷贝package.json，然后安装NPM模块，最后才拷贝其余的源代码。这样的话，即使源代码变化，也不需要重新安装NPM模块。
    
    FROM node:7-alpine
    WORKDIR /app
    COPY package.json /app
    RUN npm install
    COPY . /app

## 2.2 资源管理
### 2.2.1 存储卷
容器是无状态的，当容器删除，重新部署后，是镜像原始状态，不会保留之前所有的操作，如果想要保存某些配置或者数据，则需要配置“存储卷”，有两种方式添加存储卷。  
①通过界面创建  
如图1所示，点击“新建存储卷”后，存储卷名称任意，根据实际情况配置合适的存储卷容量。新建好存储卷后，供部署镜像时挂载到容器需要存储的目录下。  
![图1](/part2/figures/Figure1.png '图1')  
②通过导入YAML文件  
如图2所示，在功能菜单下，有“导入YAML/JSON”选项，点击“选择文件”，选择编辑好的存储卷的YAML文件，然后点击“创建”，会在“存储卷”菜单中找到通过YAML文件创建的相应的存储卷。  
![图2](/part2/figures/Figure2.png '图2')

### 2.2.2 配置文件处理
使用配置卷（ConfigMap）和密钥卷（Secret）实现配置共享。密钥卷与配置卷的区别是：密钥卷将配置卷中的value值使用base64进行了编码，使用decode解码密钥卷中的值，就可显现原值，存储卷相当于明文存储。

①配置卷  
同存储卷，也有两种方式创建配置卷，一种是通过页面新建配置卷，如图3所示，可以添加配置文件，也可以直接创建key/value值；另一种通过导入配置卷的YAML模板文件，直接创建配置卷。  
![图3](/part2/figures/Figure3.png '图3')
②密钥卷  
添加密钥卷与配置卷一样，步骤同上。

## 2.3 镜像制作
### 2.3.1 外网环境
直接在页面通过代码构建，如图4所示，使用GitHub构建必须先绑定相应的账号，从中选择代码进行构建。大多使用Git仓库的方式构建镜像，“构建名称”随意，“仓库地址”为构建项目的Git地址，“Dockerfile路径”为相对路径。
![图4](/part2/figures/Figure4.png '图4')  
### 2.3.2 内网环境
①使用本地代码仓库，必须为Git仓库，如提供的GitLab，具体方式同GitHub，先绑定GitLab。  
<!--②使用命令行制作好镜像后，上传到镜像仓库。具体命令在4.2节运维命令列出。-->

## 2.4 部署应用
构建好镜像后，可以通过3种方式部署应用。  
①构建好镜像后，在“镜像仓库”菜单下的“构建镜像”中，可查看构建好的镜像。如图5所示，选择“部署”按钮，“服务名称”任意，在下面有一个“添加卷”按钮，如图6所示，点击后，出现挂载密钥卷、配置卷和存储卷三个按钮，挂载2.4节创建好的存储卷、配置卷或密钥卷。基本配置完成后，即可点击“创建服务”，启动容器。  
如图7所示，在高级配置中，可配置弹性伸缩，配置POD数，即副本数量，POD最大数（最大为10）。  
其中，有“路由设置”，可以在部署镜像时进行基本配置，也可在部署镜像完成后，在“域名管理”中设置，此时，还可进行蓝绿部署和灰度发布，在2.8节详述。域名前缀可任意，配置好域名后，即可打开应用的页面。  
其中，也有“环境变量设置”，根据实际应用情况，添加环境变量，比如数据库的用户名和密码。  
其中，还有memory和cpu的配额限制（每个POD）。

②通过功能菜单下的导入YAML文件部署镜像。

③通过命令行启动容器。

![图5](/part2/figures/Figure5.png '图5')  
![图6](/part2/figures/Figure6.png '图6')  
![图7](/part2/figures/Figure7.png '图7') 

## 2.5 应用下线
在“部署镜像”菜单下，选择要删除应用的名称，如图8所示，删除名称为“abc2”的应用，点击后，进入“镜像部署详情”页面，如图9所示，点击右上角“删除”按钮，出现确认删除的弹窗，点击“确认”即可删除，实现应用下线。  
![图8](/part2/figures/Figure8.png '图8')  
![图9](/part2/figures/Figure9.png '图9')

## 2.6 回滚、蓝绿部署和灰度发布
### 2.6.1 回滚
当更改配置或镜像改变后，会自动重新部署镜像，每次变动，生成从1开始依此增加的应用版本，当想要变成原来版本时，可使用回滚。  
在“部署镜像”菜单下，点击应用名称，进入“镜像部署 详情”页面，在“部署记录”中，可查看不同版本。如图10所示，正在运行的为版本14，如果现在想回滚到版本13，点击版本号“#13”，然后再点击“Roll Back”按钮，出现如图11所示页面，有3个选项供选择，回滚后，可保存版本13的①副本数和选择器②部署策略③部署触发器，根据实际应用情况，进行选择。然后点击“Roll Back”按钮，实现回滚。  
![图10](/part2/figures/Figure10.png '图10')  
![图11](/part2/figures/Figure11.png '图11')  

### 2.6.2 蓝绿部署
如图12所示，在“域名管理”菜单下，点击“新建路由”按钮，进入“域名路由设置”页面，打开“流量分发”，使一个路由对应两个不同的服务，选择合适的流量分发权重，实现蓝绿部署。  
![图12](/part2/figures/Figure12.png '图12') 

### 2.6.3 灰度发布
类似于蓝绿部署，不同之处是，在灰度发布中，一个路由可对应3个及以上的不同服务，在“域名路由设置”页面，点击“添加”按钮，添加多个不同服务，如图13所示。其中，权重没有要求，计算方式是：百分比=各自权重/权重和。
![图13](/part2/figures/Figure13.png '图13')










