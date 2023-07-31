---
title: Go Module in 2019
date: 2019-06-18 15:18:39
tags:
---
2018年发布的Go 1.11包含了Go modules

如果go命令在GOPATH/src以外的目录运行，并且这个目录有一个go.mod文件，那么go是在module模式下运行。

目前可以通过$GO111MODULE环境变量来控制是否使用go module mode。这个环境变量，默认值是auto。

计划在2019年8月，发布Go 1.13，这个版本中$GO111MODULE会被默认设置为on。

目前有大量的工具、源码默认认为go的module放在GOPATH，所以如果迁移到go module mode的话，有大量的工具、源码要修改。

为了简化迁移的工作，golang团队提供了一个package，golang.org/x/tools/go/packages，这个package抽象了寻找和加载go source code的操作，所以新的代码可以使用这个package。

go的module是分散在不同的host上的，任何人都建立自己的host。这样的好处是，项目中引用private package很容易，但是这也带来一个问题，就是很难找到所有的module，或者在所有的module中查找一个module。

go新的module系统提供了一个go module index服务，利用这个服务，用户可以更容易发现新的module，更容易查找一个module。

go module index服务还会给每一个module提供一个go.sum文件，这个文件会列出一个module依赖的所有其他module的hash，这样go就可以利用go.sum来确保获得的其他module没有被修改过。

Node.js社区可以在考虑建立一个分布式的module registry，https://github.com/entropic-dev/entropic

参考
* https://blog.golang.org/modules2019