title: 如何在 WKWebView 中实现 Safari 原生的阅读模式
date: 2016-10-21 14:28:16
categories: iOS菜鸟心得


tags: [WKWebView, WebKit, Safari 阅读模式]
---

在浏览器中，阅读模式通常是一个很有用的功能，有人说这是读小说神器，有些人则认为阅读模式可以改善新闻网站的阅读体验，而有些广告商则对此抗议，认为阅读模式损害了广告主利益。当然，阅读模式对于普通用户来说是一种很方便实用的功能，这个无可厚非。

在手机上，阅读模式有两种实现方式，一种是和 Safari 的实现类似的，利用 js 去解析网页数据分析出文本，基本上手机浏览器的实现都和 Safari 类似，另外一种则是抓取网页对应的 RSS 源，解析 RSS 源中的数据格式，取出需要的文本来显示。接下来我要展开讲的就是和 Safari 类似的阅读模式的实现，且是基于 WKWebView 来实现的。

详细的项目代码在这： [PFWebViewController](https://github.com/PerfectFreeze/PFWebViewController)

### 了解 Safari 阅读模式的实现

关于实现，你可以先参考下面这两篇博客：

- [iOS Safari阅读模式分析过程](http://blog.csdn.net/horkychen/article/details/50959785) 
- [iOS Safari阅读模式研究](http://blog.csdn.net/horkychen/article/details/50959771)

> 一定要写这篇博客的原因就是，我能查到与此相关的中文博客有且仅有这两篇，我接下来讲的会更通俗易懂一些，毕竟最后模仿系统实现出来了。上面两篇的作者最后的结论和代码并不能正常运行，但是提供的信息和资源多多少少给了我很多参考。
>
> 其中，第二篇中提到的 JavaScriptCore，由于 WKWebView [不再支持](http://stackoverflow.com/questions/25792131/how-to-get-jscontext-from-wkwebview)，我们会用到 WebKit 支持的发送消息的传递信息模式。核心要用到的 JavaScript 文件在这两篇博客中也有给出。

首先，我们需要用两个 WKWebView 来实现整个流程，一个是主要的浏览窗口 ( 暂且命名为 ) **MainWebView**，另一个则是用于阅读模式页面显示的 **ReaderWebView** 。参与整个阅读模式渲染流程的文件大致有这些：

- index.html
- safari-reader.js # 这个 js 负责的是内容的格式化和渲染，挂载在 **ReaderWebView** 上
- safari-reader-check.js # 这个 js 负责的是内容的判断和提取，挂载在 **MainWebView**  上，（在最新的 Safari 中我一直没有截到这个脚本，从最后实现的效果来看，苹果可能将这部分放进原生实现了，所以这里我们就只能使用这个比较早期版本的 js 实现的内容判断，会出现一些网页最新版 Safari 会出现阅读模式的按钮而你使用它却判断无法进入阅读模式）
- ReaderContext.h # 这就是原生实现的 C++ 代码头文件，应该是作为上面两个 js 之间的原生 Bridge，用指针插入的方法用 C++ 向 js 中插入可以直接调用的原生类和方法

Safari 的基本流程就是，当 **MainWebView** 上的网页的 HTML 加载完成后，执行 safari-reader-check (当然后来可能执行原生代码去判断了，或者使用了更新的 js 文件) 中的 FinderJS 的函数来判断当前网页是否存在合适的 Node 来作为 Article，判断的规则大致就是根据 H 标签的个数，文本的长度之类的参数计算出权重，最后获得一个权重最高的 Node 作为 Article 对象。如果存在这个对象，则会告知浏览器显示阅读模式的按钮，反之浏览器则不会显示这个按钮。

当用户点击了这个按钮之后，Article 对象会被提取出来，通过 ReaderContext 传递给 safari-reader.js 处理，然后**加载到 ReaderWebView 里面**。在 **ReaderWebView** 中渲染的 HTML 是一个模板文件，阅读模式中用到的所有 CSS 样式都会在这个 HTML 里面预先定义好，我们可以通过 Safari 的控制台取到这个 HTML 文件：

![](https://ooo.0o0.ooo/2016/10/13/57ff3ba696edb.png)

在 macOS 的 Safari 中就可以取到，Safari 中在任意有阅读模式选项的页面下点击这个按钮，然后进入页面的检查模式，你会看到 HTML 代码如下：

![](https://ooo.0o0.ooo/2016/10/13/57ff3c766c7a9.png)

当 body 加载的时候，它会调用 `ReaderJS.loaded()` 方法，这个方法负责处理 ReaderContext 传递过来的 Article 对象并将生成的 `<div class="page">` 这个标签添加到 id 为 article 的标签下面。 用到的两个 js 文件都是匿名脚本，其中主要的 safari_reader.js 脚本可以通过点击空白区域添加断点来截获的（因为他实现了一个监听鼠标点击的事件）。

### 我的实现

首先，初始化两个 webView，并在 **MainWebView** 上加载原网页，记得：

```
 _webView.navigationDelegate = self;
 _readerWebView.navigationDelegate = self;
```

初始化的时候，我们需要为两个 **MainWebView** 挂载 safari-reader-check.js 的脚本，需要为 **ReaderWebView**  同时挂载 safari-reader-check.js 和 safari-reader.js 这两个脚本，初始化的时候赋给两个 webview 的 `WKWebViewConfiguration` 按照如下来定义：

```
- (WKWebViewConfiguration *)configuration {
    // Load reader mode js script
    NSBundle *bundle = [NSBundle bundleForClass:[self class]];
    NSURL *url = [bundle URLForResource:@"PFWebViewController" withExtension:@"bundle"];
    NSBundle *scriptBundle = [NSBundle bundleWithURL:url];
    
    NSString *readerScriptFilePath = [scriptBundle pathForResource:@"safari-reader" ofType:@"js"];
    NSString *readerCheckScriptFilePath = [scriptBundle pathForResource:@"safari-reader-check" ofType:@"js"];
    
    NSString *indexPageFilePath = [scriptBundle pathForResource:@"index" ofType:@"html"];
    
    // Load HTML for reader mode
    readerHTMLString = [[NSString alloc] initWithContentsOfFile:indexPageFilePath encoding:NSUTF8StringEncoding error:nil];
    
    NSString *script = [[NSString alloc] initWithContentsOfFile:readerScriptFilePath encoding:NSUTF8StringEncoding error:nil];
    WKUserScript *userScript = [[WKUserScript alloc] initWithSource:script injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:NO];
    
    NSString *check_script = [[NSString alloc] initWithContentsOfFile:readerCheckScriptFilePath encoding:NSUTF8StringEncoding error:nil];
    WKUserScript *check_userScript = [[WKUserScript alloc] initWithSource:check_script injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:NO];
    
    WKUserContentController *userContentController = [[WKUserContentController alloc] init];
    [userContentController addUserScript:userScript];
    [userContentController addUserScript:check_userScript];
    [userContentController addScriptMessageHandler:self name:@"JSController"];
    
    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
    configuration.userContentController = userContentController;

    return configuration;
}
```

这样我们就可以在 `decidePolicyForNavigationResponse` 这个回调中进行阅读模式的判断，这个回调发生的时机是在获得 HTML 响应但尚未根据 HTML 去加载的时候，所以是判断阅读模式的最佳时机：

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler {
    if ([webView isEqual:self.readerWebView]) {
        decisionHandler(WKNavigationResponsePolicyAllow);
        return;
    }
    
    // Set reader mode button status when navigation finished
    [_webView evaluateJavaScript:@"var ReaderArticleFinderJS = new ReaderArticleFinder(document);" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
    }];
    
    [_webView evaluateJavaScript:@"ReaderArticleFinderJS.isReaderModeAvailable();" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
        if ([object integerValue] == 1) {
            self.toolbar.readerModeBtn.enabled = YES;
        } else {
            self.toolbar.readerModeBtn.enabled = NO;
        }
    }];
    
    decisionHandler(WKNavigationResponsePolicyAllow);
}
```

<img src="https://ooo.0o0.ooo/2016/10/21/5809b436031ce.jpeg" width = "40%" />

接下来就是点击阅读模式按钮的响应事件，这里我们可以不用 C++ 搭桥梁的方式传递对象指针，而直接用了一种更 tricky 的办法，将提取出来的 Article 对象以不可见的形式添加到目标的 **ReaderWebView** 中，然后修改获取到的 js 文件，让 safari-reader.js 在渲染完正文内容后将临时的这个不可见的节点删除。

```
[_webView evaluateJavaScript:@"var ReaderArticleFinderJS = new ReaderArticleFinder(document);" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
        }];
[_webView evaluateJavaScript:@"var article = ReaderArticleFinderJS.findArticle();" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
        }];
[_webView evaluateJavaScript:@"article.element.outerHTML" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
	if ([object isKindOfClass:[NSString class]] && isReaderMode) {
		[_webView evaluateJavaScript:@"ReaderArticleFinderJS.articleTitle()" completionHandler:^(id _Nullable object_in, NSError * _Nullable error) {
                    readerArticleTitle = object_in;
                    
                    NSMutableString *mut_str = [readerHTMLString mutableCopy];
                    
                    // Replace page title with article title
                    [mut_str replaceOccurrencesOfString:@"Reader" withString:readerArticleTitle options:NSLiteralSearch range:NSMakeRange(0, 300)];
                    NSRange t = [mut_str rangeOfString:@"<div id=\"article\" role=\"article\">"];
                    NSInteger location = t.location + t.length;
                    
                    NSString *t_object = [NSString stringWithFormat:@"<div style=\"position: absolute; top: -999em\">%@</div>",object];
                    [mut_str insertString:t_object atIndex:location];
                    
                    [_readerWebView loadHTMLString:mut_str baseURL:self.url];
                    _readerWebView.alpha = 0.0f;
                }];
            }
        }];
		[_webView evaluateJavaScript:@"ReaderArticleFinderJS.prepareToTransitionToReader();" completionHandler:^(id _Nullable object, NSError * _Nullable error) {
     	}];
```

> 这里采用的使 div 不可见的方式比较特别，因为 visibilty 或者 display 或者 height 这些参数都会被 js 排除在计算的节点之外，所以用 **top: -999em** 的写法。

**ReaderWebView** 在 load 之后就会调用挂载在上面的 safari-reader.js 中的 loaded 方法：

```javascript
loaded: function() {
    if (!ReaderArticleFinderJS || this._shouldSkipActivationWhenPageLoads())
        return null;
    if (this.loadArticle(), ReaderAppearanceJS.initialize(), ReadingPositionStabilizerJS.initialize(), this._shouldRestoreScrollPositionFromOriginalPageAtActivation) {
        var e = 0;
        if (e > 0)
            document.body.scrollTop = e;
        else {
            var t = document.getElementById("safari-reader-element-marker");
            if (t) {
                var n = parseFloat(t.style.top) / 100,
                i = t.parentElement,
                a = i.getBoundingClientRect();
                document.body.scrollTop = window.scrollY + a.top + a.height * n, i.removeChild(t)
            }
        }
    }
    this._clickingOutsideOfPaperRectangleDismissesReader && (document.documentElement.addEventListener("mousedown", monitorMouseDownForPotentialDeactivation), document.documentElement.addEventListener("click", deactivateIfEventIsOutsideOfPaperContainer));
    var o = function() {
        this.setUserVisibleWidth(this.lastKnownUserVisibleWidth)
    }.bind(this);
    window.addEventListener("resize", o, !1);

    var article_node = document.getElementById("article");
    article_node.firstChild.remove();
    
    var message = { 'code' : 0 };
    window.webkit.messageHandlers.JSController.postMessage(message);
},
```

最后几行先是移除了 article 这个节点之下的第一个子节点，也就是我们添加上去的不可见的临时节点。然后通过 `WKWebView` 新的 js 交互方式发送消息，向原生的 Controller 发送一个加载完成的信号，我们可以在原生的 Controller 里面获取这个信息，然后随即开始阅读模式页面切换的动画：

```
- (void)userContentController:(WKUserContentController *)userContentController
      didReceiveScriptMessage:(WKScriptMessage *)message {}
```

> 除了 loaded 方法，safari-reader.js 还有许多代码需要修改，最终可用的版本请参考项目仓库中的文件。

<img src="https://ooo.0o0.ooo/2016/10/21/5809b43604002.jpeg" width = "40%" />

### 一个多余的问题记录 - 「在微信中打开」按钮点击失效

在 `WKWebView` 中，比如微信网页有一个 「在微信中打开」的按钮会失效。这是因为微信网页的 HTML 写法是直接在 a 标签的 href 里面写上了 `weixin://` 这样开头的链接，对于 `WKWebView` 来说会作为一个普通的 URL 打开，从而无响应，不会有弹框。

**解决办法**就是拦截非 Http:// 和 Https:// 开头的请求，转成应用内跳转：

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    if (![navigationAction.request.URL.absoluteString containsString:@"http://"] && ![navigationAction.request.URL.absoluteString containsString:@"https://"]) {
        
        UIApplication *application = [UIApplication sharedApplication];
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 100000
        if ([application respondsToSelector:@selector(openURL:options:completionHandler:)]) 		{
            [application openURL:navigationAction.request.URL options:@{} completionHandler:nil];
         } else {
            [application openURL:navigationAction.request.URL];
         }
#else
        [application openURL:navigationAction.request.URL];
#endif
        decisionHandler(WKNavigationActionPolicyCancel);
    } else {
        decisionHandler(WKNavigationActionPolicyAllow);
    }
}
```

### PS.

最后说一句，当然，毕竟这样抓取苹果的脚本来做和系统一样的效果是很 tricky 的办法，但是：

- 审核是能过的，但是不保证能不能一定过

- 有一些网页可能最新的 Safari 支持阅读模式，然而通过上述实现的浏览器却没法支持
- 如果遇到了 2 的问题，你可以选择使用苹果官方封装的更好的 SafariWebViewController，绝对原生的效果，当然就是自定义程度更低了