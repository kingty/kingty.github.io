---
title: Hexo&Github Blog 搭建
date: 2018-06-05 16:39:23
tags: 其他
---

今天用`Github Pages` 和 `Hexo` 来搭建了一个静态的博客。好处就是不需要数据库也不需要服务器，完全交由`Github`托管就可以拥有自己的博客。下面我记录了一下我自己的搭建的过程。
# Contex
- linux Cenos 6.5,64位
- Hexo 3.7.1

<!--more-->
# Github
因为所有的内容都是`Github`托管的，所以你首先要有一个`Github`账号。然后在创建一个`Repository`,这个`Repository`必须要满足一个规则才能生成`Pages`.详细内容可以查阅官方文档 [Github Pages](https://pages.github.com/)

简单来说就是创建一个名字为 `username.github.io`的库，其中`username`为你自己的`Github`的用户名。

创建成功之后，把项目clone到本地。我机器本身还没有`Git`所以我先安装`Git`

``` 
yum info git
yum install -y git

```

然后我还需要添加这台机器对github的ssh权限，这样才能访问github。

```
#生成key
ssh-keygen -t rsa -C "xxxx@gmail.com"

cd .ssh/
cat id_rsa.pub 
# 得到这个key后把它添加进你的github key里面

```
完成之后我们把刚才创建的`Repository`拷贝到本地。

```
git clone git@github.com:username/username.github.io.git

```
拷贝下来之后，我们在里面写一个我们的首页,并提交到我们的`Repository`

```
cd kingty.github.io

echo "Hello World" > index.html

git add --all

git commit -m "Initial commit"

git push -u origin master

```

到这里，你到浏览器输入`username.github.io`应该就可以访问到你刚才的html页面。其实到这里就相当于你的博客已经搭建完成了。但是，我们自己去写静态页面肯定是一个繁重的工作。因此我们就有了下面要介绍使用的工具`Hexo`

# Hexo

## 安装

`Hexo`是一个快速、简洁且高效的博客框架。`Hexo` 使用 `Markdown`（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。并且可以帮助我们直接部署到`github.io`也就是你上面创建的`Repository `上面ï¼这样我们就可以只需要写`Markdown`文件就完成了博客部署。

`Hexo`是有`node` 构造所以我需要先在机器上安装`node`.

```
rpm -ivh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-remi

rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm

rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-remi

yum -y install nodejs npm --enablerepo=epel

```

安装好`node`之后我们就可以用`npm`安装`hexo`了

```
npm install -g hexo

```
在安装`hexo`的时候可能会遇到问题：

```
npm ERR! Error: CERT_UNTRUSTED
npm ERR!    at SecurePair.<anonymous> (tls.js:1430:32)
npm ERR!    at SecurePair.emit (events.js:92:17)
npm ERR!    at SecurePair.maybeInitFinished (tls.js:1029:10)
npm ERR!    at CleartextStream.read [as _read] (tls.js:521:13)
npm ERR!    at CleartextStream.Readable.read (_stream_readable.js:341:10)
npm ERR!    at EncryptedStream.write [as _write] (tls.js:418:25)
npm ERR!    at doWrite (_stream_writable.js:226:10)
npm ERR!    at writeOrBuffer (_stream_writable.js:216:5)
npm ERR!    at EncryptedStream.Writable.write (_stream_writable.js:183:11)
npm ERR!    at write (_stream_readable.js:602:24)
npm ERR! If you need help, you may report this log at:
npm ERR!    <http://github.com/isaacs/npm/issues>
npm ERR! or email it to:
npm ERR!    <npm-@googlegroups.com>
npm ERR! System Linux 2.6.32-696.18.7.el6.x86_64
npm ERR! command "node" "/usr/bin/npm" "install" "-g" "hexo"
npm ERR! cwd /root/github/kingty.github.io
npm ERR! node -v v0.10.48
npm ERR! npm -v 1.3.6
npm ERR! 
npm ERR! Additional logging details can be found in:
npm ERR!    /root/github/kingty.github.io/npm-debug.log
npm ERR! not ok code 0

```
这是因为npm用https导致的，解决办法是：

```
npm config set strict-ssl false

```

然后应该就可以正常安装好了。完成之后我们需要初始化`Hexo`,**首先你还是要在刚才的`username.github.io`这个目录下，你现在应该在master分支，这时候你需要创建另外一个分支`hexo`**

```
git checkout -b hexo

```
然后在这个分支上初始化

```
hexo init

```

初始化的过程可能你会遇到问题如下：

```
/usr/lib/node_modules/hexo/node_modules/hexo-cli/lib/hexo.js:13
class HexoNotFoundError extends Error {}
^^^^^
SyntaxError: Unexpected reserved word
    at Module._compile (module.js:439:25)
    at Object.Module._extensions..js (module.js:474:10)
    at Module.load (module.js:356:32)
    at Function.Module._load (module.js:312:12)
    at Module.require (module.js:364:17)
    at require (module.js:380:17)
    at Object.<anonymous> (/usr/lib/node_modules/hexo/bin/hexo:5:1)
    at Module._compile (module.js:456:26)
    at Object.Module._extensions..js (module.js:474:10)
    at Module.load (module.js:356:32)
    
```
原因是`node`的版本过低，需要update 一下`node`到最新版

```
#安装更新的一个软件
npm install n -g

#更新到最新stable版本
n stable

#重新初始化
hexo init

```
这个时候你可能还ä¼遇到问题如下：

```
FATAL ~/github/kingty.github.io not empty, please run `hexo init` on an empty folder and then copy your files into it
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Error: target not empty
    at Context.initConsole (/usr/lib/node_modules/hexo/node_modules/hexo-cli/lib/console/init.js:30:27)
    at Context.tryCatcher (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/util.js:16:23)
    at Context.<anonymous> (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/method.js:15:34)
    at /usr/lib/node_modules/hexo/node_modules/hexo-cli/lib/context.js:44:9
    at Promise._execute (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/debuggability.js:303:9)
    at Promise._resolveFromExecutor (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:483:18)
    at new Promise (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:79:10)
    at Context.call (/usr/lib/node_modules/hexo/node_modules/hexo-cli/lib/context.js:40:10)
    at /usr/lib/node_modules/hexo/node_modules/hexo-cli/lib/hexo.js:68:17
    at tryCatcher (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:512:31)
    at Promise._settlePromise (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:569:18)
    at Promise._settlePromise0 (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:638:18)
    at Promise._resolveCallback (/usr/lib/node_modules/hexo/node_modules/bluebird/js/release/promise.js:432:57)

```

这是因为`hexo`的初始化必须在一个空目录下，包括隐藏的目录。你的文件下应该会有一个`.git`å你刚刚创建的`index.html`，你需要暂时把他们移动到另外一个目录，初始化完成之后再移动回来。

```
#移动出去
mv index.html ../temp
mv .git ../temp

#初始化
hexo init

```
这时候应该可以初始化成功了，**然后用相同的方法移动回来，才能进行下一步**安装一下

```
npm install

```
这个时候应该hexo就安装完成了。
然后你可以把这些东西push到你的hexo分支上去。

```
git add .
git commit -m "xxx"
git push -u origin hexo

```

## 配置

安装完成后，我们需要配置一些我们要生成的信息，还有就是我们要部署的地址。在当前文件夹下，你会看到一个文件叫`_config.yml`

```
vi _config.yml

```
里面会有一些信息，例如title,author都修改成你自己的就好了。注意里面有一项是`deploy`,需要修改成你的repo地址和分支，表示生成生个之后会push到你的repo的master分支上

```
deploy:
  type: git
  repo: git@github.com:username/username.github.io.git
  branch: master

```

其余的具体配置请参考[Hexo 配置](https://hexo.io/docs/configuration.html)

## 文章

配置完成后，我们尝试一下写一篇blog

```
hexo new "test"

```
这时候就会在 `source/_post`文件下生成一个 `test.md`的文件。

```
cd source/_post

vim test.md


```

在里面写一点内容，然后生成。

```

hexo generate -d

```

这时候就会生成静态页面并自动push到你的master分支，在浏览器访问`username.github.io`就可以看到刚才生成的blog主页里会有这篇文章了。




到这里，我们的博客就搭建好了，当然，你还可以为博客设置你喜欢的主题，做一些自己的定制等等。慢慢去摸索吧。












