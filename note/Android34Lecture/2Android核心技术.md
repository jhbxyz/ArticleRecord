

# Day13 Android 是如何通过 Activity 进行交互的？



## 1.taskAffinity

单纯使用 taskAffinity 不能导致 Activity 被创建在新的任务栈中，需要配合 singleTask 或者 singleInstance！

## 2.taskAffinity + allowTaskReparenting

allowTaskReparenting 赋予 Activity 在各个 Task 中间转移的特性。

## 3.通过 Binder 传递数据的限制

通常情况为 1M，但是根据不同版本、不同厂商，这个值会有区别

解决办法：

* 减少通过 Intent 传递的数据，将非必须字段使用 **transient** 关键字修饰。

* 将对象转化为 JSON 字符串，减少数据体积。

  > JVM 加载类通常会伴随额外的空间来保存类相关信息，将类中数据转化为 JSON 字符串可以减少数据大小。比如使用 Gson.toJson 方法。
  >
  > 将类转化为 JSON 字符串之后，还是会超出 Binder 限制，说明实际需要传递的数据是很大的。这种情况则需要考虑使用本地持久化来实现数据共享，或者使用 EventBus 来实现数据传递。

## 4.process 造成多个 Application

RemoteActivity 的 process 为“lagou.process”，这将导致它会在一个新的进程中创建。当在 MainActivity 中跳转到 RemoteActivity 时，Application 会被再次创建

解决办法：

* onCreate 方法中判断进程的名称，只有在符合要求的进程里，才执行初始化操作；
* 抽象出一个与 Application 生命周期同步的类，并根据不同的进程创建相应的 Application 实例。

## 5.后台启动 Activity 失效

主要目的就是尽可能的避免当前前台用户的交互被打断，保证当前屏幕上展示的内容不受影响

解决办法：
Android 官方建议我们使用通知来替代直接启动 Activity 操作：