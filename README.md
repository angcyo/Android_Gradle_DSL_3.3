# Android_Gradle_DSL_3.3
gradle android dsl 3.3 源码分析

> 由于`Gradle`的源码过大 仓库不上传`src`

> 故. 请使用本地的`gradle`源码文件夹`src`, 复制粘贴到对应版本的目录下即可.

# Gradle 源码所在路径
**MAC**
```java
/Users/用户名/.gradle/wrapper/dists
```
**Win**
```java
C:/Users/用户名/.gradle/wrapper/dists
```

`dists`目录下, 会记录本机所有已经下载好的版本.
比如我的:
```java
gradle-2.10-all   gradle-2.2.1-all  gradle-3.3-all    gradle-4.4-all    gradle-4.6-all
gradle-2.14.1-all gradle-2.4-all    gradle-4.1-all    gradle-4.4-bin
```
打开对应目录, 会有一个`16进制文件名`的路径, 继续打开, 知道找到 `gradle-4.6` 这样的文件夹. 打开, 里面的 `src` 目录, 就是对应的源码. 


# Android Build Gradle插件 源码所在路径
```
    classpath 'com.android.tools.build:gradle:3.2.1'
```
这样代码就是Android构建插件的核心.
```
只有使用的:
apply plugin: 'com.android.application'
apply plugin: 'com.android.library'
这2个插件, 都是基于上面那个实现的.
```

**MAC**
```java
/Users/用户/.gradle/caches/modules-2/files-2.1/com.android.tools.build
```
**Win**
```java
C:/Users/用户/.gradle/caches/modules-2/files-2.1/com.android.tools.build
```

还有可能出现在:
**MAC**
```java
‎⁨硬盘⁩ ▸ ⁨应用程序⁩ ▸ ⁨Android Studio.app⁩ ▸ ⁨Contents⁩ ▸ ⁨gradle⁩ ▸ ⁨m2repository⁩ ▸ ⁨com⁩ ▸ ⁨android⁩ ▸ ⁨tools⁩ ▸ ⁨build⁩ ▸ ⁨gradle⁩ ▸ ⁨3.3.0⁩
```
**Win**
```java
AS安装目录/⁨gradle⁩/⁨m2repository⁩/⁨com⁩/⁨android⁩/⁨tools⁩/build⁩/gradle⁩/⁨3.3.0⁩
```

# 实战练习
## 1.修改APK输出路径,和APK文件名.
我们在配置`Android`相关的信息时, 通常都是在:
```java
android{
   ...
   ...
}
```
这样的`DSL`语句中.

那么我们先`println`一下, 看看 `Android` DSL到底是什么玩意.

```java
android{
    println "0:" + it.class  //打印一下, 看看是那个类. 这是关键代码, 很多时候, 我们的突破点都是找到目标处理的类名.
}
```
经过运行,可以看到输出信息:

`0:class com.android.build.gradle.internal.dsl.BaseAppModuleExtension_Decorated`

很明显, 处理类是 `BaseAppModuleExtension`

通过阅读源码, 阅读继承类源码, 阅读接口源码.
可以找到:
```
//AppExtension类的getApplicationVariants方法
com.android.build.gradle.AppExtension#getApplicationVariants

//方法
public DomainObjectSet<ApplicationVariant> getApplicationVariants() {
    return applicationVariantList;
}
```
找到类:`ApplicationVariant`
```
com.android.build.gradle.api.ApkVariant#getPackageApplication

PackageAndroidArtifact getPackageApplication();
```

找到类:`PackageAndroidArtifact`
```
//找到PackageAndroidArtifact类的成员变量outputDirectory
com.android.build.gradle.tasks.PackageAndroidArtifact#outputDirectory

//见名知意, 这个应该是APP的输出目录, ok.完成1处.
protected File outputDirectory;
```

```
//继续查看成员变量
protected OutputScope outputScope;

//
com.android.build.gradle.internal.scope.OutputScope#apkDatas

//发现成员变量
@NonNull private final List<ApkData> apkDatas;

//阅读相关类
com.android.ide.common.build.ApkData

//找到成员变量, 见名知意, 这个应该是APP的输出文件名, ok.完成2处.
com.android.ide.common.build.ApkData#outputFileName

private String outputFileName;
```

OK, 修改点已经找到. 现在写代码修改.

```java
android {
    '''
    /*Gradle3.0 以上的方法*/
    applicationVariants.all { variant ->
        if (variant.buildType.name != "debug") {
            variant.getPackageApplication().outputDirectory = new File(project.rootDir.absolutePath + "/apk")
        }

        variant.getPackageApplication().outputScope.apkDatas.forEach { apkData ->
            apkData.outputFileName = ((project.name != "app") ? project.name : rootProject.name) + "-" +
                    variant.versionName + "_" +
                    apk_time + "_" +
                    variant.flavorName + "_" +
                    variant.buildType.name + "_" +
                    variant.signingConfig.name + "_" +
                    signConfig.key_alias +
                    ".apk"
        }
    }

     /*Gradle3.3 以上的方法*/
    applicationVariants.all { variant ->
        if (variant.buildType.name != "debug") {
            variant.getPackageApplicationProvider().get().outputDirectory = new File(project.rootDir.absolutePath + "/apk")
        }

        variant.getPackageApplicationProvider().get().outputScope.apkDatas.forEach { apkData ->
            apkData.outputFileName = ((project.name != "app") ? project.name : rootProject.name) + "-" +
                    variant.versionName + "_" +
                    apk_time + "_" +
                    variant.flavorName + "_" +
                    variant.buildType.name + "_" +
                    variant.signingConfig.name + "_" +
                    signConfig.key_alias +
                    ".apk"
        }
    }
    ...
}
```

结束.

## 相关类

```
//println project.pluginManager.findPlugin("com.android.application").class
org.gradle.api.internal.plugins.DefaultAppliedPlugin

com.android.build.gradle.internal.dsl.BaseAppModuleExtension
com.android.build.gradle.api.ApplicationVariant

//println it.class
com.android.build.gradle.internal.api.ApplicationVariantImpl
com.android.build.gradle.internal.dsl.BaseFlavor
```

## 2.修改AAR输出路径,和AAR文件名.

分析步骤:
1. 通过`DSL`语句, 拿到对应的`Java`处理类
2. 通过`Java`处理类, 找到对应的`成员变量`进行修改.


### a.LibraryPlugin
```
//DSL语句
apply plugin: 'com.android.library'

//拿到插件对应的Java类
println project.plugins.findPlugin("com.android.library").class
```
输出结果:
```java
//找到Java类LibraryPlugin
class com.android.build.gradle.LibraryPlugin
```

### b.LibraryExtension
```
android {
    compileSdkVersion 28
    defaultConfig {
       ...
    }

    buildTypes {
       ...
    }
    //DSL语句
    println it.class
}
```
输出结果:
```java
//找到Java类LibraryExtension
class com.android.build.gradle.LibraryExtension_Decorated
```

### c.LibraryVariantImpl
通过查看`LibraryExtension`类源码, 找到方法.
```
 public DefaultDomainObjectSet<LibraryVariant> getLibraryVariants() {
    return libraryVariantList;
 }
```
打印此方法所有值:
```
android {
    compileSdkVersion 28
    defaultConfig {
       ...
    }

    buildTypes {
       ...
    }

    //DSL语句
    libraryVariants.all { variant ->
        println variant.class
    }
}
```
输出结果:
```java
//找到关键类LibraryVariantImpl
class com.android.build.gradle.internal.api.LibraryVariantImpl_Decorated
```

### d.LibraryVariantOutputImpl
通过查看类`LibraryVariantImpl`的源码和继承关系, 找到方法:
```
@NonNull
@Override
public DomainObjectCollection<BaseVariantOutput> getOutputs() {
    return outputs;
}
```
枚举打印输出:
```
variant.outputs.all { output ->
    println  output.class
}
```
```
//找到关键类LibraryVariantOutputImpl
class com.android.build.gradle.internal.api.LibraryVariantOutputImpl_Decorated
```

### e.apkData
通过查看`LibraryVariantOutputImpl`源码, 找到方法和成员变量:
```
//关键成员
@NonNull protected final ApkData apkData;

//方法
@Override
@NonNull
protected ApkData getApkData() {
    return apkData;
}
```

### f.outputFileName
通过查看`ApkData`源码, 找到成员变量:
```
//找到关键点, 见名知意.肯定是用来修改文件名的.
private String outputFileName;
```

通过查看`LibraryVariantOutputImpl`源码, 还找到方法:
```
//关键方法getOutputFile,
//在老版本的android build gradle中,
//可以直接通过outputFile成员变量, 修改路径和文件名.
//新版本不行了, 区别就在下面的方法中.
@NonNull
@Override
public File getOutputFile() {
    Zip packageTask = getPackageLibrary();
    if (packageTask != null) {
        return new File(packageTask.getDestinationDir(), apkData.getOutputFileName());
    } else {
        return super.getOutputFile();
    }
}
```
通过上面方法的`getOutputFile`, 能知道.
文件名由`apkData.getOutputFileName()`决定;
文件路径由`packageTask.getDestinationDir()`决定.

**so**

**文件路径修改**
```
output.packageLibrary.destinationDir = new File(project.rootDir.absolutePath + "/aar")
```

仔细观察会发现`packageLibrary`在`LibraryVariantImpl`类中, 就有方法可以获取:
```
//AS 3.0版本
@Override
@Nullable
public Zip getPackageLibrary() {
    variantData
            .getScope()
            .getGlobalScope()
            .getDslScope()
            .getDeprecationReporter()
            .reportDeprecatedApi(
                    "variant.getPackageLibraryProvider()",
                    "variant.getPackageLibrary()",
                    TASK_ACCESS_DEPRECATION_URL,
                    DeprecationReporter.DeprecationTarget.TASK_ACCESS_VIA_VARIANT);
    return variantData.getTaskContainer().getBundleLibraryTask().getOrNull();
}

//AS 3.3版本
@Nullable
@Override
public TaskProvider<Zip> getPackageLibraryProvider() {
    //noinspection unchecked
    return (TaskProvider<Zip>) variantData.getTaskContainer().getBundleLibraryTask();
}
```

**最终修改方式如下:**
```
android {
    '''
    libraryVariants.all { variant ->
        if (variant.buildType.name != "debug") {
            variant.getPackageLibraryProvider().get().destinationDir = new File(project.rootDir.absolutePath + "/apk")
        }

        variant.outputs.all { output -> 
            output.apkData.outputFileName = ((project.name != "app") ? project.name : rootProject.name) + "-" +
                    defaultConfig.versionName + "_" +
                    variant.buildType.name +
                    ".aar"
        }
    }
    ...
}
```


>欢迎访问 [博文](https://blog.csdn.net/angcyo/article/details/78357512)


