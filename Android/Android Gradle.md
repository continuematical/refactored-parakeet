# 基础和构建机制
## 模块下的 build.gradle
目录 app.build.gradle。可以覆盖主目录的 build.gradle。
主要分为三个模块。
### plugin
声明接下来需要用到哪些插件的内容。
```groovy
plugins {  
	//android gradle插件
    id 'com.android.application'  
    //kotlin插件
    id 'org.jetbrains.kotlin.android'  
}
```
### android
Android 构建配置区域，包括编译版本、默认配置、构建类型等信息。
```groovy
android {  
    namespace 'com.example.greetingcard'  
    compileSdk 32  

//配置应用的默认属性
    defaultConfig {  
        applicationId "com.example.greetingcard"  
        minSdk 21  
        targetSdk 32  
        versionCode 1  
        versionName "1.0"  
  
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"  
        vectorDrawables {  
            useSupportLibrary true  
        }  
    }  
    buildTypes {  
        release {  
            minifyEnabled false  
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'  
        }  
    }    compileOptions {  
        sourceCompatibility JavaVersion.VERSION_1_8  
        targetCompatibility JavaVersion.VERSION_1_8  
    }  
    kotlinOptions {  
        jvmTarget = '1.8'  
    }  
    buildFeatures {  
        compose true  
    }  
    composeOptions {  
        kotlinCompilerExtensionVersion '1.1.1'  
    }  
    packagingOptions {  
        resources {  
            excludes += '/META-INF/{AL2.0,LGPL2.1}'  
        }  
    }
}
```
### dependencies
依赖配置，添加要使用的库或者本地`jar`包。
```groovy
dependencies {  
//kotlin扩展函数的核心库
    implementation 'androidx.core:core-ktx:1.7.0'  
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.3.1'  
    implementation 'androidx.activity:activity-compose:1.3.1'  
    implementation "androidx.compose.ui:ui:$compose_ui_version"  
    implementation "androidx.compose.ui:ui-tooling-preview:$compose_ui_version"  
    implementation 'androidx.compose.material:material:1.1.1'  
    implementation 'androidx.navigation:navigation-compose:2.6.0-alpha08'  
    testImplementation 'junit:junit:4.13.2'  
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'  
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'  
    androidTestImplementation "androidx.compose.ui:ui-test-junit4:$compose_ui_version"  
    debugImplementation "androidx.compose.ui:ui-tooling:$compose_ui_version"  
    debugImplementation "androidx.compose.ui:ui-test-manifest:$compose_ui_version"  
}
```
## 主目录下的 build.gradle
配置所有模块通用的配置信息。
```groovy
//配置构建脚本本身
buildscript {  
	//定义变量
    ext {  
        compose_ui_version = '1.1.1'  
    }  
    //可以引用，指定UI版本
}

plugins {  
    id 'com.android.application' version '7.3.1' apply false  
    id 'com.android.library' version '7.3.1' apply false  
    id 'org.jetbrains.kotlin.android' version '1.6.10' apply false  
}
```
## settings.gradle
通常用于定义项目的模块结构。表示项目中包含哪些子项目。
```groovy
//管理gradle插件部分
pluginManagement {  
    repositories {  
        maven { url=uri("https://maven.aliyun.com/repository/google") }  
        maven { url=uri("https://maven.aliyun.com/repository/public") }  
        gradlePluginPortal()  
        google()  
        mavenCentral()  
    }  
}  
//管理项目的依赖解析
dependencyResolutionManagement {  
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)  
    repositories {  
        maven { url=uri("https://maven.aliyun.com/repository/google") }  
        maven { url=uri("https://maven.aliyun.com/repository/public") }  
        google()  
        mavenCentral()  
    }  
}  
//根项目名称
rootProject.name = "GreetingCard"  
//表示包含的子项目
include ':app'
```


