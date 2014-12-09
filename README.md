repox
=====

A fast sbt/gradle/lein/ivy/maven repository proxy

## Repox是什么
Repox的主要目标是改善sbt解决依赖的速度，但由于它的服务方式与url无关，因此也支持ivy, gradle, maven, leiningen客户端，可以作为nexus/artifactory的替代品来搭建私服。由于它的服务方式与url格式无关，

## 为什么需要Repox
* 每次sbt项目的依赖发生变更，或是sbt版本升级时，sbt会先resolve很长时间，然后download很长时间，甚至有时显示downloading，但你发现事实上没有网络流量产生，sbt假死。
* 为了对已经下载过的依赖进行缓存，保证后续请求能够快速完成，你安装了nexus，并把所有sbt可能用到的仓库以及国内的代理仓库（如oschina等）都设置为nexus的上游，却发现它对于sbt更新依赖慢的问题并没有什么改善。
* 你试图安装typesafe官方使用的Artifactory，却发现其开源版本不支持sbt所使用的自定义url格式。

## sbt为什么慢
* sbt默认所使用的仓库都在国外。
* 从某一个版本起，sbt默认使用https协议访问仓库。由于众所周知的原因，连往国外的https连接不稳定，而sbt对这种网络连接的不稳定没有相应的发现和重试机制。
* sbt对依赖的请求分为四个步骤：
  1. resolve : 向所有仓库（内置的如typesafe repo, central maven等，以及build中添加的resolvers）逐个发送HEAD请求询问是否收藏有此文件
  2. download : 向收藏了此文件的仓库发送Get请求下载文件
  3. resolve checksum : 向同一仓库发送HEAD请求询问是否有此文件的sha1 checksum
  4. download checksum : 如果有，则下载 sha1 文件并保证从原文件计算出的sha1与checksum文件的内容一致
* 如果没有组织内私服，那么只在每个开发者的本地~/.ivy2/cache目录下有依赖缓存，无法在多个开发者之间共享。

## 为什么nexus私服对sbt没有帮助
* nexus是一个代理而不是镜像
* sbt的resolve使用的是HEAD请求，即使是已经缓存过的文件，nexus对HEAD请求每次都要再到上游仓库去重新逐个询问。
* nexus中的多个上游仓库的次序是设定的。对于尚未缓存的文件，在download环节nexus不会根据HEAD请求的结果剔除掉不包含此文件的仓库。

## Repox的理念
* 代理所有流量
    这样可以在所有环节进行优化。
* 要么快速完成，要么快速失败
    如果上游仓库中有请求的文件，尽量选取最快的仓库下载。
    下载过程中如果发现当前连往上游仓库的网络连接长久没有获得数据，向sbt返回404，让用户重试，而不是永久等待。
* 与url的格式无关
    因此除了改善sbt解决依赖的速度，Repox 还可以为ivy, gradle, maven, leiningen客户端服务。
* 做代理该做的事
    对于已经下载过的文件，保证resolve立即成功。

## Repox的策略
* 对已下载的文件
    无论是HEAD请求还是GET都立即响应。
* 对首次请求的文件
    HEAD请求：同时向所有上游仓库发出询问，取最早200响应的头信息返回给sbt
    GET请求：
        首先根据HEAD请求的结果对上游仓库进行过滤，HEAD返回非200的仓库被排除.
        上游仓库被指定不同的优先级，向同一优先级的所有仓库同时发送GET请求，选取第一个返回200的仓库下载，其它仓库的连接被终止。
        当某一优先级的所有仓库均失败（返回非200或超时）时，自动进入下一优先级。
        如果所有仓库都失败，则404.
更多的细节请参阅其它Wiki页。

## Repox的优势
* 轻
核心代码仅十几个文件，每个文件不超过200行。约一个周末的阅读量。
* 全异步非阻塞
undertow + akka + async-http-client
* 提供了web配置界面来根据用户的网络状况对仓库、代理、参数等进行微调
* 实用
Repox是由我们的团队自己的需求驱动的，它支持着我们的日常开发任务。

## 推荐的使用方式
1. 作为组织内私服运行
这是最推荐的使用方式，这样组织内有一个开发者完成一个项目的依赖更新后，其它开发者能够飞速更新。

2. 开发者本机运行
repox的部署和运行都非常简单，在本机运行也能显著提高生产力。
