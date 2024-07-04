# 某视频APP的AGP8升级之旅

# 前言

是的，2024年我还在做`Android`原生开发，没有`kmp`，没有`遥遥领先`。

本次Android大仓的 `AGP8` 升级涵盖多个APP多个业务方，持续3个月；分各个三大步，若干小步完成升级迁移，以下为本次升级踩坑经历。

# 升级与踩坑

本次AGP升级计划从 `7.2.2` 升级到 `8.2.2`，`AGP`中最大的变动点是 `Transform Api` 接口的废弃、以及默认编译特性的开启。
同时也要顺带升级 `Gradle` 版本，部分`Gradle`的`dsl`写法也有较大变动。基于以上变动，本着小步快跑的原则，分三步走，持续迭代。

* 第一步，小升级到 `7.4.2`，保障编译正常，计划迁移完所有的Transform接口，并提供编译数据，查看是否有劣化可能。
* 第二步，升级到 `8.2.2`，并解决编译兼容问题，部分编译。提供升级后数据，主要有apk体积、编译时间、r8 兼容等几方面。
* 第三步，细节优化，部分特性开启，优化升级后的数据劣化问题。

### Transform Api

先检测出项目中使用Transform Api的插件，找出归属业务负责人，然后排期修改。实际操作中，发现Transform Api的插件有只有4个，3个属于自研，还有一个属于华为推送。

```gradle
afterEvaluate {
    if (project.plugins.hasPlugin('com.android.application')) {
        project.gradle.buildFinished {
            def app = project.extensions.getByType(com.android.build.gradle.AppExtension.class)
            logger.quiet("buildFinished, project(${project.name}) has ${app.transforms.size()} transforms")
            app.transforms.forEach {
                logger.quiet("transforms: ${it.name} -> ${it.getClass().name}")
            }
        }
    }
}
```
第三方推送插件华为官网文档即可，自研插件需要参照Android官网文档修改。

### Namespace

默认情况下，`AGP8` 会要求开启`namespace`，可以使用脚本去除xml中的`package`，并在gradle中添加`namespace`。

此步骤可以单独上线，无需等待。后续的 `nonTransitiveRClass` 开启需依赖此步骤。

### BuildConfig

默认情况下，`AGP8` 不会生成 `BuildConfig`。所以全局默认关闭，部分模块方按需开启。

```shell
android.defaults.buildfeatures.buildconfig=true

buildFeatures { buildConfig = true }
```

### nonFinalResIds

默认情况下，`AGP8` 也会开启` nonFinalResIds`。但是这个无法再某个模块单独开启，只能全局开启。

我们公司采用大仓模块式，`gradle.properties`全局使用一份，迁移过程中发现有一个APP的某个功能模块使用了 `butterknife`，所以只能在 `ci` 脚本中单独关闭。
后续再归还迁移 `butterknife` 到 `databinding` 之后，再统一开启。

```shell
./gradlew app:assembleRelease -Pandroid.nonFinalResIds=false
```

### JVM相关

`AGP` 升级后，`JDK` 需要使用17版本，同时升级的`Gradle`默认`target`也有修改，可以全局修改，此部分可以提前上线，无需等待。

```gradle
allprojects {
    if (project.plugins.hasPlugin('com.android.application') || project.plugins.hasPlugin('com.android.library')) {
        android.compileOptions {
            sourceCompatibility JavaVersion.VERSION_1_8
            targetCompatibility JavaVersion.VERSION_1_8
        }
    }
    if (project.plugins.hasPlugin("kotlin-android")) {
        android.kotlinOptions {
            jvmTarget = "1.8"
            freeCompilerArgs += ['-Xjvm-default=all']
        }
    }

    tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).all {
        kotlinOptions {
            jvmTarget = "1.8"
            freeCompilerArgs += ['-Xjvm-default=all']
        }
    }
```

### Gradle 升级

`Gradle`版本也从7升级到8，部分官方的插件也有对应修改，比如项目中使用到的单元测试、lint、jcocoa等。

具体修改可以参考官方文档，这里只列出部分修改 https://docs.gradle.org/current/userguide/upgrading_version_7.html。

实操过程还发现一个奇怪问题，本人使用的是MacBook Pro，有同学使用Mac M2，发现编译会报错。后排查需要安装下载相关工具链。

```gradle
// settings.gradle

buildscript {
    dependencies {
       classpath "org.gradle.toolchains:foojay-resolver:${foojay_version}"
    }
}

apply plugin: "org.gradle.toolchains.foojay-resolver-convention"
```

### 第三方库

实际过程中，大部分第三方库都有适配到`AGP8`，但是也有个别库没有适配，导致编译报错。

项目中使用到了 `greendao`，需要特殊适配。

```gradle
// https://github.com/greenrobot/greenDAO/issues/1110#issuecomment-1734949669
tasks.configureEach { task ->
    if (task.name.matches("\\w*compile\\w*Kotlin")) {
        task.dependsOn('greendao')
    }
    if (task.name.matches("\\w*kaptGenerateStubs\\w*Kotlin")) {
        task.dependsOn('greendao')
    }

    if (task.name.matches("\\w*kapt\\w*Kotlin")) {
        task.dependsOn('greendao')
    }
}
```

部分自研库还有使用 `okio 1.*` 的版本，实际迁移过程中，okio版本会自动依赖到 `okio 3.*`，导致编译报错。

因为在接口定义时，添加了 `@Deprecated` 注解，导致编译报错。

```kotlin
@Deprecated(
    message = "moved to extension function",
    replaceWith = ReplaceWith(
      expression = "file.sink()",
      imports = ["okio.sink"],
    ),
    level = DeprecationLevel.ERROR,
  )
fun sink(file: File) = file.sink()
```

我们的解决方式是，在使用 `okio` 接口的模块中，手动压制编译报错，同时添加`lint`自定义检查，促使业务方尽快修改。
这样改动影响最小，只需要升级全局版本号和添加注解压制，业务方无需改动，同时不影升级进度。

```kotlin
// 临时压住okio编译问题，请尽快修复
@Suppress("DEPRECATION_ERROR")
```

# 再谈 Transform

本次升级过程中，`Transform` 是最重大的升级之一。一些自定义的插件主要使用 `Transform` 来实现 `asm` 代码的插入和修改。

以下是一个简单的 `Transform` 示例，首先定义个 `Task`，绑定到 `Variant` 生命周期中。

```kotlin
val ac = project.extensions.getByType(ApplicationAndroidComponentsExtension::class.java)
ac.onVariants { variant ->
    if (extension.enableTfTask()) {
        val taskProvider = project.tasks.register<TfNothingTask>("${variant.name}TfNothing")
        taskProvider.configure {
            it.extension = extension
        }
        variant.artifacts.forScope(ScopedArtifacts.Scope.ALL)
            .use(taskProvider)
            .toTransform(
                ScopedArtifact.CLASSES,
                TfNothingTask::allJars,
                TfNothingTask::allDirectories,
                TfNothingTask::output
            )
    }
}

abstract class TfNothingTask : DefaultTask() {

    @get:InputFiles
    abstract val allJars: ListProperty<RegularFile>

    @get:InputFiles
    abstract val allDirectories: ListProperty<Directory>

    @get:OutputFile
    abstract val output: RegularFileProperty

    @Internal
    lateinit var extension: TfNothingExtension

    @TaskAction
    fun taskAction() {
    }
}
```

有一个很大的区别是，它的 `ouput` 是一个 jar文件，而原先的 `Transform`，可以是`class`文件，也可是`jar`文件。

如果定义多个 `Transform`，那么 `allDirectories` 是空的，`allJars` 也只有一个，并且是上一个的 `output`。

乍一看，感觉没什么问题，但是实际使用过程中，发现坑很多。比如只是简单修改一个类，编译后，整个task需要重新执行，需要自己实现增量编译。

恰恰好我们的项目很大，产生的`jar`文件体积也很大，所以每次编译时间很长。

实操过程中，我们发现降低`jar`的压缩率，并且使用`buffer`操作，可以降低编译时间。注意这样会增大`jar`的体积，尤其是磁盘空间比较紧张情况下，需要格外注意。

如果是某个收集信息类型的任务，可以使用多线程来提高效率，多线程读取不会有并发问题。

```kotlin
val jos = JarOutputStream(FileOutputStream(tmpOutput).buffered())
jos.setLevel(Deflater.NO_COMPRESSION)
```

同时需要注意，在生成`jar`文件时，`entry` 需要设置时间为0，否则编译缓存会失效。

```kotlin
fun addEntry(jos: JarOutputStream, name: String, data: ByteArray) {
    val entry = JarEntry(name)
    entry.time = 0
    jos.putNextEntry(entry)
    jos.write(data)
    jos.closeEntry()
}
```

后续的任务就是 `Dex` 操作，android编译过程中会有 `dexBuilder` 和 `mergeDex` 操作。

上面已经说过，产物只有一个 jar，实际过程中任意的代码修改，会导致整个任务重新执行，增量编译失效。

我们做法是，hook `dexBuilder` 编译过程，将输入的jar大文件分成多个 小 jar 文件，每个小jar文件，再单独做 `dexBuilder` 操作。
我们这里分成30组，原始文件大概 1200M，拆分成30个jar文件，每个文件大概30M左右，使用多线程分割，5s 内可完成分割。

然后 `mergeDex` 过程中，不再一起合并，而是上一个task中分组产物小范围合并，我们是分成10个小组，小组内执行合并操作，最后不再整体合并，最好直接写入到 APK。

上面的过程会替换原有的task，并且需要禁用原始的task的缓存动作，避免浪费时间。
同时需要自己实现缓存操作，可以直接对小组内文件求hash，然后写入磁盘中，当再次执行时，判断文件hash，没有变化则直接使用缓存。

# 开启部分编译特性

## nonTransitiveRClass

这个特性已经提供了很久，但是优于历史原因项目移植无法开启，并且官方提供升级助理来一键迁移，但是我们的项目太大，机器配置也不够，迁移过程中会导致机器卡死。

为此我们自己写了一个插件，前期先收集各个模块的资源，主要是为了获取各个模块的 `packageName` 或者是 `namespace`，就是R文件的 `package`，以及各个模块的资源名称。

可以先全源码编译一次，搜索各个模块的build文件夹，主要有 `R-def.txt` 和 `R.jar`，最后再汇总到一个新的`json`文件中，供插件使用。
如下所示，部分资源会有多个 `package`，可以根据权重优先级来选择，比如优先使用基础模块或公共模块的资源。

```json
// color.json
{
  "auto_night_shade": [
    "com.xxx.lib.widget"
  ],
  "avatar_color_transparent": [
    "com.xxx.lib.widget",
    "com.xxx.lib.accountsui"
  ],
}

// id.json
{
  "abTestResultTv": [
    "com.xxx.gripper.app"
  ],
  "anr_btn": [
    "com.xx.gripper.app"
  ],
}
```

然后根据插件的`PSI`接口，直接替换掉R的导入。

```kotlin
// 替换Java代码中的R
private fun replaceExpression(expression: PsiReferenceExpression, s: String) {
    val e2 = psiFactory.createReferenceFromText(s, null)
    expression.replace(e2)
    logger.println("replaceExpression = $expression, newExpression = $s")
}

// 替换Kotlin代码中的R
 private fun replaceExpression(expression: KtDotQualifiedExpression, s: String) {
    val e2 = psiFactory.createExpression(s)
    expression.replace(e2)
    logger.println("replaceExpression = $expression, newExpression = $s")
}
```

以上即可满足90%的手动操作，剩下 10%可能会有误差，需要手动调整。


部分模块中使用 `databinding` 特性，如下所示，不能简单的进行R替换，修改2会导致颜色问题不对，因为返回的是资源id，并不是实际颜色数值。
需要根据id查找对应颜色，如下修改2才正确。亦或者直接把 color1 与 color2 下沉到公共模块中，这样修改1就可以正确了。

```xml
# 修改前
android:textColor="@{vm.success? @color/color1 : @color/color2}"
# 修改1
android:textColor="@{vm.success? com.xxx.R.color.color1 : com.yyy.R.color.color2}"
# 修改2
android:textColor="@{util.getResColor(vm.success? R.color.color1 : R.color.color2)}"
# 修改3
android:textColor="@{vm.success? @color/color1 : @color/color2}"
```

## R8 问题

优于某些原因，暂时没有 `R8 fullMode`，后续计划重新测试并开启。

部分 `C++` 代码需要调用 `Java` 代码，使用 `R8` 编译后，会导致部分方法无法调用, 报错为无法找到构造函数，需要添加一下参数，来禁用优化。

```shell
./gradlew :app:assembleRelease -Dcom.android.tools.r8.disableApiModeling=1
```

同时我们也发现 `mapping.txt` 文件格式发生了变化，部分插件化或者热修复的代码可能要对应的修改。实际操作过程中，可能需要自己选择合适的版本，官方也在不停的修复问题。

```toml
r8-plugin = "com.android.tools:r8:8.1.56"
#r8-plugin = "com.android.tools:r8:8.3.37"
```

同时我们也发现自己项目中无法对 `APK` 中的 `META-INF/**_release.kotlin_module` 进行默认剔除，手动添加规则也无法剔除，自测的demo工程是可行的，比较奇怪。

```gradle
packaging {
    exclude 'META-INF/**_release.kotlin_module'
}
```

最后我们对 `mergeReleaseJavaResource` 这个任务进行 hook，在任务执行后，对 `base.jar` 进行处理，剔除 `*_release.kotlin_module` 的 `entry`文件。
        
还有一种情况，有个单独APP，在执行 `mergeReleaseJavaResource` 任务时，无法生成 `base.jar`，导致后续任务无法执行。
最终发现 `whenTaskAdded` 对此有影响，删除或者修改`whenTaskAdded`逻辑即可解决，网上搜索说是 Gradle 的 bug。

```gradle
// 移除或者修改对应逻辑
tasks.whenTaskAdded { task ->

}
```

# 数据劣化与治理

整个升级过程持续3个月左右，编译耗时和产物体积都有明显的劣化趋势。

比如上面的单个jar文件问题，采用降低压缩率、buffer读取写入、多线程操作、分组dex可有效磨平升级代理的劣化。

开启 `nonTransitiveRClass` 特性后，对整个模块的变异速度也有较大的提升，并且对单个Jar文件的大小也有明显的降低，最终导致 Debug包体积有明显的降低(250M到200M)，Release包体积提升不明显。

# 总结

以上就是本次升级的全部过程，虽然过程比较曲折，但是最终还是完成了。路漫漫其修远兮，吾将上下而求索。

# 附录

* 插件更新 https://developer.android.com/build/releases/gradle-plugin-api-updates?hl=zh-cn

* 8.2.0 https://developer.android.com/build/releases/gradle-plugin?hl=zh-cn

* 8.1.0 https://developer.android.com/build/releases/past-releases/agp-8-1-0-release-notes?hl=zh-cn

* 8.0.0  https://developer.android.com/build/releases/past-releases/agp-8-0-0-release-notes?hl=zh-cn

* 7.4.0 https://developer.android.com/build/releases/past-releases/agp-7-4-0-release-notes?hl=zh-cn

* 7.3.0 https://developer.android.com/build/releases/past-releases/agp-7-3-0-release-notes?hl=zh-cn

* d8与kotlin https://developer.android.com/build/kotlin-support?hl=zh-cn

* Gradle 7 & 8 https://docs.gradle.org/current/userguide/upgrading_version_7.html
