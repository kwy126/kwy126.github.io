
### 创建github用户





这部分很简单，我就不做过多的介绍。



### 创建远程仓库

在创建github用户之后，需要创建远程仓库，比如我创建了kwy.gtihub.io仓库，如果我们想访问自己想要的域名，我们就需要注册域名。

### 绑定固定域名

目前有两个大厂提供域名申请，[阿里万维网](https://wanwang.aliyun.com/)，[腾讯域名](https://dnspod.qcloud.com/),我在腾讯域名注册一个帐号，不过实名认证和域名解析需要1天时间。

### 创建本地git库

```

gem install jekyll bundler

jekyll new kwy126.github.io

cd kwy126.github.io

bundle exec jekyll serve

```

注意：不能直接使用mkdir kwy126.github.io，mkdir指令后面不能接分隔符

本地如果能访问127.0.0.1:4000说明keyll

### 初始化git库并提交到远程仓库

```

git init kwy126.github.io

cd kwy126.github.io

git commit -s -m "initialized"

git add index.md

git add Gemfile.lock

git add Gemfile

git add _config.yml

git add _posts

git add CNAME

git commit -s -m "initialized"

```

注意：

1.CNAME文件的作用是实现kwy126.github.io到域名(kwy126.cn)的关键

2._posts文件下存放的是我们编写的markdown格式文件，即我们要发布的文章



#### 下面介绍如何将本地仓库关联到远程github仓库

```

git remote add origin https://github.com/kwy126/kwy126.github.io.git

git pull origin master ##验证下是否关联成功

git push remote --all

git push -u origin master

```



### 参考连接

1、[jekyll快速上手](https://jekyllrb.com/docs/quickstart/)

2、[github pages 和jekyll上手](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)

3、[本地代码上传到github](http://blog.csdn.net/hanhailong726188/article/details/46738929)
