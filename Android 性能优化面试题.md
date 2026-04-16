# Android 性能优化面试题

> 适合 Android 中高级面试复习。答案按“定义 -> 原因 -> 排查 -> 优化 -> 指标”组织，面试时可以根据时间长短裁剪。

## 一、性能优化总览

### 1. Android 性能优化通常包含哪些方向？

性能优化不是单点技巧，而是围绕用户体验和资源消耗做系统治理，常见方向包括：

- 启动优化：冷启动、温启动、热启动，降低 TTID 和 TTFD。
- 渲染优化：减少掉帧、卡顿、Frozen Frame、过度布局和过度绘制。
- 内存优化：降低内存占用、减少 GC、避免内存泄漏、降低 LMK 风险。
- 线程与 ANR 优化：主线程不卡顿，耗时任务放到合适线程，避免死锁和 Binder 阻塞。
- 包体积优化：R8、资源压缩、App Bundle、动态特性、移除无用资源。
- 电量优化：减少 wake lock、后台网络、频繁唤醒、定位和传感器滥用。
- 网络优化：减少请求次数、压缩、缓存、连接复用、弱网降级。
- 存储与 I/O 优化：避免主线程 I/O，减少频繁小文件读写，使用异步和批量写入。
- Compose 或 View 专项优化：减少无效重组、优化 RecyclerView/LazyColumn、减少复杂层级。
- 工程化监控：Android vitals、Crash/ANR 平台、埋点、Benchmark、Perfetto、CI 性能门禁。

### 2. 性能优化的基本方法论是什么？

先度量，再定位，最后优化和回归。不要凭感觉改代码。

常用流程：

1. 明确指标：启动耗时、帧率、慢帧率、ANR 率、内存峰值、包体大小、电量消耗等。
2. 复现问题：固定设备、系统版本、账号数据、网络环境、操作路径。
3. 采集数据：Logcat、Android Studio Profiler、Perfetto、dumpsys、Android vitals、Macrobenchmark。
4. 定位瓶颈：看主线程、RenderThread、Binder、I/O、锁等待、GC、布局、网络。
5. 小步优化：一次只改一个主要因素，避免引入新问题。
6. 回归验证：对比优化前后数据，接入自动化测试或线上监控。

### 3. 面试中如何回答“你做过哪些性能优化”？

可以按 STAR 结构回答：

- 背景：哪个业务场景，例如首页冷启动慢、列表滑动卡、详情页内存高。
- 指标：优化前 TTID/TTFD、慢帧率、P95 耗时、ANR 率、内存峰值是多少。
- 定位：用了哪些工具，例如 Perfetto 看到主线程被 I/O 阻塞，Profiler 看到频繁 GC。
- 方案：懒加载、异步化、缓存、减少布局层级、Baseline Profile、图片压缩等。
- 结果：优化后指标下降多少，并说明如何防止回退。

一个好的回答必须有数据闭环，而不是只说“我把耗时操作放到子线程”。

## 二、启动优化

### 4. 冷启动、温启动、热启动有什么区别？

- 冷启动：进程不存在，系统需要创建进程、加载应用、创建 Application、启动主线程、创建 Activity、inflate 布局并完成首帧绘制。优化难度最大。
- 温启动：进程可能还在，但 Activity 需要重新创建，部分对象和缓存可以复用。
- 热启动：进程和 Activity 大多还在内存中，只是从后台回到前台，开销最小。

优化启动一般优先按冷启动处理，因为冷启动优化通常也能改善温启动和热启动。

### 5. TTID 和 TTFD 是什么？

- TTID：Time To Initial Display，首帧显示时间，表示用户看到第一个界面的时间。
- TTFD：Time To Fully Drawn，完全可交互时间，表示页面核心内容加载完成、用户真正可用的时间。

TTID 低只能说明“看起来出来了”，TTFD 低才说明“真的可用了”。例如首页先显示空壳但列表数据 3 秒后才出现，TTID 可能不错，但 TTFD 很差。

### 6. 如何获取启动耗时？

常见方式：

- Logcat 中查看 ActivityManager 的 `Displayed` 日志，获取首帧耗时。
- 在页面核心内容可用后调用 `reportFullyDrawn()`，获取 TTFD。
- 使用 Macrobenchmark 的 `StartupTimingMetric` 做冷/温/热启动基准测试。
- 使用 Perfetto 查看启动阶段主线程、Binder、I/O、类加载、布局绘制等耗时。
- 线上可通过埋点记录进程创建、Application、首 Activity、首帧、首页数据完成等阶段。

### 7. 冷启动慢通常有哪些原因？

常见原因：

- `Application.onCreate()` 里初始化太多 SDK。
- 主线程执行磁盘 I/O、数据库、SP 大文件读取、网络请求。
- 首页布局层级复杂，inflate 和 measure/layout/draw 耗时高。
- 类加载、反射、动态代理、依赖注入初始化过重。
- 首屏图片过大，解码或加载阻塞。
- ContentProvider 自动初始化过多。
- 多进程初始化重复执行。
- 首屏数据必须等待多个接口串行返回。

### 8. 启动优化有哪些常见手段？

- 延迟初始化：非首屏必要 SDK 放到首帧后或用户触发时初始化。
- 异步初始化：把可并发且线程安全的任务放到后台线程。
- 分级初始化：按“必须、尽快、空闲、按需”划分任务。
- 减少 ContentProvider 自动初始化，能手动初始化就手动控制时机。
- 避免主线程 I/O，SharedPreferences 大文件改为异步或 DataStore 等方案。
- 优化首屏布局，减少层级和复杂自定义 View。
- 首页数据并行请求，缓存优先展示，接口聚合或预拉取。
- 使用 SplashScreen API 避免白屏/黑屏体验问题。
- 使用 Baseline Profiles 让关键路径提前编译，减少解释执行和 JIT 成本。

### 9. Application 初始化如何治理？

核心是“启动期只做首屏必须的事”。

可以把初始化拆成：

- 必须同步：崩溃采集最小能力、核心配置、路由基础能力。
- 必须异步：日志、部分数据库预热、轻量缓存。
- 首帧后：广告、推送、地图、分享、统计完整能力。
- 按需：用户进入相关页面时再初始化。

注意：异步不是万能的。任务之间有依赖时要明确 DAG；涉及 UI、主线程 Handler、SDK 线程要求时要遵守调用约束；多进程下要避免每个进程都执行完整初始化。

### 10. ContentProvider 为什么会影响启动？

系统会在 `Application.onCreate()` 之前创建并初始化应用声明的 ContentProvider。很多 SDK 通过 ContentProvider 做自动初始化，这会把耗时提前到启动关键路径上。

优化方式：

- 移除不必要的自动初始化 Provider。
- 使用 AndroidX Startup 时关闭非必要 initializer。
- 对 SDK 初始化做按需、延迟或进程判断。
- 检查 manifest merge 后最终生成的 Provider 列表。

### 11. Baseline Profile 是什么？为什么能优化启动？

Baseline Profile 是一组关键代码路径规则，随 APK/AAB 一起发布。安装后 ART 可以对这些代码路径做 AOT 编译，从而减少首次运行时的解释执行和 JIT 编译开销。

它可以优化：

- 冷启动。
- 页面跳转。
- 列表滚动。
- Compose 首次组合和关键交互。

官方文档中给出的典型收益口径是关键路径执行速度可提升约 30%，实际项目要用 Macrobenchmark 验证。

### 12. Baseline Profile 和 Startup Profile 有什么区别？

- Baseline Profile：覆盖启动和常用用户路径，用于指导 ART 预编译热点代码，改善启动和运行时流畅度。
- Startup Profile：更关注 DEX 布局，把启动需要的类尽量放在主 dex 或更靠前的位置，减少启动阶段类加载和 I/O 成本。

面试回答可以说：Baseline Profile 偏“哪些代码要预编译”，Startup Profile 偏“启动相关代码如何布局得更容易读取”。

## 三、卡顿与渲染优化

### 13. Android 为什么要求 16ms 一帧？

60Hz 屏幕每秒刷新 60 次，一帧预算约 16.67ms。如果主线程、RenderThread 或 GPU 在一帧内没有完成输入处理、动画、布局、绘制和提交，系统就会跳过这一帧，用户看到卡顿。

高刷新率设备预算更紧：

- 90Hz：约 11ms。
- 120Hz：约 8ms。

所以“16ms”只是 60fps 下的经典基准，不代表所有设备都足够。

### 14. Slow Frame、Frozen Frame、ANR 的区别是什么？

- Slow Frame：通常指超过帧预算的慢帧，用户可能感到轻微卡顿。
- Frozen Frame：渲染超过 700ms 的帧，用户会明显感觉界面卡住。
- ANR：主线程长时间无法响应输入、广播、服务等系统调度，前台输入超时典型阈值是 5 秒。

可以理解为：慢帧是流畅性问题，Frozen Frame 是严重卡顿，ANR 是响应性故障。

### 15. 卡顿问题如何排查？

常见排查路径：

- 使用 Android vitals 看线上慢渲染、Frozen Frame、ANR 分布。
- 使用 Profile GPU Rendering 初步观察帧耗时。
- 使用 Perfetto/System Trace 查看主线程、RenderThread、CPU 调度、锁、Binder、I/O、GC。
- 使用 Android Studio CPU Profiler 定位热点方法。
- 使用 `adb shell dumpsys gfxinfo <package>` 查看帧统计。
- 对关键场景写 Macrobenchmark，记录慢帧和 trace。

排查时优先看可稳定复现的卡顿，再治理线上影响面最大的聚类。

### 16. 主线程卡顿常见原因有哪些？

- 主线程执行网络、数据库、文件读写、Bitmap 解码。
- RecyclerView 绑定数据时做复杂计算。
- 布局层级深，频繁 requestLayout。
- 自定义 View 的 `onDraw()` 分配对象或执行复杂逻辑。
- 频繁 GC，导致 Stop-The-World。
- 锁竞争或死锁。
- Binder 调用耗时，例如跨进程同步调用系统服务或业务服务。
- 大量 LiveData/Flow/Compose state 更新引起 UI 频繁刷新。

### 17. View 系统中如何减少布局和绘制成本？

- 减少布局层级，优先使用 ConstraintLayout 等扁平结构。
- 避免在滑动中频繁修改会触发布局的属性，如宽高、margin、padding。
- 动画优先使用 `translationX/Y`、`alpha`、`rotation` 等绘制属性。
- 自定义 View 中避免在 `onDraw()` 创建对象。
- 减少过度绘制，移除无意义背景。
- 使用 ViewStub 或懒加载非首屏区域。
- 合理使用 RecyclerView 复用、DiffUtil、ListAdapter、payload 局部刷新。

### 18. RecyclerView 卡顿怎么优化？

可从数据、布局、绑定、复用四个方面回答：

- 数据：使用 DiffUtil/ListAdapter，避免 `notifyDataSetChanged()` 全量刷新。
- 绑定：`onBindViewHolder()` 不做耗时计算、I/O、图片同步解码。
- 布局：item 层级尽量简单，避免嵌套 RecyclerView 造成复杂测量。
- 复用：合理设置 viewType，避免过多 viewType 降低复用率。
- 图片：指定合适尺寸，取消离屏请求，使用缩略图和缓存。
- 预取：合理使用 RecyclerView 预取机制，嵌套列表可调 `setInitialPrefetchItemCount()`。
- 局部刷新：使用 payload 只更新变化区域。

### 19. 过度绘制是什么？如何优化？

过度绘制是同一个像素在一帧内被重复绘制多次。它会增加 GPU 和内存带宽压力。

优化方式：

- 删除重复背景，例如 Activity、根布局、子布局都设置纯色背景。
- 扁平化层级，减少不必要的装饰 View。
- 对复杂重叠区域做裁剪，避免不可见内容绘制。
- 自定义 View 中只绘制可见区域。
- 使用开发者选项的“Debug GPU overdraw”观察。

### 20. 自定义 View 性能优化要注意什么？

- `onMeasure()`、`onLayout()`、`onDraw()` 都可能频繁执行，要避免耗时操作。
- 不要在 `onDraw()` 中创建 Paint、Path、Rect、Bitmap 等对象。
- 对复杂路径、文本测量、渐变等做缓存。
- 使用 `invalidate(Rect)` 控制局部刷新范围。
- 避免频繁 `requestLayout()`。
- 大图绘制前按目标尺寸采样，避免超大 Bitmap。

## 四、Compose 性能优化

### 21. Compose 性能优化的核心是什么？

核心是减少不必要的 recomposition、layout 和 draw，并让 Compose 能识别哪些输入是稳定的、哪些节点可以跳过。

常见原则：

- Composable 中少做昂贵计算。
- 使用 `remember` 缓存计算结果。
- Lazy 列表提供稳定 key。
- 快速变化状态使用 `derivedStateOf` 限制重组。
- 尽量延迟读取 state，把读取下沉到真正需要的地方。
- 高频变化的布局偏移使用 lambda 版本 modifier。
- 避免在组合期间写入已经读取过的 state。
- 发布版本开启 R8，并使用 Baseline Profile。

### 22. Compose 中 `remember` 的作用是什么？

`remember` 用于在 recomposition 之间保存计算结果，避免每次重组都重复执行昂贵计算。

例如列表排序、过滤、格式化等，如果写在 Composable body 中，滚动或状态变化时可能被反复执行。更好的方式是：

```kotlin
val sortedList = remember(list, comparator) {
    list.sortedWith(comparator)
}
```

如果计算本身属于业务逻辑，最好进一步放到 ViewModel 或 domain 层，而不是留在 UI 组合函数中。

### 23. Compose LazyColumn 为什么要设置 key？

Lazy 列表默认按位置识别 item。列表插入、删除、移动时，如果没有稳定 key，Compose 可能认为大量 item 发生变化，从而触发不必要的重组和状态错位。

使用稳定 key：

```kotlin
LazyColumn {
    items(items = messages, key = { it.id }) { message ->
        MessageItem(message)
    }
}
```

这样 item 移动后仍能保留对应状态，并减少无效 recomposition。

### 24. `derivedStateOf` 适合什么场景？

适合从高频变化状态中派生出低频变化结果。例如滚动位置每像素变化，但“是否显示回到顶部按钮”只在首个可见 item index 从 0 变为大于 0 时变化。

```kotlin
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

它不是通用缓存工具，只适合输入频繁变化、输出较少变化的场景。

### 25. Compose 稳定性 Stable/Unstable 对性能有什么影响？

Compose 如果能判断参数稳定，并且参数值没有变化，就可以跳过对应 Composable 的重组。若类型被判断为 unstable，Compose 往往更保守，容易导致更多重组。

优化方式：

- UI state 尽量使用不可变数据结构。
- data class 字段也保持不可变。
- 避免把可变集合 `MutableList` 直接作为状态传入。
- 使用稳定的回调引用。
- 对确实稳定的类型谨慎使用 `@Stable` 或 `@Immutable`，不要滥标。

## 五、内存优化

### 26. Android 内存优化关注哪些指标？

常见指标：

- Java/Kotlin heap 使用量。
- Native heap 使用量。
- Graphics/Bitmap 内存。
- PSS/RSS。
- GC 频率和单次 GC 时间。
- 内存峰值。
- OOM 次数。
- LMK 次数。
- 页面退出后对象是否释放。

面试中不要只说“防止 OOM”，更应该说明如何降低峰值、减少泄漏、减少 GC 抖动和降低被系统杀进程概率。

### 27. Android 的内存回收有什么特点？

ART 会自动进行垃圾回收，但前提是对象不再被强引用。Android 中被应用修改过的内存页通常会常驻 RAM，系统无法像桌面系统那样随意把应用内存换出。内存压力大时，系统会通过 LMK 杀掉优先级较低的进程。

所以优化内存的关键是：

- 减少不必要分配。
- 尽快释放不再使用的引用。
- 避免生命周期长对象持有短生命周期对象。
- 控制 Bitmap、缓存、WebView、Native 资源。

### 28. 常见内存泄漏有哪些？

- 静态变量持有 Activity、View、Context。
- 单例、Manager、Repository 持有页面对象。
- Handler、Runnable、Timer、Thread 未取消，间接持有 Activity。
- 注册监听、广播、观察者后未反注册。
- Dialog、PopupWindow、Toast 持有旧 Context。
- WebView 未正确销毁。
- Coroutine/Flow/RxJava 任务超过页面生命周期。
- Adapter、Drawable callback、动画对象持有 View。
- JNI/Native 资源未释放。

### 29. 如何排查内存泄漏？

- Android Studio Memory Profiler 抓 heap dump。
- 对比页面进入、退出、GC 后对象数量。
- 使用 LeakCanary 在开发和测试阶段自动发现泄漏。
- 查看引用链，确认 GC Root 到泄漏对象的路径。
- 对 Native 内存可用 heapprofd、Perfetto、Android Studio Native Memory Profiler。
- 对线上问题关注 OOM 堆栈、内存峰值、设备分布和页面路径。

### 30. Bitmap 内存如何优化？

- 按显示尺寸加载，使用 `inSampleSize` 或图片库 resize。
- 使用 WebP/AVIF 等更高压缩率格式。
- 列表中加载缩略图，详情页再加载大图。
- 避免把超大图一次性解码到内存。
- 使用成熟图片库的内存缓存和磁盘缓存。
- 页面销毁时取消图片请求。
- 注意硬件 Bitmap、ARGB_8888/RGB_565 的取舍。
- 避免在多处重复持有同一张大图。

### 31. 内存抖动是什么？如何优化？

内存抖动是短时间内大量对象频繁创建和销毁，导致 GC 频繁触发。GC 会暂停线程，严重时造成卡顿。

常见场景：

- `onDraw()` 中创建对象。
- 列表滑动时频繁创建临时对象。
- 字符串拼接、JSON 解析、日志格式化过多。
- 高频回调中创建 lambda、集合、数组。

优化方式：

- 对可复用对象做缓存或对象池。
- 把对象创建移出高频路径。
- 减少装箱拆箱和临时集合。
- 自定义 View 预创建 Paint/Path/Rect。

### 32. `onTrimMemory()` 有什么作用？

系统内存紧张时会回调 `onTrimMemory(level)`，应用可以根据级别释放缓存或降低内存占用。

常见处理：

- UI 不可见时释放 UI 相关缓存。
- 后台时降低图片内存缓存。
- 内存严重不足时清理非必要内存结构。

注意不要在回调里做太重的同步清理，否则可能引发卡顿。

### 33. LMK 是什么？如何降低被杀概率？

LMK 是 Low Memory Killer，系统内存压力大时会按进程优先级和 oom_adj_score 杀掉相对不重要的进程，通常后台进程先被杀。

降低风险：

- 控制后台内存占用。
- 释放不可见页面资源。
- 避免后台持有大 Bitmap、WebView、缓存。
- 避免滥用前台服务保活。
- 保存必要状态，支持进程重建。
- 关注 Android vitals 的 LMK 指标。

## 六、ANR、线程与并发

### 34. ANR 产生的典型条件有哪些？

常见触发条件：

- 前台应用 5 秒内没有响应输入事件。
- BroadcastReceiver 在规定时间内未执行完。
- Service 的 `onCreate()`、`onStartCommand()`、`onBind()` 执行过久。
- 调用 `startForegroundService()` 后没有在 5 秒内调用 `startForeground()`。
- JobService 的 `onStartJob()` 或 `onStopJob()` 未及时返回。

本质是主线程或系统期望的回调线程长时间无法完成工作。

### 35. ANR 常见原因有哪些？

- 主线程耗时计算或 I/O。
- 主线程等待锁，锁被子线程持有。
- Binder 同步调用阻塞。
- BroadcastReceiver 做耗时任务。
- CPU 被打满，主线程得不到调度。
- 死锁。
- 大量消息堆积，主线程 Looper 无法及时处理输入事件。
- GC 或内存压力导致长时间停顿。

### 36. 如何排查 ANR？

- 查看 Android vitals 或线上 ANR 平台的聚类。
- 分析 ANR traces，重点看 main 线程堆栈。
- 看 main 线程是在执行、等待锁、Binder 调用、I/O，还是 native 等待。
- 结合 Perfetto 看 ANR 前后的 CPU、调度、锁等待、Binder、I/O。
- 检查 BroadcastReceiver、Service、JobService 是否在回调中做重活。
- 用 StrictMode 在开发阶段发现主线程 I/O 和不合理调用。

### 37. 如何避免 ANR？

- 主线程只做 UI 和轻量逻辑。
- 网络、数据库、文件、JSON 解析、图片解码放到后台线程。
- BroadcastReceiver 中快速返回，必要时用 WorkManager 或 `goAsync()` 并确保 finish。
- 避免主线程等待子线程结果。
- 减少锁粒度，避免嵌套锁和反向加锁。
- Binder 调用尽量异步化，设置超时和降级。
- 前台服务启动后及时调用 `startForeground()`。
- 对核心线程池做监控，避免任务堆积。

### 38. 线程池如何设计更合理？

要根据任务类型隔离线程池：

- CPU 密集型：线程数接近 CPU 核心数。
- I/O 密集型：可以略多，但要限制上限。
- 单线程任务：用于有顺序要求的数据库、日志等。
- 高优先级任务：避免被低优先级任务阻塞。

不要把所有任务都丢到一个无限线程池，也不要随意 `new Thread()`。线程池要设置队列、拒绝策略、线程名、监控和生命周期。

### 39. StrictMode 有什么用？

StrictMode 是开发阶段发现性能问题的工具，可以检测：

- 主线程磁盘读写。
- 主线程网络。
- 资源未关闭。
- Activity 泄漏等部分问题。

它适合在 debug 包打开，帮助尽早发现“偶发卡顿”的根源，但不应把 StrictMode 发现问题等同于完整性能优化。

## 七、网络、I/O 与数据库优化

### 40. 网络性能优化有哪些方向？

- 减少请求次数：接口聚合、批量请求、去重、防抖。
- 降低数据量：gzip/br、精简字段、分页、增量更新。
- 缓存：HTTP 缓存、本地缓存、内存缓存。
- 连接复用：HTTP/2、连接池、DNS 缓存。
- 弱网优化：超时、重试、降级、离线可用。
- 图片优化：合适尺寸、WebP/AVIF、缩略图、懒加载。
- 后台网络收敛：避免后台频繁唤醒和移动网络消耗。

### 41. I/O 为什么容易造成卡顿？

磁盘 I/O 延迟不可控，可能受设备性能、系统负载、文件大小、闪存状态影响。主线程 I/O 会阻塞消息循环，导致输入、动画和绘制无法及时处理。

优化方式：

- 避免主线程读写文件、数据库、SharedPreferences 大文件。
- 批量写入，减少频繁小 I/O。
- 使用异步 API 和事务。
- 日志落盘异步化。
- 缓存热点数据，避免重复读取。
- 对关键 I/O 使用 trace 标记，便于 Perfetto 分析。

### 42. SharedPreferences 有哪些性能风险？

- 首次加载 XML 可能触发磁盘读。
- 大文件解析慢。
- `commit()` 同步写磁盘，会阻塞调用线程。
- 频繁写入会造成 I/O 压力。
- 多进程一致性差。

优化建议：

- 不存大对象和大量数据。
- 写入用 `apply()`，但仍要注意落盘时机。
- 启动关键路径避免首次读取大 SP。
- 大数据或复杂数据迁移到 DataStore/数据库。
- 按业务拆分文件，避免单个 SP 过大。

### 43. SQLite/Room 优化有哪些？

- 查询放后台线程。
- 建立合适索引，避免全表扫描。
- 分页加载，避免一次查出大量数据。
- 批量写入使用事务。
- 只查询必要字段。
- 避免频繁打开关闭数据库。
- 用 `EXPLAIN QUERY PLAN` 分析查询计划。
- 数据库迁移和清理要避开启动关键路径。

## 八、电量与后台优化

### 44. 电量优化主要关注什么？

电量优化关注 CPU、网络、定位、传感器、Wake Lock、Alarm、前台服务等资源是否在不必要的时候被唤醒或长时间占用。

Android vitals 中与电量相关的重点包括：

- excessive partial wake locks。
- stuck partial wake locks。
- excessive wakeups。
- excessive background network usage。
- excessive background Wi-Fi scans。

### 45. Wake Lock 为什么危险？

Partial Wake Lock 会在屏幕关闭后仍保持 CPU 运行。若后台长时间持有，会阻止设备进入低功耗状态，造成明显耗电。

优化方式：

- 能不用就不用，优先用 WorkManager、JobScheduler 等系统调度能力。
- 必须使用时设置超时。
- 在 `finally` 中释放。
- 用明确 tag，便于 Android vitals 归因。
- 页面不可见或任务结束立即释放。
- 避免后台长时间持有。

### 46. AlarmManager 频繁唤醒如何优化？

频繁使用 `RTC_WAKEUP` 或 `ELAPSED_REALTIME_WAKEUP` 会唤醒设备，增加耗电。

优化方式：

- 非精确任务使用非 wakeup alarm。
- 合并多个定时任务。
- 使用 WorkManager 处理可延迟后台任务。
- 避免高频轮询，改为服务端推送或事件驱动。
- 遵守 Doze 和 App Standby 限制。

### 47. Doze 和 App Standby 对性能有什么影响？

Doze 会在设备长时间未使用且未充电时限制后台 CPU 和网络活动；App Standby 会限制长时间未被用户使用的应用后台网络和任务。

面试重点：

- 后台任务不能假设随时执行。
- 非即时任务交给 WorkManager。
- 需要用户感知且立即执行的任务才考虑前台服务。
- 重要消息使用高优先级 FCM 时要谨慎，不能滥用。

### 48. 后台网络耗电如何优化？

- 减少后台轮询。
- 合并请求，批量同步。
- 只在 Wi-Fi 或充电时执行大同步。
- 使用缓存和增量同步。
- 避免应用在 cached/background 状态频繁拉起网络。
- 对失败重试做指数退避。
- 使用 Android vitals 观察后台移动网络过量使用。

## 九、包体积优化

### 49. 包体积为什么也属于性能优化？

包体积影响下载转化率、安装时间、磁盘占用、首次加载、内存映射和更新成本。大包在弱网和低端设备上尤其影响用户体验。

### 50. Android 包体积优化有哪些手段？

- 使用 Android App Bundle，让 Google Play 按设备生成优化 APK。
- 开启 R8：代码压缩、优化、混淆。
- 开启 `shrinkResources` 移除无用资源。
- 删除无用 assets、raw、so、语言资源、密度资源。
- 使用 WebP/AVIF，压缩 PNG/JPEG。
- 资源复用，能用 shape/vector drawable 就少放多套图片。
- ABI 拆分，避免用户下载不需要的 so。
- 动态特性模块，非核心功能按需下载。
- 检查第三方库体积，替换过重依赖。

### 51. R8 和 ProGuard 的关系是什么？

R8 是现在 Android 构建中默认的代码 shrinker、optimizer、obfuscator 和 dexer 相关工具链的一部分。它兼容 ProGuard 规则，但不只是混淆，还会做无用代码删除和优化。

面试中可以说：ProGuard 更多是规则和历史工具名，现代 Android 项目通常使用 R8 执行压缩、优化和混淆。

### 52. `shrinkResources` 为什么需要配合代码压缩？

资源是否“无用”很多时候依赖代码引用分析。构建时通常先由 R8 移除无用代码，再由 Android Gradle Plugin 根据剩余代码引用移除无用资源。因此 `shrinkResources` 需要配合 `minifyEnabled` 使用。

## 十、工具与监控

### 53. Android Studio Profiler 能分析什么？

- CPU Profiler：方法耗时、线程活动、调用栈。
- Memory Profiler：堆内存、对象分配、GC、heap dump。
- Network Profiler：请求、响应、流量。
- Energy Profiler：唤醒、定位、网络等耗电线索。

Profiler 适合开发阶段定位具体问题，但要注意 debug 包和 profiler 本身会带来额外开销，最终结论最好用 release 包和基准测试验证。

### 54. Perfetto 适合解决什么问题？

Perfetto 是系统级 tracing 工具，适合分析：

- 启动耗时。
- UI jank。
- 主线程阻塞。
- RenderThread/GPU 问题。
- Binder 调用。
- CPU 调度。
- 锁等待。
- I/O。
- GC。
- Native 内存。

它能从系统视角看“为什么这段时间没有跑到我的代码”。

### 55. Systrace 和 Perfetto 有什么区别？

Systrace 是较老的系统 trace 工具；Perfetto 是 Android 10 之后推荐的更现代 tracing 方案，数据源更丰富，能打开 Perfetto trace，也能兼容查看 Systrace 结果。

面试可以回答：现在优先用 Perfetto，老设备或历史链路中可能仍会看到 Systrace。

### 56. Macrobenchmark 和 Microbenchmark 有什么区别？

- Macrobenchmark：测试完整用户场景，例如启动、列表滚动、页面跳转、动画，能输出 trace，适合端到端性能。
- Microbenchmark：测试小段代码或函数级性能，例如算法、序列化、数据结构操作。

启动优化、滚动优化、Baseline Profile 验证通常用 Macrobenchmark。

### 57. Android vitals 有什么价值？

Android vitals 是 Google Play 提供的线上质量监控，可以看到真实用户设备上的稳定性、性能、电量和权限问题。

重点关注：

- user-perceived crash rate。
- user-perceived ANR rate。
- excessive partial wake locks。
- slow rendering。
- frozen frames。
- startup time。
- LMK。
- excessive wakeups。
- background network usage。

它的价值在于覆盖真实设备、真实系统版本和真实用户路径，是性能治理闭环的重要线上数据源。

### 58. 如何建立性能监控体系？

可以分层建设：

- 本地开发：StrictMode、Profiler、Perfetto、LeakCanary。
- 自动化测试：Macrobenchmark、启动耗时、滚动慢帧、包体积检查。
- 灰度阶段：启动、卡顿、内存、ANR、崩溃、电量指标监控。
- 线上阶段：Android vitals、APM、日志聚类、设备维度分析。
- 门禁机制：关键指标超过阈值阻止发布或触发告警。

好的性能体系要能回答三个问题：问题是否存在、影响多少用户、是哪次变更引入的。

## 十一、专项场景题

### 59. 首页启动慢，你会怎么优化？

回答模板：

1. 先测量：区分冷/温/热启动，记录 TTID、TTFD、首屏接口耗时。
2. 拆阶段：进程创建、Application、Activity、inflate、首帧、数据加载。
3. 查主线程：Perfetto 看是否有 I/O、锁、Binder、SDK 初始化。
4. 查布局：首屏层级、inflate、首帧 draw。
5. 查数据：接口是否串行、是否可缓存、是否可骨架屏先展示。
6. 优化：延迟初始化、异步并发、首屏最小化、缓存预取、Baseline Profile。
7. 回归：Macrobenchmark 和线上指标验证。

### 60. 列表滑动卡顿，你会怎么优化？

排查：

- 是否 `onBindViewHolder()` 耗时。
- 是否图片尺寸过大或同步解码。
- 是否 item 布局复杂。
- 是否频繁全量刷新。
- 是否嵌套滚动导致测量过重。
- 是否 GC 频繁。

优化：

- DiffUtil + payload。
- 异步计算和图片加载。
- 简化 item 布局。
- 固定尺寸，减少动态测量。
- 复用 ViewHolder，控制 viewType。
- 图片按需加载和取消。
- 对滚动场景做 Macrobenchmark。

### 61. 页面内存持续上涨，你会怎么处理？

步骤：

- 先确认是正常缓存增长、泄漏，还是 Native/Bitmap 增长。
- 重复进入退出页面，手动 GC 后抓 heap dump。
- 查看 Activity/View/ViewModel/Adapter 是否残留。
- 分析引用链，找到 GC Root。
- 检查监听、协程、Handler、单例、图片请求、WebView。
- 修复后重复验证，并加入泄漏检测。

### 62. 线上 ANR 增多，你会怎么处理？

- 看 ANR 类型和影响版本、设备、页面。
- 聚类 main 线程堆栈，找 Top 问题。
- 对高发路径本地复现并抓 Perfetto。
- 判断是主线程耗时、锁、Binder、I/O、CPU 饥饿还是广播/服务问题。
- 修复后灰度观察 user-perceived ANR rate。
- 对类似代码加 StrictMode、线程监控或超时保护。

### 63. 如何优化 WebView 性能？

- WebView 初始化很重，非首屏可延迟或预创建。
- 独立进程可降低主进程内存风险，但会增加进程成本。
- 开启合理缓存策略。
- 静态资源使用 CDN 和缓存。
- JS Bridge 调用要减少频率和数据量。
- 页面销毁时停止加载、移除 View、清理引用并调用 destroy。
- 避免多个页面同时持有重 WebView。

### 64. 如何优化图片加载性能？

- 根据目标 View 尺寸请求合适图片。
- 列表使用缩略图，避免原图。
- 使用内存缓存、磁盘缓存。
- 滑动中控制请求优先级，离屏取消。
- 预加载要有边界，避免抢占带宽和内存。
- 使用现代格式 WebP/AVIF。
- 避免主线程解码。

### 65. 如何做低端机性能优化？

- 使用真实低端设备建立基准。
- 降低首屏同步工作量。
- 减少动画和复杂阴影。
- 控制图片分辨率和内存缓存。
- 降低列表 item 复杂度。
- 减少后台任务和线程数。
- 做功能降级，例如关闭高成本特效。
- 按设备维度观察 Android vitals 和 APM 数据。

### 66. 如何防止性能劣化反复出现？

- 关键链路接入 Macrobenchmark。
- CI 检查启动耗时、慢帧、包体积。
- 代码评审关注主线程 I/O、启动期初始化、列表绑定耗时。
- 建立 SDK 初始化清单。
- 线上指标按版本、设备、页面聚类。
- 发布前灰度，指标异常自动告警。
- 性能问题修复后补测试或监控，避免只靠人肉记忆。

## 十二、常用指标速记

| 指标 | 含义 | 面试口径 |
| --- | --- | --- |
| TTID | 首帧显示时间 | 用户第一次看到 UI |
| TTFD | 完全绘制时间 | 页面核心内容可交互 |
| 16.67ms | 60Hz 单帧预算 | 超过可能掉帧 |
| 11ms/8ms | 90Hz/120Hz 单帧预算 | 高刷设备更严格 |
| Frozen Frame | 超过 700ms 的严重慢帧 | 用户明显感到卡住 |
| ANR 5s | 前台输入典型超时 | 主线程长时间无响应 |
| PSS | 按比例分摊后的内存 | 常用于观察进程实际内存压力 |
| LMK | 低内存杀进程 | 后台高内存更容易被杀 |
| Wake Lock | 保持 CPU 运行 | 后台滥用会耗电 |
| Baseline Profile | 关键路径预编译规则 | 优化首次启动和交互 |

## 十三、参考资料

- Android Developers: [App startup time](https://developer.android.com/topic/performance/vitals/launch-time)
- Android Developers: [Slow rendering and frozen frames](https://developer.android.com/topic/performance/vitals/frozen)
- Android Developers: [ANRs](https://developer.android.com/topic/performance/vitals/anr)
- Android Developers: [Android vitals](https://developer.android.com/topic/performance/vitals)
- Android Developers: [Baseline Profiles overview](https://developer.android.com/topic/performance/baselineprofiles/overview)
- Android Developers: [Write a Macrobenchmark](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview)
- Android Developers: [Overview of system tracing](https://developer.android.com/topic/performance/tracing/)
- Android Developers: [Perfetto](https://developer.android.com/tools/perfetto)
- Android Developers: [Manage your app's memory](https://developer.android.com/topic/performance/memory)
- Android Developers: [Overview of memory management](https://developer.android.com/topic/performance/memory-overview)
- Android Developers: [Low memory killers](https://developer.android.com/topic/performance/vitals/lmk)
- Android Developers: [Better performance through threading](https://developer.android.com/topic/performance/threads)
- Android Developers: [Jetpack Compose Performance](https://developer.android.com/develop/ui/compose/performance)
- Android Developers: [Compose performance best practices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- Android Developers: [Reduce your app size](https://developer.android.com/topic/performance/reduce-apk-size)
- Android Developers: [Optimize for Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby)
- Android Developers: [Excessive partial wake locks](https://developer.android.com/topic/performance/vitals/excessive-wakelock)
- Android Developers: [Excessive wakeups](https://developer.android.com/topic/performance/vitals/wakeup)
- Android Developers: [Excessive background network usage](https://developer.android.com/topic/performance/vitals/bg-network-usage)
