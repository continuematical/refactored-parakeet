# 基本概念
Navigation Graph
资源，收集各个节点的跳转信息。
NavHost
作为一个节点，容器显示Graph中处于栈顶的Destination。
NavController
执行跳转行为的管理者。具有状态并跟踪组成应用程序屏幕的可组合项的后退栈以及每个屏幕的状态。
# 导航
添加依赖
```xml
implementation "androidx.navigation:navigation-compose:2.7.1"
```
创建一个NavController
```kotlin
val navController = rememberNavController()
```
可以使用`composable()`方法来添加导航结构。该方法要求您提供一个路由和应与目的地关联的可组合项。
