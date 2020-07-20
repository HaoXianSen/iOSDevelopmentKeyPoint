#### 1. 前言

------

Flutter 亚秒级别的热重载是开发者的神兵利器，它能够提供我们快速修改UI、增加功能、修复bug，不需要重新启动应用，就可以看到改动效果。那么Flutter是如何做到这样的功能的呢？下面我们一起探究一下其中的原理。

#### 2. 首先需要知道JIT 和AOT

------

##### JIT

JIT（Just In Time），指的是编译运行时编译，Flutter 在debug模式下采用这种方式，在运行时动态下发和执行代码，启动速度快，但是由于在运行时编译，性能会受影响。

![img](https://pic2.zhimg.com/80/v2-5ac75ecfd558061556120bad41866ddd_1440w.jpg)

##### AOT

AOT（Ahead Of Time），指的是在运行之前进行编译，Flutter 在Release模式下采用，可以为特定的平台生成稳定的二进制代码、执行性能好、运行速度快，但是每次都需要重新运行编译，开发效率低。

![img](https://pic4.zhimg.com/80/v2-494be33d7893b43cc45146974b49f0c7_1440w.jpg)

所以，Flutter提供的两种编译模式中，AOT 是静态编译，编译成设备直接可执行的二进制代码；而JIT 则是先生成中间代码（Script snapshot），然后通过dart Vm 解释执行。

#### 3. Hot Reload 原理

------

1. 扫描修改的文件；
2. 生成kernal file（app.dill.incremental.dill）；
3. 通过Http协议下发到dartVM；
4. VM服务通过RPC 调用_reloadSources，进行资源加载；
5. VM 资源加载成功，将FlutterDevice UI线程重置（uiIsolate），通过RPC调用，触发Flutter树的重建、重绘。

如下图形象说明：

![img](https://www.yuque.com/api/filetransfer/images?url=http%3A%2F%2Fgw.alicdn.com%2Fmt%2FTB1Qe3hisfpK1RjSZFOXXa6nFXa-895-477.png&sign=e8bfac5a86b2ca4bce992c9cdd10c865d27516def919ce6fb7afee3f7cb41bd4)

#### 4. 源码分析

------

​	通过命令行输入r或者触发闪电按钮，其实会触发flutter_tools的run_hot文件的HotRunner的restart() 方法（位于flutter/pakages/flutter_tools/）；

​	flutter_tools 调试步骤：

​	1）用AS打开Flutter_tools，打开 Run/Debug Configurations;	![image-20200706151828238](/Users/ddreader/Library/Application Support/typora-user-images/image-20200706151828238.png)

​	2) 添加Dart Command Line App， Name 设置为Flutter Tools Debugger， Dart file 设置为flutter_tools的文件目录，Working directory 设置为一个测试项目路径

​	3) 运行Debug，运行完成后，断点打在HotRunner 的restart（）方法处。

​	4）随便修改测试项目的Widget，在flutter_tools 工程的Console输入r回车；发现断点断在了restart方法。下面我们从restart（）方法开始逐步分析，热重载的原理。

1. restart（）方法

```dart
@override
Future<OperationResult> restart({
  bool fullRestart = false,
  bool pauseAfterRestart = false,
  String reason,
  bool benchmarkMode = false
}) async {
  String targetPlatform;
  String sdkName;
  bool emulator;
  // 判断当前的设备，并且获取当前设备的targetPlatform、sdkName、emulator
  if (flutterDevices.length == 1) {
    final Device device = flutterDevices.first.device;
    targetPlatform = getNameForTargetPlatform(await device.targetPlatform);
    sdkName = await device.sdkNameAndVersion;
    emulator = await device.isLocalEmulator;
  } else if (flutterDevices.length > 1) {
    targetPlatform = 'multiple';
    sdkName = 'multiple';
    emulator = false;
  } else {
    targetPlatform = 'unknown';
    sdkName = 'unknown';
    emulator = false;
  }
  final Stopwatch timer = Stopwatch()..start();
  if (fullRestart) {
    	... 完全重新启动、非热重载部分省略
      return;
  }
  final OperationResult result = await _hotReloadHelper(
    targetPlatform: targetPlatform,
    sdkName: sdkName,
    emulator: emulator,
    reason: reason,
    pauseAfterRestart: pauseAfterRestart,
  );
  if (result.isOk) {
    final String elapsed = getElapsedAsMilliseconds(timer.elapsed);
    printStatus('${result.message} in $elapsed.');
  }
  return result;
}
```

​	调用到restart方法之后，会获取当前设备的一些信息，然后根据是否是fullRestart执行不同流程，我们研究的是Hot reload不是fullRestart所以我们走的是下边的流程，执行_hotReloadHelper（）；

 2. _hotReloadHelper

    ```dart
     Future<OperationResult> _hotReloadHelper({
        String targetPlatform,
        String sdkName,
        bool emulator,
        String reason,
        bool pauseAfterRestart = false,
      }) async {
        final bool reloadOnTopOfSnapshot = _runningFromSnapshot;
        final String progressPrefix = reloadOnTopOfSnapshot ? 'Initializing' : 'Performing';
        Status status = logger.startProgress(
          '$progressPrefix hot reload...',
          timeout: timeoutConfiguration.fastOperation,
          progressId: 'hot.reload',
        );
        OperationResult result;
        try {
          result = await _reloadSources(
            targetPlatform: targetPlatform,
            sdkName: sdkName,
            emulator: emulator,
            pause: pauseAfterRestart,
            reason: reason,
            onSlow: (String message) {
              status?.cancel();
              status = logger.startProgress(
                message,
                timeout: timeoutConfiguration.slowOperation,
                progressId: 'hot.reload',
              );
            },
          );
        } on rpc.RpcException {
          HotEvent('exception',
            targetPlatform: targetPlatform,
            sdkName: sdkName,
            emulator: emulator,
            fullRestart: false,
            reason: reason).send();
          return OperationResult(1, 'hot reload failed to complete', fatal: true);
        } finally {
          status.cancel();
        }
        return result;
      }
    ```

    这个方法呢，其实也是对于核心逻辑方法_reloadSources（）方法的封装。

 3. _reloadSources()

    ```dart
     Future<OperationResult> _reloadSources({
        String targetPlatform,
        String sdkName,
        bool emulator,
        bool pause = false,
        String reason,
        void Function(String message) onSlow
      }) async {
       // 遍历所有Device的FlutterView, 并且uiIsolate 不存在的话，执行失败
        for (FlutterDevice device in flutterDevices) {
          for (FlutterView view in device.views) {
            if (view.uiIsolate == null) {
              return OperationResult(2, 'Application isolate not found', fatal: true);
            }
          }
        }
        bool shouldReportReloadTime = !_runningFromSnapshot;
       // 开启计时器
        final Stopwatch reloadTimer = Stopwatch()..start();
    		// 判断FlutterView的uiIsolate是否为null或者处于暂停状态，如果是刷新所有view
        if (!_isPaused()) {
          printTrace('Refreshing active FlutterViews before reloading.');
          // 触发所有设备的vmService RPC 发送_flutter.listViews消息
          await refreshViews();
        }
    
        final Stopwatch devFSTimer = Stopwatch()..start();
       	// 同步修改的文件
        final UpdateFSReport updatedDevFS = await _updateDevFS();
        // Record time it took to synchronize to DevFS.
        _addBenchmarkData('hotReloadDevFSSyncMilliseconds', devFSTimer.elapsed.inMilliseconds);
        if (!updatedDevFS.success) {
          return OperationResult(1, 'DevFS synchronization failed');
        }
        String reloadMessage;
        final Stopwatch vmReloadTimer = Stopwatch()..start();
        Map<String, dynamic> firstReloadDetails;
        try {
          final String entryPath = fs.path.relative(
            getReloadPath(fullRestart: false),
            from: projectRootPath,
          );
          final List<Future<DeviceReloadReport>> allReportsFutures = <Future<DeviceReloadReport>>[];
          for (FlutterDevice device in flutterDevices) {
            if (_runningFromSnapshot) {
              // Asset directory has to be set only once when we switch from
              // running from snapshot to running from uploaded files.
              await device.resetAssetDirectory();
            }
            final Completer<DeviceReloadReport> completer = Completer<DeviceReloadReport>();
            allReportsFutures.add(completer.future);
            // 触发RPC 调用_reloadSources 重新加载资源
            final List<Future<Map<String, dynamic>>> reportFutures = device.reloadSources(
              entryPath, pause: pause,
            );
            // 处理RPC 调用_reloadSources 返回结果
            unawaited(Future.wait(reportFutures).then(
              (List<Map<String, dynamic>> reports) async {
                // TODO(aam): Investigate why we are validating only first reload report,
                // which seems to be current behavior
                final Map<String, dynamic> firstReport = reports.first;
                // Don't print errors because they will be printed further down when
                // `validateReloadReport` is called again.
                await device.updateReloadStatus(
                  validateReloadReport(firstReport, printErrors: false),
                );
                completer.complete(DeviceReloadReport(device, reports));
              },
            ));
          }
          final List<DeviceReloadReport> reports = await Future.wait(allReportsFutures);
          for (DeviceReloadReport report in reports) {
            final Map<String, dynamic> reloadReport = report.reports[0];
            if (!validateReloadReport(reloadReport)) {
              // Reload failed.
              HotEvent('reload-reject',
                targetPlatform: targetPlatform,
                sdkName: sdkName,
                emulator: emulator,
                fullRestart: false,
                reason: reason,
              ).send();
              return OperationResult(1, 'Reload rejected');
            }
            // Collect stats only from the first device. If/when run -d all is
            // refactored, we'll probably need to send one hot reload/restart event
            // per device to analytics.
            firstReloadDetails ??= reloadReport['details'];
            final int loadedLibraryCount = reloadReport['details']['loadedLibraryCount'];
            final int finalLibraryCount = reloadReport['details']['finalLibraryCount'];
            printTrace('reloaded $loadedLibraryCount of $finalLibraryCount libraries');
            reloadMessage = 'Reloaded $loadedLibraryCount of $finalLibraryCount libraries';
          }
        } on Map<String, dynamic> catch (error, stackTrace) {
          printTrace('Hot reload failed: $error\n$stackTrace');
          final int errorCode = error['code'];
          String errorMessage = error['message'];
          if (errorCode == Isolate.kIsolateReloadBarred) {
            errorMessage = 'Unable to hot reload application due to an unrecoverable error in '
                           'the source code. Please address the error and then use "R" to '
                           'restart the app.\n'
                           '$errorMessage (error code: $errorCode)';
            HotEvent('reload-barred',
              targetPlatform: targetPlatform,
              sdkName: sdkName,
              emulator: emulator,
              fullRestart: false,
              reason: reason,
            ).send();
            return OperationResult(errorCode, errorMessage);
          }
          return OperationResult(errorCode, '$errorMessage (error code: $errorCode)');
        } catch (error, stackTrace) {
          printTrace('Hot reload failed: $error\n$stackTrace');
          return OperationResult(1, '$error');
        }
        // Record time it took for the VM to reload the sources.
        _addBenchmarkData('hotReloadVMReloadMilliseconds', vmReloadTimer.elapsed.inMilliseconds);
        final Stopwatch reassembleTimer = Stopwatch()..start();
        // Reload the isolate.
        final List<Future<void>> allDevices = <Future<void>>[];
       // 触发所有的FlutterView 的uiIsolate 刷新
        for (FlutterDevice device in flutterDevices) {
          printTrace('Sending reload events to ${device.device.name}');
          final List<Future<ServiceObject>> futuresViews = <Future<ServiceObject>>[];
          for (FlutterView view in device.views) {
            printTrace('Sending reload event to "${view.uiIsolate.name}"');
            futuresViews.add(view.uiIsolate.reload());
          }
          final Completer<void> deviceCompleter = Completer<void>();
          unawaited(Future.wait(futuresViews).whenComplete(() {
            deviceCompleter.complete(device.refreshViews());
          }));
          allDevices.add(deviceCompleter.future);
        }
        await Future.wait(allDevices);
        // We are now running from source.
        _runningFromSnapshot = false;
        // Check if any isolates are paused.
        final List<FlutterView> reassembleViews = <FlutterView>[];
        String serviceEventKind;
        int pausedIsolatesFound = 0;
       	// 添加 ressembleViews
        for (FlutterDevice device in flutterDevices) {
          for (FlutterView view in device.views) {
            // Check if the isolate is paused, and if so, don't reassemble. Ignore the
            // PostPauseEvent event - the client requesting the pause will resume the app.
            final ServiceEvent pauseEvent = view.uiIsolate.pauseEvent;
            if (pauseEvent != null && pauseEvent.isPauseEvent && pauseEvent.kind != ServiceEvent.kPausePostRequest) {
              pausedIsolatesFound += 1;
              if (serviceEventKind == null) {
                serviceEventKind = pauseEvent.kind;
              } else if (serviceEventKind != pauseEvent.kind) {
                serviceEventKind = ''; // many kinds
              }
            } else {
              reassembleViews.add(view);
            }
          }
        }
        if (pausedIsolatesFound > 0) {
          if (onSlow != null)
            onSlow('${_describePausedIsolates(pausedIsolatesFound, serviceEventKind)}; interface might not update.');
          if (reassembleViews.isEmpty) {
            printTrace('Skipping reassemble because all isolates are paused.');
            return OperationResult(OperationResult.ok.code, reloadMessage);
          }
        }
       //删除dirty 资源
        printTrace('Evicting dirty assets');
        await _evictDirtyAssets();
        assert(reassembleViews.isNotEmpty);
        printTrace('Reassembling application');
        bool failedReassemble = false;
        final List<Future<void>> futures = <Future<void>>[];
       // rpc触发所有的FlutterView uiIsolate ressemble
        for (FlutterView view in reassembleViews) {
          futures.add(() async {
            try {
              await view.uiIsolate.flutterReassemble();
            } catch (error) {
              failedReassemble = true;
              printError('Reassembling ${view.uiIsolate.name} failed: $error');
              return;
            }
          }());
        }
        final Future<void> reassembleFuture = Future.wait<void>(futures).then<void>((List<void> values) { });
        await reassembleFuture.timeout(
          const Duration(seconds: 2),
          onTimeout: () async {
            if (pausedIsolatesFound > 0) {
              shouldReportReloadTime = false;
              return; // probably no point waiting, they're probably deadlocked and we've already warned.
            }
            // Check if any isolate is newly paused.
            printTrace('This is taking a long time; will now check for paused isolates.');
            int postReloadPausedIsolatesFound = 0;
            String serviceEventKind;
            for (FlutterView view in reassembleViews) {
              await view.uiIsolate.reload();
              final ServiceEvent pauseEvent = view.uiIsolate.pauseEvent;
              if (pauseEvent != null && pauseEvent.isPauseEvent) {
                postReloadPausedIsolatesFound += 1;
                if (serviceEventKind == null) {
                  serviceEventKind = pauseEvent.kind;
                } else if (serviceEventKind != pauseEvent.kind) {
                  serviceEventKind = ''; // many kinds
                }
              }
            }
            printTrace('Found $postReloadPausedIsolatesFound newly paused isolate(s).');
            if (postReloadPausedIsolatesFound == 0) {
              await reassembleFuture; // must just be taking a long time... keep waiting!
              return;
            }
            shouldReportReloadTime = false;
            if (onSlow != null)
              onSlow('${_describePausedIsolates(postReloadPausedIsolatesFound, serviceEventKind)}.');
          },
        );
        // Record time it took for Flutter to reassemble the application.
        _addBenchmarkData('hotReloadFlutterReassembleMilliseconds', reassembleTimer.elapsed.inMilliseconds);
    
        reloadTimer.stop();
        final Duration reloadDuration = reloadTimer.elapsed;
        final int reloadInMs = reloadDuration.inMilliseconds;
    
        // Collect stats that help understand scale of update for this hot reload request.
        // For example, [syncedLibraryCount]/[finalLibraryCount] indicates how
        // many libraries were affected by the hot reload request.
        // Relation of [invalidatedSourcesCount] to [syncedLibraryCount] should help
        // understand sync/transfer "overhead" of updating this number of source files.
        HotEvent('reload',
          targetPlatform: targetPlatform,
          sdkName: sdkName,
          emulator: emulator,
          fullRestart: false,
          reason: reason,
          overallTimeInMs: reloadInMs,
          finalLibraryCount: firstReloadDetails['finalLibraryCount'],
          syncedLibraryCount: firstReloadDetails['receivedLibraryCount'],
          syncedClassesCount: firstReloadDetails['receivedClassesCount'],
          syncedProceduresCount: firstReloadDetails['receivedProceduresCount'],
          syncedBytes: updatedDevFS.syncedBytes,
          invalidatedSourcesCount: updatedDevFS.invalidatedSourcesCount,
          transferTimeInMs: devFSTimer.elapsed.inMilliseconds,
        ).send();
    
        if (shouldReportReloadTime) {
          printTrace('Hot reload performed in ${getElapsedAsMilliseconds(reloadDuration)}.');
          // Record complete time it took for the reload.
          _addBenchmarkData('hotReloadMillisecondsToFrame', reloadInMs);
        }
        // Only report timings if we reloaded a single view without any errors.
        if ((reassembleViews.length == 1) && !failedReassemble && shouldReportReloadTime)
          flutterUsage.sendTiming('hot', 'reload', reloadDuration);
        return OperationResult(
          failedReassemble ? 1 : OperationResult.ok.code,
          reloadMessage,
        );
      }
    ```


​	这个方法主要干了 5件事情

	1. 扫描修改的文件，生成dill文件，并且通过Http服务下发到设备资源文件；
 	2. RPC 调用_reloadSources 触发VM重新加载修改后的文件
 	3. RPC 调用 flutter View的uiIsolate refreshView
 	4. 删除dirty文件
 	5. rpc触发所有的FlutterView uiIsolate ressemble



#### 5. 不会发生Hot Reload的场景

----

 1. 应用被杀死

 2. 编译错误

    当代码更改导致编译错误时，热重载会生成类似于以下内容的错误消息：

    ```dart
    Hot reload was rejected:
    '/Users/obiwan/Library/Developer/CoreSimulator/Devices/AC94F0FF-16F7-46C8-B4BF-218B73C547AC/data/Containers/Data/Application/4F72B076-42AD-44A4-A7CF-57D9F93E895E/tmp/ios_testWIDYdS/ios_test/lib/main.dart': warning: line 16 pos 38: unbalanced '{' opens here
      Widget build(BuildContext context) {
                                         ^
    '/Users/obiwan/Library/Developer/CoreSimulator/Devices/AC94F0FF-16F7-46C8-B4BF-218B73C547AC/data/Containers/Data/Application/4F72B076-42AD-44A4-A7CF-57D9F93E895E/tmp/ios_testWIDYdS/ios_test/lib/main.dart': error: line 33 pos 5: unbalanced ')'
        );
    ```

    在这种情况下，只需更正上述代码的错误，即可以继续使用热重载

	3. CupertinoTabView's builder

    在修改了CupertinoTableView 的builder内容时，Hot reload 不会生效

	4. 枚举类型改变成class 类型

    当一个枚举类型，改成一个类时，Hot reload 不会生效

	5. 字体修改

	6. 泛型修改

    修改前

    ```
    class A<T> {
      T i;
    }
    ```

    修改后

    ```
    class A<T, V> {
      T i;
      V v;
    }
    ```

    

	7. Native code 

    更改原生代码， hot reload 不会生效

	8. statelessWidget 和 statefulWidget 的互改

	9. 静态变量和全局变量的改变



### 参考

[ Flutter Hot Reload doc ](https://flutter.cn/docs/development/tools/hot-reload )

[深入理解flutter的编译原理与优化](https://mp.weixin.qq.com/s?__biz=MzU4MDUxOTI5NA==&mid=2247483816&idx=1&sn=deedc011bc902511d0712de959bb8b06&scene=19#wechat_redirect)

[揭秘Flutter Hot Reload（原理篇)](https://www.yuque.com/xytech/flutter/uhw8vw#tx8udt)

[底层原理 - Flutter Hot Reload 详解](https://xiaozhuanlan.com/topic/0845671932)

[美团-Flutter原理与实践](https://tech.meituan.com/2018/08/09/waimai-flutter-practice.html)

[头条开发攻城狮-深入理解Dart虚拟机启动](http://gityuan.com/2019/06/23/dart-vm/)

[Dart VM 简介](https://annatarhe.github.io/2019/01/31/introduction-to-dart-vm.html)	

[深入浅出RPC原理](https://ketao1989.github.io/2016/12/10/rpc-theory-in-action/)



