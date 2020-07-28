---
date: 2020-07-25 22:00:00
title: SBT设置国内源
categories:
- Scala
---

```
[repositories]
  local
  maven-huawei: https://repo.huaweicloud.com/repository/maven/
  ivy-huawei: https://repo.huaweicloud.com/repository/ivy/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
  typesafe: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
  sonatype-oss-releases
  maven-central
  sonatype-oss-snapshots
```

添加JVM参数如下

```
 # 配置该参数后才能使用REPO镜像
 -Dsbt.override.build.repos=true 
 # (可选)选择配置REPO配置文件路径
 -Dsbt.repository.config=<REPO_CONFIG_PATH>
```