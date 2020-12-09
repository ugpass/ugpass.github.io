---
title:      "RN的WebView、OC的WKWebView和H5交互" 
date:       2019-09-02 22:00:00
tags:
    - iOS
categories:
    - iOS
---

### OC WKWebView和H5交互

#### H5给OC发消息（H5调用OC的方法）

1、在OC端通过以下方法，注册要被调用的函数名

	 - (void)addScriptMessageHandler:(id<WKScriptMessageHandler>)scriptMessageHandler name:(NSString *)name; 

> Adding a script message handler with name name causes the JavaScript function window.webkit.messageHandlers.name.postMessage(messageBody) to be defined in all frames in all web views that use the user content controller.

2、H5通过```window.webkit.messageHandlers.name.postMessage(messageBody)```
来调用在OC端注册了的函数。

	name即在OC端注册的函数名。

	messageBody可作为参数传递给OC端。

> 当不需要传递参数时，应该```window.webkit.messageHandlers.name.postMessage(null);```,不写null时OC是收不到消息的


3、OC通过以下代理监听H5发过来的消息

```
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
	    NSString *messageName = message.name;
	    id messageBody = message.body;
	    if ([messageName isEqualToString:name]) {
	        //doSomething...
	    }
	}
```

通过判断messageName和自己注册的是否一直，分别进行操作。

messageBody可以是字符串，可以是字典格式。

#### OC给H5发消息（OC调用H5的方法或者回调）

```
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^)(id, NSError *error))completionHandler;

NSString *javaScriptString = [NSString stringWithFormat:@"OCCallJS('%@','%@')",@"v1",@"v2"];
```
	
> 传递参数为字符串时，```OCCallJS('%@','%@')```,```%@```必须被引号引住
> 
> 在H5端，alert收到的参数是不生效的，不知道为什么😂

### react-native WebView和H5交互

H5给RN发消息 

```
window.postMessage("jsCallRN");
```

RN获取H5发送的消息

```
onMessage = (e) => {
	let jsMessage = e.nativeEvent.data;
	if (jsMessage && jsMessage == "jsCallRN") {
		//doSomething...
	}
}

```

---

RN给H5发消息

```
this.webView.postMessage("messageStr");
```

H5获取RN发送的消息

```
window.document.addEventListener('message', function (e) {
    const messageStr = e.data;
    //doSomething
});
```

> 注意
> 调试时，H5页面报错，可能互相交互传递的消息收不到。