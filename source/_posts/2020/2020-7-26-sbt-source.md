---
date: 2020-07-25 22:00:00
title: SBT设置国内源
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
