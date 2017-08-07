# Dockerfiles最佳实践

## 通用准则

* 容器应该是短暂的
* 使用.dockerignore文件
* 避免安装不必要的包
* 每个容器应该只有一个关注点
* 尽量减少层的使用
* 含参数较多时排序
* 建立缓存  
	 * 从cache中已存在的父镜像开始比较，下一个指令将与所有由该父镜像派生的镜像进行比较，以查看其中是否使用完全相同的指令构建。如果没有，缓存无效 。
	 * 一般Dockerfile中指令与其中一个子镜像比较即可。某些情况下，需要更多的检查和解释。
	 * 对于ADD和copy指令，将检查镜像中每个文件的内容，并计算其校验和。这些校验和没有考虑到文件最后修改和访问时间。在缓存查找期间，校验和将与已存在的镜像校验和进行对比。如果文件中有任何带动，则缓存无效。
	 * 处理ADD和copy指令，cache查找不会检查容器中的文件来确定cache匹配。

eg：
<pre>
RUN apt-get update&& apt-get install -y \
  brz \
  cvs \
  mercurial\
  subversion
</pre>

## Dockerfile指令

### From指令

用法：    
`FROM <image> [AS<NAME>]`    
`FROM <image> [:<tag>] [AS<NAME>]`    
`FROM <image> [@<digest>] [AS<NAME>]`

* 只有`ARG`可出现在FROM之前
* 一个Dockerfile可多次出现FROM,以创建多个镜像或一个镜像依赖
* 可以通过添加AS NAME，建立一个新的镜像层，以备后用
* `ARG`用于声明变量，处于FROM之前，处于建立镜像之前，所以只有FROM之后的指令才能使用该变量

eg：
<pre>
ARG CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD /code/run-app

FROM extras:${CODE_VERSION}
CMD /code/run-extras
</pre>

<pre>
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
</pre>

### LABEL指令

对象标签，应用于Images，contains，local daemons，volumes，networks，Swarm node，Swarm Service。

用法：
`LABEL <key>=<value> <key>=<value>.....`

eg1：设置一个或多个标签
<pre>
LABEL "com.example.vendor"="ACME Incorporated"
LABEL "com.example.label-with-value"="fool"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-value can span multiple lines"
</pre>

eg2：一行设置多个标签（推荐）
<pre>
LABEL multi.label1="value1" multi.label2="value2" other="value3"
</pre>

eg3：使用`\`一次设置多个标签（不推荐）
<pre>
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
</pre>

### RUN指令：

尽量将长的RUN声明，使用`\`划分为小的段  
避免使用`RUN apt-get upgrade`或`dist-upgrade`

RUN命令会在当前镜像顶部的新层执行任何命令，并提交结果。提交的镜像将会在Dockerfile进一步使用

分层运行指令和生成提交符合Docker核心概念，这使得提交成本很低，并且可以从镜像的历史记录中创建容器 

用法：  
`RUN <command>`(shell 格式，默认由/bin/sh -c)
`RUN ["executable", "param1", "param2"]`(exec form)

eg1:使用`\`将单个`RUN`指令分为两行
<pre>
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
</pre>

eg2:使用一行运行单个`RUN`指令
<pre>
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
</pre>

eg3：使用exec格式运行上述命令
<pre>
RUN ["/bin/bash", "-c", "echo hello"]
</pre>

eg4:`apt-get`问题,通常将`RUN apt-get update`和`apt-get install`放在同一个`RUN`声明中，可以避免缓存问题：
<pre>
	RUN apt-get update && apt-get install -y \
	    package-bar \
	    package-baz \
	    package-foo
</pre>

eg5:避免缓存问题推荐的方法：  
    每个包占一行，可以防止包重复的错误  
    最后清除apt的缓存，它不在镜像的某一层。
<pre>
    RUN apt-get update && apt-get install -y \
        aufs-tools \
        automake \
        build-essential \
        curl \
        dpkg-sig \
        libcap-dev \
        ruby1.9.1 \
        s3cmd=1.1.*
    && rm -rf /var/lib/apt/list/*
</pre>

eg6：使用管道`|`
<pre>
    RUN wget -O - https://some.site | wc -l > /number
</pre>
Docker使用`/bin/sh -c`解释器来执行上述命令  





