## jetpack之workManager官方demo解析

### 基础介绍

workmanager是一个可延期的后台异步任务，可以用来取代以前的android后台调度任务

通俗的讲就是可以用来做后台异步任务，那他有什么优势呢，和以前的后台api方法相比有什么区别呢？

带着这俩个问题开始往下看

### 兼容性

首先workmanager兼容性很好，包括 [FirebaseJobDispatcher](https://developer.android.google.cn/topic/libraries/architecture/workmanager/migrating-fb)、[GcmNetworkManager](https://developer.android.google.cn/topic/libraries/architecture/workmanager/migrating-gcm) 和 [JobScheduler](https://developer.android.google.cn/reference/android/app/job/JobScheduler)

都可以替换成workmanager，同时支持 API 级别 14，对电量续航也做了优化（省电）

### 基础功能

创建方式：通过单例创建workmanager，enqueue方法将任务加入队列

```kotlin
val myWorkRequest = ...
WorkManager.getInstance(myContext).enqueue(myWorkRequest)
```

任务（workrequest）是抽象基类，有俩个子类

OneTimeWorkRequest和PeriodicWorkRequest

`OneTimeWorkRequest` 适用于调度非重复性工作，而 `PeriodicWorkRequest` 则更适合调度以一定间隔重复执行的工作。

一次性工作：

```kotlin
val myWorkRequest = OneTimeWorkRequest.from(MyWork::class.java)
```

如果是复杂一点的一次性工作可以用构建器，可以在这里对你的任务添加一些策略

```kotlin
val uploadWorkRequest: WorkRequest =
   OneTimeWorkRequestBuilder<MyWork>()
       // Additional configuration
       .build()
```

定期性工作

```kotlin
val saveRequest =
       PeriodicWorkRequestBuilder<SaveImageToFileWorker>(1, TimeUnit.HOURS)
    // Additional configuration
           .build()
//工作的运行时间间隔定为一小时。
//可以定义的最短重复间隔是 15 分钟
```

request由work类组成。work类就是具体执行流程的类

如下TestWork类继承Workr类，实现doWork方法，在里面上传图片

```kotlin
class TestWorker(appContext: Context, workerParams: WorkerParameters):
       Worker(appContext, workerParams) {
   override fun doWork(): Result {

       // Do the work here--in this case, upload the images.
       uploadImages()

       // Indicate whether the work finished successfully with the Result
       return Result.success()
   }
}
从 doWork() 返回的 Result 会通知 WorkManager 服务工作是否成功，以及工作失败时是否应重试工作。
Result.success()：工作成功完成。
Result.failure()：工作失败。
Result.retry()：工作失败，应根据其重试政策在其他时间尝试。
```

到这里，一个简单的workmanager就介绍完了，包括workmanager的创建方式，request的组成，work的创建



接下来是workmanger的一些配置

### 约束

可以给任务添加约束（比如在wifi条件下才运行，在充电状态下才运行）

```kotlin
val constraints = Constraints.Builder()
   .setRequiredNetworkType(NetworkType.UNMETERED)
   .setRequiresCharging(true)
   .build()

val myWorkRequest: WorkRequest =
   OneTimeWorkRequestBuilder<MyWork>()
       .setConstraints(constraints)
       .build()
//可以添加的约束类型如下
```

![image-20201010102129167](/Users/yanzhe/android/知识整理/jetpack/图片/image-20201010102129167.png)



### 延迟工作

下面的示例为延迟10分钟

```kotlin
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .setInitialDelay(10, TimeUnit.MINUTES)
   .build()
```



### 调度

对于一组完整的异步任务，可以运行一次，也可以重复运行，对每一组任务进行命名，用来单独操作他们

```kotlin
//给一个单独的work添加标识
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .addTag("cleanup")
   .build()
//可以通过WorkManager.cancelAllWorkByTag(String)取消任务
//可以通过WorkManager.getWorkInfosByTag(String)获取这个任务，得到一个WorkInfo对象
//可以通过workManager.getWorkInfosByTagLiveData(String)得到一个livedata，当这个对象的tag的work执行完成之后，这个livedata会触发 
```



### 灵活的重试机制

LINEAR和EXPONENTIAL俩种策略

当你在work的dowork方法中返回Result.retry()的时候，会进行重试机制

```kotlin
val myWorkRequest = OneTimeWorkRequestBuilder<MyWork>()
   .setBackoffCriteria(
       BackoffPolicy.LINEAR,
       OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
       TimeUnit.MILLISECONDS)
   .build()
//MIN_BACKOFF_MILLIS为10秒
//LINEAR策略，每次尝试重试的时候，重试间隔都会增加10秒，20，30，40类推
//如果是EXPONENTIAL策略，那么重试时长是20、40、80 秒，以此类推。
```



### work中的数据传递

```kotlin
// Define the Worker requiring input
class UploadWork(appContext: Context, workerParams: WorkerParameters)
   : Worker(appContext, workerParams) {

   override fun doWork(): Result {
       val imageUriInput =
           inputData.getString("IMAGE_URI") ?: return Result.failure()

       uploadFile(imageUriInput)
       return Result.success()
   }
   ...
}

// Create a WorkRequest for your Worker and sending it input
val myUploadWork = OneTimeWorkRequestBuilder<UploadWork>()
   .setInputData(workDataOf(
       "IMAGE_URI" to "http://..."
   ))
   .build()

//还可以通过 Result.success(outputData)将数据传递给下一个work
```



### 工作状态

#### 一次性工作状态



对于 [`one-time`](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/define-work#schedule_one-time_work) 工作请求，工作的初始状态为 [`ENQUEUED`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#ENQUEUED)。

在 `ENQUEUED` 状态下，您的工作会在满足其 [`Constraints`](https://developer.android.google.cn/reference/androidx/work/Constraints) 和初始延迟计时要求后立即运行。接下来，该工作会转为 [`RUNNING`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#RUNNING) 状态，然后可能会根据工作的结果转为 [`SUCCEEDED`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#SUCCEEDED)、[`FAILED`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#FAILED) 状态；或者，如果结果是 [`retry`](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/reference/androidx/work/ListenableWorker.Result#retry())，它可能会回到 `ENQUEUED` 状态。在此过程中，随时都可以取消工作，取消后工作将进入 [`CANCELLED`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#CANCELLED) 状态。

![image-20201010141544051](/Users/yanzhe/android/知识整理/jetpack/图片/image-20201010141544051.png)

`SUCCEEDED`、`FAILED` 和 `CANCELLED` 均表示此工作的终止状态。如果您的工作处于上述任何状态，[`WorkInfo.State.isFinished()`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State#isFinished()) 都将返回 true。

#### 定期工作的状态

成功和失败状态仅适用于一次性工作和[链式工作](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/chain-work)。[定期工作](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/define-work#schedule_periodic_work)只有一个终止状态 `CANCELLED`。这是因为定期工作永远不会结束。每次运行后，无论结果如何，系统都会重新对其进行调度。图 2 描述了定期工作的精简状态图。

![image-20201010141711434](/Users/yanzhe/android/知识整理/jetpack/图片/image-20201010141711434.png)



### 唯一工作

确保在某一个时刻只有一个相同的实例存在

- [`WorkManager.enqueueUniqueWork()`](https://developer.android.google.cn/reference/androidx/work/WorkManager#enqueueUniqueWork(java.lang.String, androidx.work.ExistingWorkPolicy, androidx.work.OneTimeWorkRequest))（用于一次性工作）
- [`WorkManager.enqueueUniquePeriodicWork()`](https://developer.android.google.cn/reference/androidx/work/WorkManager#enqueueUniquePeriodicWork(java.lang.String, androidx.work.ExistingPeriodicWorkPolicy, androidx.work.PeriodicWorkRequest))（用于定期工作）

这两种方法都接受 3 个参数：

- uniqueWorkName - 用于唯一标识工作请求的 `String`。
- existingWorkPolicy - 此 `enum` 可告知 WorkManager 如果已有使用该名称且尚未完成的唯一工作链，应执行什么操作。如需了解详情，请参阅[冲突解决政策](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/managing-work#conflict-resolution)。
- work - 要调度的 `WorkRequest`。

```kotlin
val sendLogsWorkRequest =
       PeriodicWorkRequestBuilder<SendLogsWorker>(24, TimeUnit.HOURS)
           .setConstraints(Constraints.Builder()
               .setRequiresCharging(true)
               .build()
            )
           .build()
WorkManager.getInstance(this).enqueueUniquePeriodicWork(
           "sendLogs",
           ExistingPeriodicWorkPolicy.KEEP,
           sendLogsWorkRequest
)
```

上述代码在 sendLogs 作业已处于队列中的情况下运行，系统会保留现有的作业，并且不会添加新的作业

ExistingPeriodicWorkPolicy有三种选项

- [`REPLACE`](https://developer.android.google.cn/reference/androidx/work/ExistingWorkPolicy#REPLACE)：用新工作替换现有工作。此选项将取消现有工作。
- [`KEEP`](https://developer.android.google.cn/reference/androidx/work/ExistingWorkPolicy#KEEP)：保留现有工作，并忽略新工作。
- [`APPEND`](https://developer.android.google.cn/reference/androidx/work/ExistingWorkPolicy#APPEND)：将新工作附加到现有工作的末尾。此政策将导致您的新工作[链接](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/chain-work)到现有工作，在现有工作完成后运行。

- [`APPEND_OR_REPLACE`](https://developer.android.google.cn/reference/androidx/work/ExistingWorkPolicy#APPEND) 功能类似于 `APPEND`，不过它并不依赖于**先决条件**工作状态。即使现有工作变为 `CANCELLED` 或 `FAILED` 状态，新工作仍会运行。

### 状态监听

可以通过三种方式获取查询work

```kotlin
// by id
workManager.getWorkInfoById(syncWorker.id) // ListenableFuture<WorkInfo>

// by name
workManager.getWorkInfosForUniqueWork("sync") // ListenableFuture<List<WorkInfo>>

// by tag
workManager.getWorkInfosByTag("syncTag") // ListenableFuture<List<WorkInfo>>
```

该查询会返回 [`WorkInfo`](https://developer.android.google.cn/reference/androidx/work/WorkInfo) 对象的 [`ListenableFuture`](https://guava.dev/releases/23.1-android/api/docs/com/google/common/util/concurrent/ListenableFuture.html)，该值包含工作的 [`id`](https://developer.android.google.cn/reference/androidx/work/WorkInfo#getId())、其标记、其当前的 [`State`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.State) 以及通过 [`Result.success(outputData)`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker.Result#success(androidx.work.Data)) 设置的任何输出数据

下面的代码展示了，当syncWorker完成后，显示一个消息

```kotlin
workManager.getWorkInfoByIdLiveData(syncWorker.id)
               .observe(viewLifecycleOwner) { workInfo ->
   if(workInfo?.state == WorkInfo.State.SUCCEEDED) {
       Snackbar.make(requireView(),
      R.string.work_completed, Snackbar.LENGTH_SHORT)
           .show()
   }
}
```

复杂的work状态查询

以下示例说明了如何查找带有“syncTag”标记、处于 `FAILED` 或 `CANCELLED` 状态，且唯一工作名称为“preProcess”或“sync”的所有工作。

```kotlin
val workQuery = WorkQuery.Builder
       .fromTags(listOf("syncTag"))
       .addStates(listOf(WorkInfo.State.FAILED, WorkInfo.State.CANCELLED))
       .addUniqueWorkNames(listOf("preProcess", "sync")
    )
   .build()

val workInfos: ListenableFuture<List<WorkInfo>> = workManager.getWorkInfos(workQuery)
```

```
WorkQuery` 中的每个组件（标记、状态或名称）与其他组件都是 `AND` 逻辑关系。组件中的每个值都是 `OR` 逻辑关系。例如：`(name1 OR name2 OR ...) AND (tag1 OR tag2 OR ...) AND (state1 OR state2 OR ...)
```

`WorkQuery` 也适用于等效的 LiveData 方法 [`getWorkInfosLiveData()`](https://developer.android.google.cn/reference/androidx/work/WorkManager#getWorkInfosLiveData(androidx.work.WorkQuery))

### 取消和停止工作

```kotlin
// by id
workManager.cancelWorkById(syncWorker.id)

// by name
workManager.cancelUniqueWork("sync")

// by tag
workManager.cancelAllWorkByTag("syncTag")
```

### 观察进度

在work的dowork方法中通过

```kotlin
setProgress(firstUpdate)//设置进度
```

观察进度

```kotlin
WorkManager.getInstance(applicationContext)
        // requestId is the WorkRequest id
        .getWorkInfoByIdLiveData(requestId)
        .observe(observer, Observer { workInfo: WorkInfo? ->
                if (workInfo != null) {
                    val progress = workInfo.progress
                    val value = progress.getInt(Progress, 0)
                    // Do something with progress information
                }
        })
```

### 链接工作

您可以使用 WorkManager 创建工作链并将其加入队列。工作链用于指定多个依存任务并定义这些任务的运行顺序。当您需要以特定顺序运行多个任务时，此功能尤其有用。

如需创建工作链，您可以使用 [`WorkManager.beginWith(OneTimeWorkRequest)`](https://developer.android.google.cn/reference/androidx/work/WorkManager#beginWith(androidx.work.OneTimeWorkRequest)) 或 [`WorkManager.beginWith(List)`](https://developer.android.google.cn/reference/androidx/work/WorkManager#beginWith(java.util.List))，这会返回 [`WorkContinuation`](https://developer.android.google.cn/reference/androidx/work/WorkContinuation) 实例。

然后，可以使用 `WorkContinuation` 通过 [`then(OneTimeWorkRequest)`](https://developer.android.google.cn/reference/androidx/work/WorkContinuation#then(androidx.work.OneTimeWorkRequest)) 或 [`then(List)`](https://developer.android.google.cn/reference/androidx/work/WorkContinuation#then(java.util.List)) 添加依存 `OneTimeWorkRequest`。 .

每次调用 `WorkContinuation.then(...)` 都会返回一个新的 `WorkContinuation` 实例。如果添加了 `OneTimeWorkRequest` 实例的 `List`，这些请求可能会并行运行。

最后，您可以使用 [`WorkContinuation.enqueue()`](https://developer.android.google.cn/reference/androidx/work/WorkContinuation#enqueue()) 方法对 `WorkContinuation` 工作链执行 `enqueue()` 操作。

下面我们来看一个示例。在本例中，有 3 个不同的工作器作业配置为运行（可能并行运行）。然后这些工作器的结果将联接起来，并传递给正在缓存的工作器作业。最后，该作业的输出将传递到上传工作器，由上传工作器将结果上传到远程服务器。

```kotlin
WorkManager.getInstance(myContext)
   // Candidates to run in parallel
   .beginWith(listOf(plantName1, plantName2, plantName3))
   // Dependent work (only runs after all previous work in chain)
   .then(cache)
   .then(upload)
   // Call enqueue to kick things off
   .enqueue()
```

如果想将plantName1, plantName2, plantName3work的输出作为下一个work cache的输入。那么就需要用到输入合并器

WorkManager 提供两种不同类型的 `InputMerger`：

- [`OverwritingInputMerger`](https://developer.android.google.cn/reference/androidx/work/OverwritingInputMerger) 会尝试将所有输入中的所有键添加到输出中。如果发生冲突，它会覆盖先前设置的键。
- [`ArrayCreatingInputMerger`](https://developer.android.google.cn/reference/androidx/work/ArrayCreatingInputMerger) 会尝试合并输入，并在必要时创建数组。

也可以创建 `InputMerger` 的子类来编写自己的用例。

by the last最后说一下

请注意，[`Worker.doWork()`](https://developer.android.google.cn/reference/androidx/work/Worker#doWork()) 是同步调用 - 您将会以阻塞方式完成整个后台工作，并在方法退出时完成工作。如果您在 `doWork()` 中调用异步 API 并返回 [`Result`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker.Result)，则回调可能无法正常运行。如果您遇到这种情况，请考虑使用 [`ListenableWorker`](https://developer.android.google.cn/reference/androidx/work/ListenableWorker)（请参阅[在 ListenableWorker 中进行线程处理](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced/listenableworker)）。