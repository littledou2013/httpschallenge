# httpschallenge
关于https认证的处理

苹果2017年开始强制客户端使用https，在ios客户端需要哪些变化呢？如果你是使用AFNetworking，则不需要做任何更改，AFNetworking已经帮你做了处理，如果没有用AFNetworking，则需要自己做处理（当然不做处理，对于认证过的https也是能够访问的），下面是AFNetworking 3.2.1版本中对认证的处理代码:

```
- (void)URLSession:(NSURLSession *)session
	didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
	 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
	{
	    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
	    __block NSURLCredential *credential = nil;


	    if (self.sessionDidReceiveAuthenticationChallenge) {
	        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
	    } else {
	        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
	            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
	                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
	                if (credential) {
	                    disposition = NSURLSessionAuthChallengeUseCredential;
	                } else {
	                    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
	                }
	            } else {
	                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
	            }
	        } else {
	            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
	        }
	    }


	    if (completionHandler) {
	        completionHandler(disposition, credential);
	    }
	}

```

```
- (void)URLSession:(NSURLSession *)session
	              task:(NSURLSessionTask *)task
	didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
	 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
	{
	    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
	    __block NSURLCredential *credential = nil;


	    if (self.taskDidReceiveAuthenticationChallenge) {
	        disposition = self.taskDidReceiveAuthenticationChallenge(session, task, challenge, &credential);
	    } else {
	        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
	            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
	                disposition = NSURLSessionAuthChallengeUseCredential;
	                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
	            } else {
	                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
	            }
	        } else {
	            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
	        }
	    }


	    if (completionHandler) {
	        completionHandler(disposition, credential);
	    }
	}


```

#### 关于处理认证的官方文档：

https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge?language=objc

#### 关于手动信任服务器证书的官方文档：
https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge/performing_manual_server_trust_authentication?language=objc


## 在该案例中主要简单演示了最简单的信任服务器证书的代码：

### 信任服务器证书：
当您通过使用安全连接（例如https）时，您将收到NSURLAuthenticationMethodServerTrust的身份验证质询。与服务器要求您的应用程序进行身份验证的其他挑战不同，这是您验证服务器凭据的机会。

## 实现：

验证服务器凭据是一个session-level的认证，首先会调用URLSession:didReceiveChallenge:completionHandler:，如果该方法没有实现，则会调用 URLSession:task:didReceiveChallenge:completionHandler:，如果该方法还是没有实现，系统会使用默认的方法对证书进行认证

## 为什么我们要手动处理服务器证书的认证：
如果服务器的证书是经过官方认证的，则我们不需要处理，系统默认处理将能信任证书，但是如果服务器的证书不是经过官方认证的，系统的默认处理将会不信任证书，我们无法拿到数据，所以需要我们手动去信任这些证书（在信任这些证书之前也可以做相应的判断）
