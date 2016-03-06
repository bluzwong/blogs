title: 闷声大发财！JitPack一键发布代码库
date: 2016-02-18 23:24:58
tags: android
---


## 0.中央代码库
中央代码库是个啥？谁说对了代码都给他！
中央代码库就是 build.gradle 里
```groovy
compile 'com.android.support:appcompat-v7:23.1.0'
```
去寻找下载代码的地方。较新版本 Android studio 中默认使用
```groovy
repositories {
    jcenter()
}
```
也就是Bintray Jcenter中央代码库。

## 1. 万恶的Jcenter
鄙人以前，一直用Jcenter作为repository。最近，库的owner变成了![](http://7xr14l.com1.z0.glb.clouddn.com/owner_bintray.png)
你问我资词不资词? 我说不资词。无法提交新版本，尝试提交Ownership request依然不行。

## 2. JitPack介绍
趁此机会，尝试一下最近很火的[JitPack](https://www.jitpack.io/)，果然如[一些人所说](http://www.dss886.com/android/2015/10/17/16-23/?utm_source=tuicool&utm_medium=referral)，Jcenter Maven这些代码库操作起来给人一种上世纪的感觉。Jitpack有多简单？首次进行简单配置，以后每次需要发布新版本只需要打个tag, push到[GitHub](https://github.com/)上，然后在Jitpack 上Loop up。根本不需要如Jcenter填写繁复的配置，还得publish，审核等等。

## 3. 发布纯Java jar到JitPack
### 3.1 首先准备好自己要发布的Java模块
这里要注意，JitPack据称是使用JDK1.8编译，给人一种已经钦定了的感觉。但是我怎么提交都只看到Java7，所以还是尽量使用Java7的语法以及标准库。
然后在该模块的 `gradle.build` 内，添加这些内容
```groovy
apply plugin: 'maven'

group = 'com.github.bluzwong'
task sourceJar(type: Jar) {
    from sourceSets.main.allSource
    classifier = 'sources'
}

task javadocJar(type: Jar, dependsOn: javadoc) {
    classifier = 'javadoc'
    from javadoc.destinationDir
}
```
> 上面这些可以直接复制去。当然啦，你们的决定权也是很重要的！
group要按照基本法，按照发布的法，去产生。

**group = '改成你自己的group'**
### 3.2 在命令行
```
gradlew install
```
![BUILD SUCCESSFUL](http://7xr14l.com1.z0.glb.clouddn.com/gradlew_install.png)

### 3.3 本地就OK了
现在 [Commit](http://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E8%AE%B0%E5%BD%95%E6%AF%8F%E6%AC%A1%E6%9B%B4%E6%96%B0%E5%88%B0%E4%BB%93%E5%BA%93#提交更新) 这个版本，并打上 [Tag](http://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E6%89%93%E6%A0%87%E7%AD%BE)
![COMMIT AND TAG](http://7xr14l.com1.z0.glb.clouddn.com/commit_tag.png)


将这笔要发布的 Tag [Push](http://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E8%BF%9C%E7%A8%8B%E4%BB%93%E5%BA%93%E7%9A%84%E4%BD%BF%E7%94%A8#推送到远程仓库) 到GitHub上。
在GitHub上看到这样就可以了
![PUSH DONE](http://7xr14l.com1.z0.glb.clouddn.com/push_done.png)
### 3.4 发布
打开 [JitPack](https://www.jitpack.io/) 填上代码库地址。
![jitpack0](http://7xr14l.com1.z0.glb.clouddn.com/jitpack0.png)
点击 `Look up` JitPack便会找到有Tag的提交，并且进行编译。待风火轮停下。
![jitpack1](http://7xr14l.com1.z0.glb.clouddn.com/jitpack1.png)

`Status` 上的 `Get it` 变成绿色就是好啦。如果不是绿色，就是编译失败，点 `Log` 图标看看问题在哪。

至此，发布Java jar全部完成。可以在项目中按如下方式依赖。
![jitpack2](http://7xr14l.com1.z0.glb.clouddn.com/jitpack2.png)


## 4. 发布aar到JitPack
### 4.1 aar
jar包中无法包含 android lib 的资源，故出现一种打包方式 `aar`。所以包含有 android sdk 内容的要打包成 `aar`。
### 4.2 修改build.gradle
在项目的 `build.gradle` 中增加

```groovy
classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'
```
*看起来是这样的*
```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.0.0-beta5'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

在需要发布的 android lib 模块的 `build.gradle` 中增加
```groovy
apply plugin: 'com.github.dcendents.android-maven'
group = 'com.github.bluzwong'
```
*看起来是这样的*
```groovy
apply plugin: 'com.android.library'
apply plugin: 'com.github.dcendents.android-maven'
group = 'com.github.bluzwong'

android { 
...
}
```
**当然 group还是要改成你自己的**

### 4.3 在命令行敲
```
gradlew install
```
BUILD 无误后，git commit && tag && push。
### 4.4 发布
回到 `JitPack` 点击 `Loop up` 会对这次的Tag编译。
![jitpack3](http://7xr14l.com1.z0.glb.clouddn.com/jitpack3.png)
### 4.5 使用
这次由于项目中有两个模块(还有上一部的Java jar)，所以依赖方式略有区别
![jitpack4](http://7xr14l.com1.z0.glb.clouddn.com/jitpack4.png)
点击 `Subproject` 可以切换成另一个模块的依赖

## 5. 总结
以后需要提交代码库新版本时
![flow](http://7xr14l.com1.z0.glb.clouddn.com/flow.png)

---
[示例](https://github.com/bluzwong/tryjitpack)
# 很惭愧，只做了一些微小的工作。

# 蟹蟹大家！
