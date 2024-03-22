---
layout:     post
title:      "浅谈iOS崩溃解析"
subtitle:   ""
date:       2024-03-22 15:54:00
author:     "Self"
header-style: text
catalog: true
tags:
    - iOS
    - APM
---

[Untitled]:/img/post/20240322/Untitled.png
[Untitled 1]:/img/post/20240322/Untitled 1.png
[Untitled 2]:/img/post/20240322/Untitled 2.png
[Untitled 3]:/img/post/20240322/Untitled 3.png
[Untitled 4]:/img/post/20240322/Untitled 4.png
[Untitled 5]:/img/post/20240322/Untitled 5.png
[Untitled 6]:/img/post/20240322/Untitled 6.png
[Untitled 7]:/img/post/20240322/Untitled 7.png
[Untitled 8]:/img/post/20240322/Untitled 8.png
[Untitled 9]:/img/post/20240322/Untitled 9.png
[Untitled 10]:/img/post/20240322/Untitled 10.png
[Untitled 11]:/img/post/20240322/Untitled 11.png
[Untitled 12]:/img/post/20240322/Untitled 12.png

在开发过程中我们经常会遇到崩溃，而一般来说崩溃会被iOS系统收集，你可以从设置-隐私与安全-分析与改进-分析数据目录下找到历史崩溃信息，一般为.ips格式。但是当我们打开日志文件查看内容时，关键的堆栈信息都是十六进制的。所以最关键的一步就是要解析这个日志，这一过程有一个专业的术语：符号化（symbolicate）

![Untitled]

# 符号化

> Symbolication is the process of resolving backtrace addresses to source code method or function names, known as symbols. Without first symbolicating a crash report it is difficult to determine where the crash occurred. 
>
> 符号化就是解决把栈回溯地址转化为为源码方法名或函数名的过程。

## 符号化级别

符号化的三个级别：未符号化、半符号化和全符号化。

![Untitled 1]

## 符号化文件

### APP的符号文件

我们app的符号文件就藏在我们archive之后的包中，后缀为.dSYM。app的崩溃日志中包含一个uuid，这个和该app对应压缩包中的dsYM的uuid是一一对应的，也就是说我们每次打包app都会产生新的uuid，如果你企图随便使用一个build包的日志和另一个build包的dSYM文件来进行解析是无效的，即使他们都是同一个版本号也不行。

![Untitled 2]

### 系统符号文件

一个崩溃日志中除了包含我们自己的代码行外还包括系统的调用堆栈，我们再使用dSYM时只会解析出属于我们app的代码行，系统的代码行仍然是原始格式。因此，我们还需要解析系统的符号才能获得完整版的崩溃日志。[https://zuikyo.github.io/2016/12/18/iOS Crash日志分析必备：符号化系统库方法/](https://zuikyo.github.io/2016/12/18/iOS%20Crash%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90%E5%BF%85%E5%A4%87%EF%BC%9A%E7%AC%A6%E5%8F%B7%E5%8C%96%E7%B3%BB%E7%BB%9F%E5%BA%93%E6%96%B9%E6%B3%95/)这篇博客总结的很到位。简单说，每个iOS的版本都会对应一个系统符号表，要想做到完整解析所有系统的崩溃日志就要求我们收集尽可能多的系统符号表，通过崩溃日志中的iOS版本使用对应系统版本的系统符号表解析。这是一个长期的、庞大的收集过程。

### 获取系统符号文件的方法

- **从真机上获取**

    大部分系统库符号文件只能从真机上获取，苹果也没有提供下载。

    当你用Xcode第一次连接某台设备进行真机调试时，会看到Xcode显示`Processing symbol files`，这时候就是在拷贝真机上的符号文件到Mac系统的`/Users/xxx/Library/Developer/Xcode/iOS DeviceSupport`目录下。目录下的`10.2(14C82)`这样的文件夹就是对应的符号文件，通常都有1-3GB的大小，很占用空间，动不动就累积成3、40GB。很多讲清理Mac垃圾文件的教程都会说要删除这个目录下的文件，真是坑爹。正确做法是做成压缩包保存到外部硬盘里，需要符号化的时候再重新解压到此目录。

- **寻找苹果官方的下载地址**

    之前watch的调试出现bug时，有人找出过几个watch的符号文件下载地址。见[No symbols for paired Apple Watch](https://forums.developer.apple.com/thread/3380)。

- **从已解密的固件中提取符号文件**

    某些已经被破解的固件可以直接提取系统文件，但是未破解的固件（较新的固件和`arm64`的固件），无法用这种方式获取。

- **下载旧版本Xcode，提取SDK**

    旧版本的Xcode里包含了对应的iPhoneSDK，可以从中获得符号文件。

    路径是`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/`。里面的`System/Library`里就可以看到framework，而且同时包含了`armv7`,`armv7s`,`arm64`3个平台的版本。

    `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/version.plist`可以查看是哪个版本。把`iPhoneOS.sdk`文件夹的名字改成对应的`CFBundleVersion (ProductBuildVersion)`格式，然后在里面加一层`Symbols`子文件夹，把`System`,`Library`,`usr`都放进`Symbols`里，就可以和其他符号文件一样使用了。

    但是当iOS版本只包含了bug修复，而没有改变API，Xcode就不会有附带对应的SDK，还是需要从真机上获取。而且从Xcode7开始，苹果用`tbd`文件代替了真机符号文件，所以这个方法也失效了。

    参考：[Xcode software image for user iOS in order to symbolicate iOS calls](http://stackoverflow.com/questions/14941773/xcode-software-image-for-user-ios-in-order-to-symbolicate-ios-calls), [Missing iOS symbols at “~/Library/Developer/Xcode/iOS DeviceSupport”](http://stackoverflow.com/a/28408052/6380485)。

- **博主的开源项目直接下载**

    [iOS-System-Symbols](https://github.com/Zuikyo/iOS-System-Symbols)

## 如何符号化

### 查看UUID

dSYM和崩溃日志的uuid匹配时成功解析的第一步。

```swift
dwarfdump --uuid dSYM文件路径

崩溃日志的前几行一般都会包含对应的uuid，

{
    "app_name": "XXXXXX",
    "timestamp": "2017-06-03 02:15:01.50 +0800",
    "app_version": "1.2.00",
    "slice_uuid": "ab0ec401-77b7-3ff6-bf99-567fa14a45fc",
    "adam_id": 1128617051,
    "build_version": "1.0.1",
    "share_with_app_devs": false,
    "is_first_party": false,
    "bug_type": "109",
    "os_version": "iPhone OS 10.3.2 (14F89)",
    "incident_id": "1DE9E3EF-D2DF-4A54-992A-1912BF8D285A",
    "name": "XXXXXX"
}
Incident Identifier: 1DE9E3EF-D2DF-4A54-992A-1912BF8D285A
CrashReporter Key:   5abb1af3a567bd61ac640b7daf57c472c59d9547
Hardware Model:      xxx
Version:             1.0.1 (1.2.00)
Code Type:           ARM-64 (Native)
Role:                Foreground
Parent Process:      launchd [1]
```

### symbolicatecrash解析

symbolicatecrash 是苹果官方提供的符号化工具，一般藏在Xcode的路径里面。

```swift
find /Applications/Xcode.app -name symbolicatecrash -type f

通常symbolicatecrash的路径为/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash
```

将崩溃日志，dSYM以及symbolicatecrash复制出来放到同一个文件夹，然后cd到当前文件夹
，运行如下命令解析

```swift
export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer

./symbolicatecrash temp.crash testxcConfig.app.dSYM > result.log

```

到这里可以将我们的app的崩溃行解析出来，之后结合系统崩溃符号表再次执行命令可以将系统堆栈也解析出来。

### **atos解析**

atos是OS X系统的地址符号化工具，它接收可执行文件路径、可执行文件加载地址和符号地址，需要特别注意，地址值需要转换为十六进制。这玩意是一行一行解析的获取的输出中包括符号、库、文件和行号。系统库和商业平台SDK经常会阉割符号表，导致atos输出信息不完整，拆分时需要兼容。

```swift
atos -o 符号文件地址 -arch 架构 -l 可执行文件加载地址 符号地址1 符号地址2 ...

atos -o LJBaseCrashReporter_Example.app.dSYM/Contents/Resources/DWARF/LJBaseCrashReporter_Example -arch arm64 -l 0x104F18000 0x104FA11A8

输出：

-[LJCrashDebugMachsController tableView:didSelectRowAtIndexPath:] (in LJBaseCrashReporter_Example) (LJCrashDebugMachsController.m:135)
```

## 崩溃解析原理

1. **计算崩溃地址对应符号表中的地址**

    ![Untitled 3]

    其中真正保存保存数据的是 DWARF 文件, DWARF(Debuging With Arbitrary Format)是ELF 和 Mach-O 等文件格式中用来存储和处理调试信息的标准格式。 DWARF 中的数据是高度 压 缩 的 ， 可以通过`dwarfdump`命令提取可读信息，比如提取关键的调试信息.debug_info、.debug_line。

    ![Untitled 4]

    第一列为运行时的堆栈地址，第二列为进程运行时的起始地址(testxcConfig 所有行起始地址都相同)，第三列为运行时的偏移地址。运行时堆栈地址=运行时起始地址+偏移地址，以第 4 行为例，0x1022cd990=0x1022c8000 + 0x5990（22928）。以上地址均为 app 发生崩溃时的运行地址，根据虚拟内存偏移地址不变的原理，只要知道符号表 TEXT 段的起始地址，加上偏移量（0x5990）就能得到崩溃地址对应符号表中的地址， 符号表 TEXT 段的起始地址可通过以下命令获得。那么崩溃地址（`0x1022cd990`）对应符号表中的地址为：`0x100005990 =0x0000000100000000+0x5990`

    ![Untitled 5]

1. **地址重映射**

    获取符号表地址后，在 `debug-info` 章节中查找包含该地址的 `DIE(Debug Information Entry)`单元就能获知该符号地址对应的函数名称(name)、 函数所在的文件路径(decl file)和函数所在行数(decl line)，如下图所示。

    ![Untitled 6]

1. **获取准确行数**

    上述步骤解析出了函数相关信息， 下面进一步获取该地址对应的准确行数， 这需要借助`debug_line`章节， `debug_line` 章节以文件为单位，准确记录了文件中的每一行对应的符号表地址， `0x100005990` 对应 `AppDelegate.m` 的第 `20` 行。

    ![Untitled 7]

# MetricKit

MetricKit 是苹果 iOS13 推出的框架，他会在一天结束后，将过去 24 小时内收集的性能数据归集在一起，并在下一次 App 启动时，通过 delegate 方法回调给我们。一个 MXMetricPayload 对象就是一个周期（24 小时）内收集到的所有性能指标的集合。如果有 24 小时以前未被收集过的数据，也会在这里一并返回给我们。所以 delegate 方法这里给到我们的是一个数组。

```swift
- (void)didReceiveDiagnosticPayloads:(NSArray<MXDiagnosticPayload *> * _Nonnull)payloads API_AVAILABLE(ios(14.0)){
				if (@available(iOS 14.0, *)) {
				for (MXDiagnosticPayload *payload in payloads) {
						NSDictionary *payloadDic = [payload dictionaryRepresentation];
							});
				}
		}
}
```

## 性能指标

```objectivec
@interfaceMXMetricPayload: NSObject< NSSecureCoding>

/// 电池指标

@property( readonly, strong, nullable) MXCellularConditionMetric *cellularConditionMetrics;

@property( readonly, strong, nullable) MXCPUMetric *cpuMetrics;

@property( readonly, strong, nullable) MXDisplayMetric *displayMetrics;

@property( readonly, strong, nullable) MXGPUMetric *gpuMetrics;

@property( readonly, strong, nullable) MXLocationActivityMetric *locationActivityMetrics;

@property( readonly, strong, nullable) MXNetworkTransferMetric *networkTransferMetrics;

/// 性能指标

@property( readonly, strong, nullable) MXAppExitMetric *applicationExitMetrics; // New for iOS 14

@property( readonly, strong, nullable) MXAppRunTimeMetric *applicationTimeMetrics;

@property( readonly, strong, nullable) MXMemoryMetric *memoryMetrics;

/// 用户交互相关的响应性指标

@property( readonly, strong, nullable) MXAppLaunchMetric *applicationLaunchMetrics;

@property( readonly, strong, nullable) MXAnimationMetric *animationMetrics; // New for iOS 14

@property( readonly, strong, nullable) MXAppResponsivenessMetric *applicationResponsivenessMetrics;

/// 磁盘存取指标

@property( readonly, strong, nullable) MXDiskIOMetric *diskIOMetrics;

/// 自定义指标

@property( readonly, strong, nullable) NSArray<MXSignpostMetric *> *signpostMetrics;

@end
```

## 集成MetricKit

添加动态库依赖，注册 MetricKit 监听者。

![Untitled 8]

```swift
import MetricKit

if (@available(iOS 14.0, *)) {
MXMetricManager *manager = [MXMetricManager sharedManager];
if (self && manager && [manager respondsToSelector:@selector(addSubscriber:)]) {
		[manager addSubscriber:self];
	}
}
```

## MetricKit的一些基类

- MXDiagnostic ：所有诊断类集成的基类
- MXDiagnosticPayload ：诊断包，包含一天结束时的所有诊断
- MXCallStackTree ：新数据类，用于封装当前环境的调用栈，并没有经过符号化，旨在用于设备外处理，非 debug 用。

![Untitled 9]

## 使用MetricKit进行崩溃解析

应用程序退出是 MetricKit 在 iOS 14 上新增的一个指标 MXAppExitMetric 。他统计的是每天应用程序在前台、后台运行的时候退出或被杀的原因概述。

当收到回调消息后，需要对关键信息做组装，获取崩溃堆栈和相关关键信息。

```swift
NSArray *callStackRootFrames = [dicFrame  ArrayValueForKey:@"callStackRootFrames"];
if (callStackRootFrames.count <= 0) {
    continue;
}
NSDictionary *dicZero = [callStackRootFrames ObjectAtIndex:0];
int rootIndex = 0;
while (dicZero && dicZero.count > 0) {
  //获取Image 的 UUID
    NSString *binaryUUID = [dicZero   stringValueForKey:@"binaryUUID"];
  //获取Image 的 名称
    NSString *binaryName = [dicZero   stringValueForKey:@"binaryName"];
  //获取Image 的加载地址
    long long baseAdd = [[dicZero NumberValueForKey:@"offsetIntoBinaryTextSegment"] longLongValue];
     //获取崩溃函数的地址
    long long address = [[dicZero  numberValueForKey:@"address"] longLongValue];
  //看上一层调用堆栈的
    NSArray *subFrames = [dicZero  arrayValueForKey:@"subFrames"];
    [strStack appendFormat:@"%d %@ 0x%llx 0x%llx + %lld\n", rootIndex, binaryName, baseAdd, address, address - model.baseAddress];
    rootIndex++;
    if (subFrames && subFrames.count >= 0) {
        dicZero = [subFrames  ObjectAtIndex:0];
    } else {
        dicZero = nil;
    }
}
```

## **Metrickit 收集崩溃的不足**

1. 只支持 iOS14 以后的崩溃日志收集；PS:MetricKit是iOS13开始有的框架，但是崩溃日志的支持是iOS14开始支持的。
2. 崩溃日志没有返回具体的崩溃时间和启动时间，崩溃场景信息除了堆栈外没有其余信息，附加信息较少，需要另外的手段来收集
3. 如果使用了段迁移编译技术，主程序 Mach-O 的加载地址和 uuid MetricKit无法给出正确的值，需要例外处理。可通过 Mach-O文件的LC-MAIN入口来获取主程序main函数的地址，从而算出加载其起始地址。
4. iOS14 的崩溃日志是24小时会回调通知一次，时效性低；iOS15 之后，崩溃日志会在下次启动之后就返回，但经验证，可能有例外情况。

# References

[iOS基础之崩溃日志符号化 - 掘金](https://juejin.cn/post/6844903641036357640)

[iOS 性能优化：使用 MetricKit 2.0 收集数据](https://www.jianshu.com/p/3e4b579f4ebe)

[iOS崩溃解析&原理介绍](https://www.jianshu.com/p/49bc468d2e04)

[海神平台iOS端崩溃日志解析踩坑之旅 - 腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/news/672999)

[](https://segmentfault.com/a/1190000042951892)

[iOS崩溃日志ips文件解析-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1190741)

[iOS Crash分析必备：符号化系统库方法](https://zuikyo.github.io/2016/12/18/iOS%20Crash%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90%E5%BF%85%E5%A4%87%EF%BC%9A%E7%AC%A6%E5%8F%B7%E5%8C%96%E7%B3%BB%E7%BB%9F%E5%BA%93%E6%96%B9%E6%B3%95/)