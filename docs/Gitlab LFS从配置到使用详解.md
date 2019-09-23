[TOC]

# Gitlab LFS从配置到使用详解

### 一. GitLab 对 Git LFS 支持需要满足以下要求(官网摘录)

- Git LFS is supported in GitLab starting with version 8.2. (gitlab版本需要 >= 8.2)
- Git LFS must be enabled under project settings (必须在项目设置中开启LFS)
- Users need to install Git LFS client version 1.0.1 and up （本地git lfs客户端版本 >= 1.0.1）

> 查看Gitlab版本号

      cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
    
> 当前Gitlab版本号10.2.4-ee

### 二. 安装Git LFS

> 最新版本
> [Release v2.8.0 · git-lfs/git-lfs · GitHub](https://github.com/git-lfs/git-lfs/releases/tag/v2.8.0)

```
wget https://github.com/git-lfs/git-lfs/releases/download/v2.8.0/git-lfs-linux-amd64-v2.8.0.tar.gz

tar -zxvf git-lfs-darwin-amd64-2.2.1.tar.gz

cd git-lfs-2.2.1

./install.sh
```

### 三. 配置Gitlab

1. **配置 GitLab 是否开启 Git LFS 以及修改默认 LFS 存储路径。**
  > 新版 GitLab 默认是开启 Git LFS 支持，默认存储路径为：{gitlab_rails['shared_path']}/lfs-objects。若我们想关闭 Git LFS 或者修改存储路径的话，可以通过下边方法修改：
  > 
  > 1、GitLab 以 Omnibus packages 混合包安装
  > 修改/etc/gitlab/gitlab.rb
  > 
    gitlab_rails['lfs_enabled'] = true | false
    #默认位置：`/var/opt/gitlab/gitlab-rails/shared/lfs-objects`
    gitlab_rails['lfs_storage_path'] = "/mnt/storage/lfs-objects"
    
  > 2、GitLab 以 source 源码安装
  > 修改config/gitlab.yml:
  > 
    lfs:
        enabled: false | true
        storage_path: /mnt/storage/lfs-objects

2. **GitLab 的 Project 设置开启 LFS**

> 在 GitLab 中设置项目的 LFS 开启|关闭：project -> Settings -> General ->Permissions -> Git Large File Storage: enabled/disabled



### 四. 常用 Git LFS 命令

- 查看当前使用 Git LFS 管理的匹配列表
> git lfs track

- 使用 Git LFS 管理指定的文件
> git lfs track "*.psd"

- 不再使用 Git LFS 管理指定的文件
> git lfs untrack "*.psd"

- 类似 git status，查看当前 Git LFS 对象的状态
> git lfs status

- 枚举目前所有被 Git LFS 管理的具体文件
> git lfs ls-files

- 检查当前所用 Git LFS 的版本
> git lfs version
- 针对使用了 LFS 的仓库进行了特别优化的 clone 命令，显著提升获取LFS 对象的速度，接受和 git clone 一样的参数。

> git lfs clone https://xxxxxx.git

>git lfs clone 通过合并获取 LFS 对象的请求，减少了 LFS API 的调用，并行化 LFS 对象的下载，从而达到显著的速度提升。git lfs clone 命令同样也兼容没有使用 LFS 的仓库。即无论要克隆的仓库是否使用 LFS，都可以使用 git lfs clone 命令来进行克隆。
目前最新版本的 git clone 已经能够提供与 git lfs clone 一致的性能，因此自 Git LFS 2.3.0 版本起，git lfs clone 已不再推荐使用。

- 只获取仓库本身，而不获取任何 LFS 对象

如果自己的相关工作不涉及到被 Git LFS 所管理的文件的话，可以选择只获取 Git 仓库自身的内容，而完全跳过 LFS 对象的获取。

> GIT_LFS_SKIP_SMUDGE=1 git clone https://xxxxxx.git
  或
git -c filter.lfs.smudge= -c filter.lfs.required=false clone https://xxxxxx.git

注：GIT_LFS_SKIP_SMUDGE=1 及 git -c filter.lfs.smudge= -c filter.lfs.required=false 同样使用于其他 git 命令，如 checkout, reset 等。

- 获取当前 commit 下包含的 LFS 对象的当前版本
如果起初获取代码时，没有一并获取 LFS 对象，而随后又需要这些被 LFS 管理的文件时，可以单独执行 LFS 命令来获取并签出 LFS 对象：
> git lfs fetch
  git lfs checkout
  或
  git lfs pull

### 五. 常见问题
1. **Git LFS 对象在本地仓库的存放位置？**

> 通过 Git LFS 所管理的对象实际在本地的存储位置是在 .git/lfs/objects 目录下，该目录根据对象的 sha256 值来组织。
  作为对比，Git 自身所管理的对象则是存储在 .git/objects 目录下，根据 commit, tree, blob, tag 的 sha1 值来组织。

2. **已经使用 git lfs track somefile 追踪了某个文件，但该文件并未以 LFS 存储。**

> 如果被 LFS 追踪管理的文件的大小为 0 的话，则该文件不会以 LFS 的形式存储起来。
  只有当一个文件至少有 1 个字节时，其才会以 LFS 的形式存储。

3. **执行 git lfs fetch 或 git lfs pull 时报错**

batch request: exit status 255: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
如果在克隆仓库时使用了 SSH 协议，而本地的 SSH 私钥又有密码保护，那么向服务器获取文件时就会报错，因为目前 Git LFS 不会向用户请求密码，从而导致认证失败。

> 解决办法是使用 ssh-add 命令，预先加载好本地的 SSH 私钥，从而使得 Git LFS 能够访问到私钥。

4. **使用 Git LFS 时，报错缺失协议 (missing protocol)或协议不支持 (unsupported protocol scheme)。**

出现这种错误通常有两种原因：

**第一种**是克隆仓库时使用的地址没有包含用户名，如克隆时使用了类似 git clone github.com:user/repo.git 的命令，从而导致 Git LFS 错误地将服务器地址当作协议名来看待，而报出协议不支持的错误：

    [z@zzz.buzz lfs-test]$ git push
    Git LFS: (0 of 1 files) 0 B / 1 B
    Post git.zzz.buzz:z/lfs-test.git/info/lfs: unsupported protocol scheme "git.zzz.buzz"
    Post git.zzz.buzz:z/lfs-test.git/info/lfs: unsupported protocol scheme "git.zzz.buzz"
    error: failed to push some refs to 'git.zzz.buzz:z/lfs-test.git'
> 解决办法就是在仓库地址中加上 git 用户名，如：

    git remote set-url origin git@github.com:user/repo.git
    
**第二种**原因则是克隆仓库时使用的是 SSH 协议，而使用的 Git 服务器不支持在 SSH 下使用 Git LFS（如低于 8.12 版本的 GitLab）
> 解决办法为将克隆仓库时使用的 SSH 协议换为 HTTPS 协议即可。
如原先 origin 设置为 git@github.com:user/repo.git，则可以运行如下命令：

    git remote set-url origin https://xxxxxx.git
> 随后再执行 Git LFS 相关的命令。

或使用 HTTPS 协议重新克隆仓库：

    git clone https://xxxxxx.git
    
### 六.测试案例(新建仓库使用LFS)

我们先分别提交稍大一些的文件到各个项目中

    $ git clone http://xxxxxx/demo1.git
    $ cd demo1
    $ cp ~/Downloads/soft/apache-tomcat-8.0.36.zip ./
    $ git add .
    $ git commit -m "test no lfs"
    $ git push origin master

    $ git clone http://xxxxxx/demo2.git
    $ cd demo2
    $ cp ~/Downloads/soft/apache-tomcat-8.0.36.zip ./
    $ git lfs track "*.zip"  #设置存储到 LFS 的文件扩展名，这里我设置 .zip 后缀格式的文件
    $ cat .gitattributes  #自动生成的文件，需一并提交到 Git，否则 Clone 项目的时候 Git LFS 不起作用
      * .zip filter=lfs diff=lfs merge=lfs -text
    $ git add .
    $ git commit -m "test with lfs"
    $ git push origin master

> 注意：我们对比下使用 LFS 和不使用 LFS 的项目操作，只需要在想加入的大文件时，增加文件后缀，执行git lfs track "*.zip"·一条语句即可，并未产生额外的 Git 指令，还是很容易上手的。

然后，我们再分别 Clone 各个项目

    $ git clone http://xxxxxx/demo1.git
    Cloning into 'demo1'...
    remote: Counting objects: 6, done.
    remote: Compressing objects: 100% (4/4), done.
    remote: Total 6 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (6/6), done.

    $ git clone http://xxxxxx/demo2.git
    Cloning into 'demo2'...
    remote: Counting objects: 7, done.
    remote: Compressing objects: 100% (4/4), done.
    remote: Total 7 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (7/7), done.
    Downloading apache-tomcat-8.0.36.zip (9.9 MB)
    
或者

    $ git lfs clone http://xxxxxx/demo2.git
    Cloning into 'demo2'...
    remote: Counting objects: 7, done.
    remote: Compressing objects: 100% (4/4), done.
    remote: Total 7 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (7/7), done.
    Git LFS: (1 of 1 files) 9.40 MB / 9.40 MB
    
> 注意：
这里我们可以看出，使用 LFS 的项目，Clone 时会提示 Downloading … 或者 Git LFS … ，当 Push 的文件更大一些的时候，我们会发现使用 LFS 的项目复制和提取文件会更快一些。
这里可使用git clone ...或者使用git lfs clone ...即指定该项目使用 lfs 均可，具体 git lfs 其他命令，可参考git lfs help命令。
开启 LFS 的项目，当 Push 大文件之后，在 GitLab Web 页面上是删除不了的，需要通过接口删除该文件。 


### 七.已有仓库使用LFS

> 已有仓库需要经过转换才能使用LFS

转换工具,主要有两种
- [git-lfs-migrate](https://github.com/bozaro/git-lfs-migrate)
- [bfg-repo-cleaner](https://github.com/rtyley/bfg-repo-cleaner)
> git-lfs-migrate 会生成转换完成的仓库，在本文中选择使用这个工具。
> BFG 与 git-lfs-migrate 都是在原有仓库上操作

命令参数说明:

    java -jar git-lfs-migrate.jar
    -s color-and-play-2-lfs.git //镜像
    -d color-and-play-2-lfs_converted.git //转换后的镜像
    -g https://xxxxxx.git //远程仓库地址
    --write-threads 64 //开启的转换线程数
    "*.png" "*.jpg" "*.prefab" "*.unity" "*.wav" "*.mp4" "*.ogg" "*.fbx"


> 添加-g参数转换出现异常时,可尝试以下方式
  在本地创建所有LFS文件，然后手动将它们推到LFS服务器。

    git clone --mirror https://xxxxxx.git
    java -jar git-lfs-migrate.jar \ -s repo.git \ -d repo-converted.git \ "*.psd"
    cd repo-converted.git
    git fsck
    git remote add origin https://xxxxxx/repo-converted.git
    git push origin master # (or git push --all origin)
    git lfs push --all origin(or git lfs push origin master)

执行:

     java -jar F:\git-lfs-migrate\git-lfs-migrate.jar \ -s color-and-play-2-lfs.git \ -d color-and-play-2-lfs_converted.git --write-threads 16 "*.png" "*.jpg" "*.wav" "*.mp4" "*.ogg"
                

> 可能出现的异常(文件过大导致OutOfMemory):

    [main] INFO git.lfs.migrate.Main -   processed: 63216/64111
    Exception in thread "main" org.eclipse.jgit.errors.LargeObjectException$OutOfMemory: Out of memory loading unknown object
            at ......
    Caused by: java.lang.OutOfMemoryError: Java heap space
            at ......
            ... 5 more
 
 
> 解决办法: 拆分pack文件

    git repack --max-pack-size=500M -a

> 必须加上-a参数才能强制执行,否则会出现`Nothing new to pack.`官方文档并没有说明清楚这个参数