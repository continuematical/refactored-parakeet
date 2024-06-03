[ButterKnife被弃用，ViewBinding才是findView的未来](https://zhuanlan.zhihu.com/p/321262046)
### Kotlin Android Extensions
添加插件
```kotlin
plugins {  
    id("com.android.application")  
    id("org.jetbrains.kotlin.android")  
    id("kotlin-android")  
    id("kotlin-android-extensions")  
}
```
就可以直接使用`ID`获取控件。
### ViewBinding
Android Studio对于ViewBinding的支持是从3.6版本开始的，设置配置：
```kotlin
android {
    buildFeatures {
        viewBinding = true
    }
}
```
重新编译后系统会为每个布局文件生成对应的Binding类，其中包含对应布局所有ID的直接引用，其目录在`_模块根目录/build/generated/data_binding_base_class_source_out_` 下。

编写`activity_main`布局文件，如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/text_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
完成后`gradle`插件会自动生成一个名为`ActivityMainBinding`的`Java`类，在`Activity`中通过`ActivityMainBinding`获取`Binding`实例，如下：
```java
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding = ActivityMainBinding.inflate(layoutInflater)
        //通过这个实例可以直接访问布局文件中的控件
        setContentView(binding.root)
        binding.textView.text = "Hello World"
    }
```
1. `Binding` 文件名根据布局文件名驼峰命名规则生成；
2. 如果想在生成绑定类时忽略某个布局文件，将 `tools:viewBindingIgnore="true"`属性添加到相应布局文件的根视图中。