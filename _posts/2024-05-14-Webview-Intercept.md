---
layout:     post
title:      "WKWebView请求拦截"
subtitle:   ""
date:       2024-05-14 16:28:00
author:     "Self"
header-style: text
catalog: true
tags:
    - iOS
    - Web
---

# loadFileURL

- 客户端在特定时机下载 html 等静态资源及题目相关的动态资源到本地。
- 客户端完成资源下载后，通过 jsbridge（ydk）将资源远程 url 和本地文件路径的映射关系传给前端，前端再进行路径替换或是文件获取等操作。

```swift
/*! @abstract Navigates to the requested file URL on the filesystem.
 @param URL The file URL to which to navigate.
 @param readAccessURL The URL to allow read access to.
 @discussion If readAccessURL references a single file, only that file may be loaded by WebKit.
 If readAccessURL references a directory, files inside that file may be loaded by WebKit.
 @result A new navigation for the given file URL.
 */
- (nullable WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL API_AVAILABLE(macos(10.11), ios(9.0));
```

这种方式的不足之处：

1. 对于每个需要改造的前端项目，资源获取都需要改写一遍，方案复用性较差。
2. 由于 file 协议引起的问题，如：
    - 带不上 cookie ，登录态丢失问题（目前是通过 ydk 走 native 的 ajax 来解决）
    - 跨域，因为 file 域问题导致远程请求都变成跨域请求，导致大部分接口请求无法响应
    - 由于 iOS 对 js 读取本地文件的限制，某些情况下可能需要通过一些苹果不建议的方式允许 webview 访问本地文件，存在一定安全风险。

```objectivec
WKWebViewConfiguration * config = [[WKWebViewConfiguration alloc] init];
WKPreferences *preferences = [[WKPreferences alloc] init];
[preferences setValue:@(true) forKey:@"allowFileAccessFromFileURLs"];
config.preferences = preferences;
[config setValue:@(true) forKey:@"allowUniversalAccessFromFileURLs"];
```

# LocalServer

通过建立本地server来模拟请求http(s)以解决跨域的问题，iOS可以通过 CocoaHttpServer （可以很好的支持https）｜ GCDWebServer（支持http，oc）｜ Telegraph （支持http(s),swift）三方库来开启本地服务。这种方案也会带来许多额外的问题，如性能消耗、电量消耗、资源访问权限安全等。

```objectivec
- (void)startLocalServer {
    if ([self isM3U8]) {
        [self stopServer];
        self.localServer = [[GCDWebDAVServer alloc] initWithUploadDirectory:[NSString stringWithFormat:@"%@/", self.localPath]];
        if ([self.localServer startWithPort:9999 bonjourName:nil]) {
            self.localPath = [NSString stringWithFormat:@"http://127.0.0.1:9999/%@", self.recordName.length > 0 ? self.recordName : [self.localPath lastPathComponent]];
        } else {
            [self showHint:@"播放失败~"];
        }
    }
}
使用GCDWebDAVServer模拟播放本地m3u8视频
```

# **NSURLProtocol**

可以拦截所有基于 URL Loading System 的网络请求并进行修改，可以让开发者可以在不修改应用内原始请求代码的情况下，去改变 URL 加载的全部细节。

存在的问题：

1. NSURLProtocol 作用范围是全局，一般我们只需要拦截自己的业务页面，但注册 NSURLProtocol 后，可能会导致应用内其他三方页面或 native 的请求被拦截
2. 存在 Post 请求丢失 Body 的问题
3. 私有 API 有审核被拒的风险

# **WKURLSchemeHandler**

支持iOS 11+

hook webview的handlesURLScheme:方法，当scheme是http或https时，返回NO来进行拦截。

```objectivec
@implementation WKWebView (MyURLSchemeHandler)
 
+ (void)load {
    static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            Method originalMethod = class_getClassMethod(self, @selector(handlesURLScheme:));
            Method swizzledMethod = class_getClassMethod(self, @selector(sp_handlesURLScheme:));
            method_exchangeImplementations(originalMethod, swizzledMethod);
        });
}
 
+ (BOOL)my_handlesURLScheme:(NSString *)urlScheme {
    if ([urlScheme isEqualToString:@"https"]) {
        return NO;
    } else {
        return [self my_handlesURLScheme:urlScheme];
    }
}
 
@end
```

自定义 MyURLSchemeHandler实现WKURLSchemeHandler协议，并在WKWebViewConfiguration进行注册

```objectivec
WKWebViewConfiguration * config = [[WKWebViewConfiguration alloc] init];
[config setURLSchemeHandler:[MyURLSchemeHandler new] forURLScheme:@"https"];
```

拦截时实现2个关键方法：

```objectivec
/*! @abstract Notifies your app to start loading the data for a particular resource 
 represented by the URL scheme handler task.
 @param webView The web view invoking the method.
 @param urlSchemeTask The task that your app should start loading data for.
 */
- (void)webView:(WKWebView *)webView startURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask;

/*! @abstract Notifies your app to stop handling a URL scheme handler task.
 @param webView The web view invoking the method.
 @param urlSchemeTask The task that your app should stop handling.
 @discussion After your app is told to stop loading data for a URL scheme handler task
 it must not perform any callbacks for that task.
 An exception will be thrown if any callbacks are made on the URL scheme handler task
 after your app has been told to stop loading for it.
 */
- (void)webView:(WKWebView *)webView stopURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask;
```

定义SchemeService来处理具体的请求，在startURLSchemeTask: 中进行分发和处理。

```objectivec
- (void)webView:(WKWebView *)webView startURLSchemeTask:(id<WKURLSchemeTask>)urlSchemeTask {
    [self processVersion13];
    if (!urlSchemeTask) {
        return;
    }
    MyWebSchemeService *service = [[MyWebSchemeService alloc] initWithProduct:self.product];
    service.urlSchemeTask = urlSchemeTask;
    service.delegate = self.delegate;
    NSString *taskDescription = urlSchemeTask.description;
    if (!taskDescription) {
        return;
    }
    [self.tasks setValue:service forKey:taskDescription];
    @weakify(self);
    [service startLoading:^{
        @strongify(self);
        [self.tasks removeObjectForKey:taskDescription];
    }];
}
```

构造请求

```objectivec
// 把下载好的本地资源返回
    NSURLRequest *fileUrlRequest = [[NSURLRequest alloc] initWithURL:[NSURL fileURLWithPath:fileURL] cachePolicy:NSURLRequestReloadIgnoringLocalCacheData timeoutInterval:0];
    @weakify(self);
    
    NSURLSessionDataTask *dataTask = [[NSURLSession sharedSession] dataTaskWithRequest:fileUrlRequest completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        __block NSURLResponse *resp = response;
        dispatch_async(dispatch_get_main_queue(), ^{
            @strongify(self);
            if (!self.urlSchemeTask) {
                return;
            }
            if (error) {
                [self.urlSchemeTask didFailWithError:error];
                if ([self.delegate respondsToSelector:@selector(loadFileFailed:locaPath:)]) {
                    [self.delegate loadFileFailed:requestUrl locaPath:fileURL];
                }
            } else {
                NSDictionary *headerFields = [self getResponseHeader:response.MIMEType
                                                              length:[NSString stringWithFormat:@"%ld", data.length]
                                                            filePath:fileURL];
                resp = [[NSHTTPURLResponse alloc] initWithURL:self.urlSchemeTask.request.URL statusCode:200 HTTPVersion:@"HTTP/1.1" headerFields:headerFields];
                // 将数据回传给webView
                [self.urlSchemeTask didReceiveResponse:resp];
                [self.urlSchemeTask didReceiveData:data];
                [self.urlSchemeTask didFinish];
                
                if ([self.delegate respondsToSelector:@selector(loadFileSuccess:locaPath:)]) {
                    [self.delegate loadFileSuccess:requestUrl locaPath:fileURL];
                }
                !completeBlock ?: completeBlock();
            }
        });
    }];
    [dataTask resume];
```

执行原始的请求

```objectivec
- (void)loadOriginComplete:(void (^)(void))completeBlock {
    if (!self.urlSchemeTask) {
        return;
    }
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
    @weakify(self);
    NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:self.urlSchemeTask.request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        @strongify(self);
        dispatch_async(dispatch_get_main_queue(), ^{
            if (!self.urlSchemeTask) {
                [session finishTasksAndInvalidate];
                return;
            }
            @try {
                [self.urlSchemeTask didReceiveResponse:response];
                [self.urlSchemeTask didReceiveData:data];
                [self.urlSchemeTask didFinish];
            } @catch (NSException *exception) {
                [LCTLog err:exception];
            } @finally {
                !completeBlock ?: completeBlock();
                [session finishTasksAndInvalidate];
            }
        });
    }];
    [dataTask resume];
}
```

使用 WKURLSchemeTask 实例进行数据回调时，如果此时实例已经被释放就会发生crash。释放的操作是在 WebKit 内核进行，外部无法控制其生命周期。WKURLSchemeHandler 会在提供的协议方法 stopURLSchemeTask 中通知我们 task 不可用，在这个方法里进行对应处理，不再对 task 进行后续操作。

```objectivec
- (void)webView:(WKWebView *)webView stopURLSchemeTask:(id<WKURLSchemeTask>)urlSchemeTask {
    // task 此时已经被结束，不能在进行其他操作
    NSString *taskDescription = urlSchemeTask.description;
    if (!taskDescription) {
        return;
    }
    MyWebSchemeService *service = self.tasks[taskDescription];
    [service stopLoading];
    [self.tasks removeObjectForKey:taskDescription];
}
```

iOS 13.x post请求crash

```objectivec
// https://stackoverflow.com/questions/60198551/ios-wkwebview-wkurlschemehandler-crash-on-posting-body-exc-bad-access/72933152#72933152
- (void)processVersion13 {
    // 获取系统版本 ios13才特殊处理防止post崩溃
    NSString *versionNum = [[UIDevice currentDevice] systemVersion];
    if ([versionNum containsString:@"13."]) {
        SEL selector = sel_registerName("_setLoadResourcesSerially:");
        id webViewClass = NSClassFromString(@"WebView");
        if ([webViewClass respondsToSelector:selector]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
            [webViewClass performSelector:selector withObject:@NO];
#pragma clang diagnostic pop
        }
    }
}
```

处理重定向

```objectivec
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURLRequest * _Nullable))completionHandler {
    // 调用 _didPerformRedirection:newRequest: 执行重定向
    NSArray *privateSelStrArr = @[@"st:", @"que", @"ewRe", @"n:n", @"ctio", @"dire", @"ormRe", @"_didPerf"];
    NSString *selName = [[[privateSelStrArr reverseObjectEnumerator] allObjects] componentsJoinedByString:@""];
    SEL sel = NSSelectorFromString(selName);
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    [self.urlSchemeTask performSelector:sel withObject:response withObject:request];
#pragma clang diagnostic pop
    completionHandler(request);
}

```

# Reference

[「拒绝踩坑」唯一一种拦截 WKWebView 资源请求的方式](https://juejin.cn/post/7219490209716158523)

[性能提升30%以上-JDHybrid h5加载优化实践](https://zhuanlan.zhihu.com/p/600035338)

[WKWebView实现请求拦截，http/https/file等](https://juejin.cn/post/6997294531498475556)

[iOS秒开H5实战总结](https://ramboqiu.github.io/posts/iOS秒开H5实战总结/)