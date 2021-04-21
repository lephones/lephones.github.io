title: 使用jenkins为android工程打包，支持多包名，改资源（优化方案）
date: 2020-09-04 15:32:15
category: android打包

tag: [android,jenkins]
---

# 需求

没有需求，自己看了一眼自己之前写的打包脚本，简直无法看下去。而且，产品经理的定制化需求越来越多，用shell脚本的可读性也越来越差，再加上里面一堆的sed命令，惨不忍睹。

改!!!

# 分析

gradle其实支持自定义参数，关于自定义参数的介绍，参考官方文档：https://docs.gradle.org/current/userguide/build_environment.html，简单说一下用到的：[Gradle properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties)：

<!-- more -->

有两种方式传给gradle，一种是-P。

```
gradlew clean -Pname=zhangsan
```

另一种是gradle.properties，我们可以在build.gradle文件所在的目录下创建一个gradle.properties文件，写上

```name=zhangsan
name=zhangsan
```

通过这两种方式，在gradle脚本中就可以直接使用这个变量名。

比如如下的gradle配置，其中包含了int型，String型，boolean型，基本能满足需求了。

```
        buildConfigField "boolean", "SHOW_INFO", "$SHOW_INFO"
        applicationId "$PACKAGE_NAME"

        versionCode Integer.valueOf(VERSION_CODE)
        versionName VERSION_NAME

        buildConfigField "String", "QQ_ID", "\"$qq_id\""
        buildConfigField "String", "WX_RELEASE_ID", "\"$wx_id\""
```

对应的gradle.properties文件

```
VERSION_NAME=1.2
VERSION_CODE=1110
PACKAGE_NAME=me.lefo.jenkins
SHOW_INFO=true
qq_id=1001
wx_id=1002
```



# 结合jenkins的参数化构建

通常用jenkins在构建的时候，都会自定义一些参数，比如上文的VERSION_NAME，VERSION_CODE。当然，我们还有个奇葩的需求，改包名。

jenkins在搭建的时候，构建那一步一定要选`invoke gradle script`，点开下面的高级选项，勾选`Pass all job parameters as Project properties`，勾选这一项会将jenkins参数化构建时的参数写到gradle中，还会替换掉gradle.properties中的默认值。其实就是通过-P把参数传到了gradle，-P传入的优先级高于properties文件。

**一定要保证参数名和gradle.properties文件中的名字一样！！！**

**一定要保证参数名和gradle.properties文件中的名字一样！！！**

**一定要保证参数名和gradle.properties文件中的名字一样！！！**

# 和包名绑定的配置

我们有个改包名的需求，有些配置化的值，是要跟着包名变动的，比如第三方平台的id。怎么办呢？这里的做法是，不同包名，建立不同的properties文件，执行时，使用cp -f命令，替换掉gradle.properties文件。

```
cp -f $PACKAGE_NAME.properties gradle.properties
```

像一些构建时jenkins传入的参数，可以不在`包名.properties`里出现，但是，默认的gradle.properties文件必须有这些值，在as run的时候，还是得在gradle.properties中有个默认值的。但jenkins选了`invoke gradle script`，这些参数会通过-P传进去。

# manifest文件

manifest文件中，也会配置一些值，可以通过placeHolder传进去。比如channel

```
defaultConfig {
	manifestPlaceholders = [label:"@string/app_name",
                                channel:"$CHANNEL"]
 }
```

在manifest中可以通过${channel}来使用，manifest中的authorities也可以通过这种方式来处理。



# 改应用名

我们有两个需求，一个是改用户界面显示的应用名，另一个是改APP内部显示的应用名，这两个有可能不一样。这里当然是通过sed改的，貌似也没有别的办法，上一段中的label也是用于这个功能的。我们的做法是把包含应用名关键字的strings.xml单独提取到一个value资源文件，然后通过sed统一修改。

```
sed -i 's/默认应用名/'"$APP_NAME"'/g' myflavor/res/values/stringfile.xml
```



# 完成

通过这波修改，之前的一坨sh文件中，就只剩下修改应用名的脚本了，其它的都通过gradle的环境变量来支持了。后续如果要加配置项，只需要在gradle.properties中加个默认项，如果是跟包名的，就放在包名.properties中，如果是jenkins配置的，就按对应的名称配在jenkins就OK。







