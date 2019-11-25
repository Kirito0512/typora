### 1. 打包相关的命令行指令

[编译和部署APK](https://developer.android.google.cn/studio/build/building-cmdline?hl=zh-cn#build_apk)

#### 1.1 Debug包

要立即测试和调试应用，您可以编译调试API。调试APK会使用SDK工具提供的调试密钥进行签名，并允许通过adb调试。

要编译调试APK，请打开命令行，然后转到项目根目录，调用 assembleDebug 任务

`gradlew assembleDebug`

这样将在 `project_name/module_name/build/outputs/apk/` 中创建一个名为 `module_name-debug.apk` 的 APK。该文件已使用调试密钥进行签名并使用 [`zipalign`](https://developer.android.com/tools/help/zipalign.html?hl=zh-cn) 对齐，因此您可以立即将其安装在设备上。

或者，要编译 APK 并立即将其安装在运行的模拟器或连接的设备上，请调用 `installDebug`：

`gradlew installDebug`

#### 1.2 Release包

> 打release包，需要签名文件，可以直接使用Android Studio生成jks文件。
>
> 参考 [生成上传密钥和密钥库](https://developer.android.com/studio/publish/app-signing?hl=zh-cn#generate-key)

我们可以配置Gradle来为应用签名

打开模块级 `build.gradle` 文件，并添加带有 `storeFile`、`storePassword`、`keyAlias` 和 `keyPassword` 条目的 `signingConfigs {}` 块，然后将该对象传递到您的版本类型中的 `signingConfig` 属性

```
android {
        ...
        defaultConfig { ... }
        signingConfigs {
            release {
                // You need to specify either an absolute path or include the
                // keystore file in the same directory as the build.gradle file.
                storeFile file("my-release-key.jks")
                storePassword "password"
                keyAlias "my-alias"
                keyPassword "password"
            }
        }
        buildTypes {
            release {
                signingConfig signingConfigs.release
                ...
            }
        }
    }
   
```

> **注意**：在这种情况下，密钥库和密钥密码可直接在 `build.gradle` 文件中查看。为提升安全性，您应[从编译文件中移除签名信息](https://developer.android.google.cn/studio/publish/app-signing?hl=zh-cn#secure-shared-keystore)。



现在，当您通过调用 Gradle 任务来编译您的应用时，Gradle 会为您的应用签名（并运行 zipalign）。

此外，因为您已使用签名密钥配置了发布版本，所以该版本类型可以执行“安装”任务。因此，您可以使用 `gradlew installRelease` 任务在模拟器或设备上编译、对齐、签署和安装发布版 APK。

如果不想安装，可以运行`gradlew assembleRelease`



### 参考

[对应用进行签名](https://developer.android.google.cn/studio/publish/app-signing?hl=zh-cn#secure-shared-keystore)

[从命令行编译您的应用](https://developer.android.google.cn/studio/build/building-cmdline?hl=zh-cn)