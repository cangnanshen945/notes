# Android 编译
Android apk的编译开发中是利用Android Studio集成gradle plugin进行，此外项目中经常也会用到jenkins集成gradle来进行编译。
## gradle task
Android的编译是通过一系列gradle task来完成的，过程大致如下：
- -> res/jniLibs/shaders/AIDL(prebuild、 merge、 process等流程生成R.java、AIDL的binder、以及资源文件、so文件、manifest文件等的合并工作) 
- -> kapt(生成代码) 
- -> compileReleaseKotlin kotlin编译
- -> compileJava java编译
- -> mergeProguardFiles (混淆文件合并)
- -> mergeReleaseClasses/mergeReleaseJavaResources class文件合并
- -> proguard 文件混淆（minify）
-  -> dexBuilderRelease 合成dex
-  -> multidexRelease/mergeDexRelease dex拆分
-  -> 使用dynamicfeature时 会有splitReleaseDex来在merge后进行分包
-  -> packageRelease/assembleRelease 真正完成apk的打包
## agp版本
随着gradle版本升级，api也会有很多不兼容的地方，task的内容与执行方式也有变化，所以在使用脚本时注意gradle版本

例如修改multidex，并且指定maindex的类就有以下三种方式：
1. 遍历执行的任务，在包含dx前缀的任务中，增加配置参数
    ```
    afterEvaluate {
        tasks.matching {
            it.name.startsWith('dex')
        }.each { dx ->
            if (dx.additionalParameters == null) {
                dx.additionalParameters = []
            } else {
                dx.additionalParameters += '--multi-dex'
            }
        }
    }
    ```
2. 在dexOptions中配置参数
   ```
   dexOptions {
        additionalParameters = ['--main-dex-list=' + 'multidex.keep'] // 具体的可用参数，可以利用Library/Android/sdk/buildtools/version/dx --help获取
    }
    ```
3. 目前官网方案，在android的config中加入multidexKeefFile
    ```
    multiDexEnabled true
    multiDexKeepProguard file('multidexkeep.pro') // keep specific classes using proguard syntax
    multiDexKeepFile file('multidex.keep')
    ```