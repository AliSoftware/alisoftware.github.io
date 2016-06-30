---
layout: post
title: "Networking Architecture in Swift"
categories: swift architecture
published: false
---

# Network Architecture in Swift

### Network Stack

```swift
protocol NetworkStackProtocol {
  func sendRequest<T>(request: NSURLRequest, parser: (NSData) -> T) -> Observable<T>
}

class AlamofireNetworkStack: NetworkStackProtocol {
  func sendRequest(request: NSURLRequest) -> Observable<NSData> {
    // Some implementation using Alamofire to send the request
  }
}
// and any other implementation, including a StubNetworkStack, is possible
```

### Authentication

```swift
protocol AuthProtocol {
  func enrichRequestWithCredentials(NSMutableURLRequest)
  func credentialsExpired() -> Bool
  func renewToken() -> Observable<Void>
  func responseNeedsReauth(NSURLHTTPResponse) -> Bool
}
class OAuthService: AuthProtocol {
  func enrichRequestWithCredentials(request: NSMutableURLRequest) { /* add "Authaurization: Bearer XXX" header */ }
  func credentialsExpired() -> Bool { /* return token.expirationDate > now */ }
  func renewToken() -> Observable<Void> { /* use refresh token to renew access token */ }
  func responseNeedsReauth(response: NSURLHTTPResponse) -> Bool { return response.statusCode == 401 }
}
class OTPService: AuthProtocol {
  func enrichRequestWithCredentials(request: NSMutableURLRequest) { /* add One-Time Password token header */ }
  func credentialsExpired() -> Bool { return otpLastWindow != otpCurrentTimeWindow }
  func renewToken() -> Observable<Void> { /* recompute next OTP token */ }
  func responseNeedsReauth(response: NSURLHTTPResponse) -> Bool { return response.statusCode == 401 }
}
```

### Parser Layer

```swift
protocol SerializationProtocol {
  func parse(data: NSData) -> Person
  func parse(data: NSData) -> Document
  func parse(data: NSData) -> [Document]
  …
}
class SerializationService: SerializationProtocol {
  // my implementation of each function for parsing each different kind of expected response
}
```

### Api Client

```swift
class WebServiceClient {
  let networkStack: NetworkStackProtocol = NetworkStack() // in practice will be dependency-injected
  let authService: AuthProtocol = OAuthService() // in practice will be dependency-injected
  let parser: SerializationProtocol = SerializationService() // in practice will be dependency-injected
  func fetchDocuments() -> Observable<[Document]> {
    return retryAuth(3) {
      let request = … // build the request (potentially using private helper functions)
      return networkStack.sendRequest(request)
        .map { (data: NSData) -> [Document] in parser.parse(data) }
        .map { (documents: [Document]) in self.storeInDB(documents) }
    }
    .subscribeOn(bkgQueue)
  }

  private retryAuth<T>(retryCount: Int = 1, blockToRetry: () -> Observable<T>) -> Observable<T> {
    blockToRetry().catchError{ (error) in
      if retryCount > 0, case Error.Http(let response) = error where authService.responseNeedsReauth(response) {
        retryAuth(retryCount-1, blockToRetry)
      } else {
        throw error
      }
    }
  }
}
```

### Example of call site

_wherever that is (ViewController, Coordinator, ViewModel, depending from where you prefer to pilot your WebService calls)_

```swift
class SayItsInMyVC: UIViewController {
  let wsClient = WebServiceClient() // typically dependency-injected
  func reloadDocumntsList() {
    wsClient.fetchDocuments()
      .observeOn(DispatchMain.instance)
      .subscribeNext { (docs) in
        self.dataSource = docs
        self.tableView.reloadData)
      }
      .addDisposableTo(rx_disposeBag)
  }
}
```
