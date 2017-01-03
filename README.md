# XloggerSwiftDemo
A swift demo of Wechat Mars/Xlogger
Swift 接入微信开源组件 Mars 的示例代码

To see the guideline about this project, you can click the below link：
[http://www.jianshu.com/p/fe8c5f3f6389](http://www.jianshu.com/p/fe8c5f3f6389)

> 支持开源，但吐槽一句，作为公司级开源项目，文档严重不全，希望微信的开发大大能尽快补上。

# Step 1 下载代码
使用 git  clone 或者直接下载 zip，解压后进入目录 mars-master/mars/libraries，看到有一个 build_apple.py 的文件（官方文档没说清楚在哪里目录下 orz...）
￼![](https://github.com/Jinkeycode/XloggerSwiftDemo/blob/master/README_image/xlogger1.jpg)

# Step 2 编译Mars
在终端进入工程目录输入
```shell
python build_apple.py
```
然后回答一系列问题
第一个问题
> input prefix for save directory. like `trunk`,`br`,`tag`: 
> 输入保存目录的前缀

第二个问题
> Enter menu:
> 1. build mars for iphone.
> 2. build mars for iphone with bitcode.
> 3. build xlog for iphone
> 4. build mars for macosx.
> 5. build all.
> 6. exit.

选择 3 回车，报错：
> xcodebuild: error: Unknown build action 'Center/marsmaster/mars/libraries/../marslogiphone.xcodeproj'.
!!!!clean iphoneos10.0 failed!!!

看看控制台打印的记录发现路径和我目录的路径不一致：
> Download\ Center/mars-master/mars/mars-log-iphone.xcodeproj

对比之后发现一个大坑：**build_apple.py 的路径不能有空格！！！**

编译成功之后生成一个以你自定义前缀的目录，里面就有 framwork：
￼
从 mars-log-iphone.xcodeproj 的 iOS deployment target 来看，最低支持 iOS 7.

# Step 3 引入项目
将 mars.framework 拖入 Linked Frameworks and Libraries 并且加入其他四个系统库，弄好之后如下：
￼

# Step 4 引入辅助库
将编译得到的 `log_crypt.cc`（log_crypt.cc.rewriteme 直接重命名去掉 .rewriteme）、`log_crypt.h` 拖入 Xcode 左侧目录结构，弹出的对话框勾选 

￼

`mars-master/samples/iOS/iOSDemo/Component` 目录下的 `LogHelper.h`、`LogHelper.mm`、`LogUtil.h`、`LogUtil.m` 拖入 Xcode 左侧目录结构，弹出的对话框勾选 `“Copy items if needed”`
为了整洁，对几个文件进行了分组
￼

最终的文件目录和工程目录如下：(忽略 Appender2SwiftBridge和）， 下文会说到)
￼

# Step 5 桥接 Objective-C 和 C++ 代码
新建两个文件（不想写的可以直接下载 Github 下的示例代码拖入工程）
appender-swift-bridge.h
```objective-c
//  Created by Jinkey on 2017/1/2.
//  Copyright © 2017年 Jinkey. All rights reserved.
//  appender-swift-bridge.h

#include <stdio.h>
#import <Foundation/Foundation.h>
#import "LogUtil.h"

typedef NS_ENUM(NSUInteger, XloggerType) {
    
    debug,
    info,
    warning,
    error,
    
};

@interface JinkeyMarsBridge: NSObject

- (void)initXlogger: (XloggerType)debugLevel releaseLevel: (XloggerType)releaseLevel path: (NSString*)path;
- (void)deinitXlogger;

- (void)log: (XloggerType) level tag: (const char*)tag content: (NSString*)content;

@end
```
appender-swift-bridge.mm
```objective-c
//  Created by Jinkey on 2017/1/2.
//  Copyright © 2017年 Jinkey. All rights reserved.
//  appender-swift-bridge.mm

#import "appender-swift-bridge.h"
#import <mars/xlog/appender.h>
#import <mars/xlog/xlogger.h>
#import <sys/xattr.h>

@implementation JinkeyMarsBridge

// 封装了初始化 Xlogger 方法
// initialize Xlogger
-(void)initXlogger: (XloggerType)debugLevel releaseLevel: (XloggerType)releaseLevel path: (NSString*)path{
    
    NSString* logPath = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0] stringByAppendingString:path];
    
    // set do not backup for logpath
    const char* attrName = "io.jinkey";
    u_int8_t attrValue = 1;
    setxattr([logPath UTF8String], attrName, &attrValue, sizeof(attrValue), 0, 0);
    
    // init xlog
    #if DEBUG
    switch (debugLevel) {
        case debug:
            xlogger_SetLevel(kLevelDebug);
        case info:
            xlogger_SetLevel(kLevelInfo);
        case warning:
            xlogger_SetLevel(kLevelWarn);
        case error:
            xlogger_SetLevel(kLevelError);
        default:
            break;
    }
    appender_set_console_log(true);
    #else
    switch (releaseLevel) {
        case debug:
            xlogger_SetLevel(kLevelDebug);
        case info:
            xlogger_SetLevel(kLevelInfo);
        case warning:
            xlogger_SetLevel(kLevelWarn);
        case error:
            xlogger_SetLevel(kLevelError);
        default:
            break;
    }
    appender_set_console_log(false);
    #endif
    appender_open(kAppednerAsync, [logPath UTF8String], "Test");
    
}

// 封装了关闭 Xlogger 方法
// deinitialize Xlogger
-(void)deinitXlogger {
    appender_close();
}


// 利用微信提供的 LogUtil.h 封装了打印日志的方法
// print log using LogUtil.h provided by Wechat
-(void) log: (XloggerType) level tag: (const char*)tag content: (NSString*)content{
    
    NSString* levelDescription = @"";
    
    switch (level) {
        case debug:
            LOG_DEBUG(tag, content);
            levelDescription = @"Debug";
            break;
        case info:
            LOG_INFO(tag, content);
            levelDescription = @"Info";
            break;
        case warning:
            LOG_WARNING(tag, content);
            levelDescription = @"Warn";
            break;
        case error:
            LOG_ERROR(tag, content);
            levelDescription = @"Error";
            break;
        default:
            break;
    }
    
    #if DEBUG
    NSLog(@"[%s][%s]%@%@", levelDescription, tag, content, @">>>该行 log 由于目前Xlogger 在控制台输出中文会乱码而使用 NSlog 输出的, 不会记录到 Xlogger 文件中且在 Release 版本中不会输出到控制台");
    #endif
}

@end
```
> Xlogger 目前在 Xcode 的控制台输出中文会乱码，不清楚是 Xcode 还是 Xlogger 的问题，待官方解决吧

# Step 6 桥接 Swift 和 Objective-C
新建文件 <工程名>-Bridging-Header.h，我这里的示例工程名为XloggerSwiftDemo 所以新建文件XloggerSwiftDemo-Bridging-Header.h
写入以下代码
```objective-c
#import "appender-swift-bridge.h"
```
至此，Xlogger 的基本方法已暴露给 Swift 使用了。

# Step 7 初始化和反初始化 Xlogger
打开 AppDelegate.swift
在 didFinishLaunchingWithOptions 方法中加入以下代码初始化
```swift
var jmb = JinkeyMarsBridge()
jmb.initXlogger(.debug, releaseLevel: .info, path: "/jinkeylog")
```
> 其中 initXlogger 的第一个参数是开发环境显示日志的级别，第二个参数是生产环境显示日志的级别，第三个是储存路径。日志的级别我在示例代码中封装了 debug，info，warning，error 四个级别。

在 applicationWillTerminate 方法中加入以下代码反初始化
```swift
JinkeyMarsBridge().deinitXlogger()
```

# Step 8 打印日志
在想要打印日志的地方写入以下代码
```swift
var jmb = JinkeyMarsBridge()
jmb.log(.debug, tag: "JinkeyIO", content: "我的公众号是 jinkey-love")
```
> 这里为了说明方便而在打印日志的地方实例化，生产环境使用建议使用单例模式实例化JinkeyMarsBridge

# Step 9 分析日志
通过以下代码在控制台打印出模拟器中示例程序沙盒所在的目录
```swift
var logPath = NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, true)[0]
print(logPath)
```
通过 MacOS 的 Finder-前往文件夹粘贴该路径打开
￼

