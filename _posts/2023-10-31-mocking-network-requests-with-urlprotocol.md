---
layout: post
title: 'Mocking Network Requests with URLProtocol'
categories: [iOS dev]
tag: [swift, url protocol, url session, unit test, networking]
---

>
*Notice:* The content presented in this blog post is inspired by and derived from the [Essential Developer](https://www.essentialdeveloper.com/) course. All credit for the foundational concepts and methodologies goes to the creators of this course. This post is an attempt to share and expand upon the knowledge I've gained from it.
>

At the early stages of development, it's common for the backend server to be not ready yet. So, how can we app developers create tests for network requests in such situations? Furthermore, even with a functioning server, unstable network conditions can cause flaky test results and longer test durations, potentially slowing development. How can we tackle these challenges while still ensuring robust testing for our apps?

As to the first question, the answer is "YES". Even without a complete server, you can still craft tests to assess various scenarios using networking frameworks like URLSession, Alamofire, and others. According to Apple's testing guidelines presented in [WWDC](https://developer.apple.com/videos/play/wwdc2018/417), our test suite should resemble a pyramid. At its base are numerous unit tests, followed by a smaller set of medium-sized integration tests, and capped off with a select few end-to-end tests.
 
>
These(Unit tests) are characterized by being simple to read, producing clear failure messages when we detect a problem, and by running very quickly, often in the order of hundreds or thousands of tests per minute.
>

In this post, I'll delve into writing unit tests for requests without actually initiating network tasks. I will probably write another post about end-to-end tests in the future, for now, let's focus solely on unit testing.

Below is a typical API protocol that outlines a typealias result and a load function.

```swift
public protocol APIClient {
    typealias Result = Swift.Result<(Data, HTTPURLResponse), Error>
    func load(request: URLRequest, completion: @escaping (Result) -> Void)
}
```

We've created a class called `URLSessionAPIClient` that conforms to the `APIClient` protocol.

```swift
public class URLSessionAPIClient: APIClient {
    private let session: URLSession
    
    public init(session: URLSession = .shared) {
        self.session = session
    }
    
    private struct UnexpectedValuesRepresentation: Error {}
    
    public func load(request: URLRequest, completion: @escaping (APIClient.Result) -> Void) {
        session.dataTask(with: request) { data, response, error in
            completion(Result {
                if let error = error {
                    throw error
                } else if let data = data, let response = response as? HTTPURLResponse {
                    return (data, response)
                } else {
                    throw UnexpectedValuesRepresentation()
                }
            })
        }.resume()
    }
}
```

This class holds a `URLSession` property, which it utilizes for making requests. You have the option to inject a `URLSession` instance or to use the default `shared` one. Next, we'll establish a test file named `URLSessionAPIClientTests`. For creating instances in our tests, we'll use the previously mentioned [factory method](https://pinyuanchen.github.io/posts/xctestcase-sut/).

```swift
final class URLSessionAPIClientTests: XCTestCase {
    // MARK: - Helpers
    private func makeSUT() -> APIClient {
        let sut = URLSessionAPIClient()
        return sut
    }
}
```

Time to write some tests. 📝

We aim to ensure that the behaviors of `URLSessionAPIClient` match our expectations. Initially, we'll check if it can accurately initiate a GET request.

```swift
func test_getFromURL_performsGETRequestWithURL() { }    
```

Here comes a problem: how can we initiate a request using a mock URL? 

The solution lies in intercepting the request and return a response aligning with our specifications. `URLProtocol` is the handy tool that fulfills what we need. Despite its name, `URLProtocol` isn't a protocol but a class. When a URL session request is initiated, the URL loading system automatically creates a `URLProtocol` instance behind the scene to manage the process. Our objective is to simulate `URLProtocol` and intercept the request by implementing the requisite methods.

When subclassing `URLProtocol`, there are four essential methods to implement.

>
class func canInit(with request: URLRequest)
class func canonicalRequest(for request: URLRequest) -> URLRequest
func startLoading()
func stopLoading()
>

In the `startLoading` section lies the crux of our implementation. Here, we'll ascertain if any stubbed data exists and, if so, return them as responses, essentially intercepting the request. The four methods are implemented as follows:

```swift
private class URLProtocolStub: URLProtocol {
    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }
        
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }

    override func startLoading() {
        // We check our stubbed artifacts here.
    }

    override func stopLoading() {}
}
```

For `canInit`, we straightforwardly return `true`, ensuring the request is valid, and the same logic applies to `canonicalRequest`. As there's no need for any specific action in `stopLoading`, we leave it empty. 
Shift back to the `startLoading` method. We'll require a property to capture and later verify the request. This can be achieved by using a closure property.

```swift
private class URLProtocolStub: URLProtocol {
    private static var requestObserver: ((URLRequest) -> Void)?

    static func observeRequests(observer: @escaping (URLRequest) -> Void) {
        requestObserver = observer
    }
}
```

Given that the URL loading system autonomously creates the `URLProtocol` instance, we are not allowed to control its instantiation. As a workaround, we designate `requestObserver` as a static property. Furthermore, we introduce a static function, `observeRequests`, furnishing an external interface for storing the property. Subsequently, we append the following code to the `startLoading` function:"

```swift
override func startLoading() {
    if let requestObserver = URLProtocolStub.requestObserver {
        client?.urlProtocolDidFinishLoading(self)
        return requestObserver(request)
    }
    client?.urlProtocolDidFinishLoading(self)
}

```

The code checks for the presence of a request observer. If it finds one, it completes the loading and triggers the callback.

Now, returning to the test file. Before diving into testing, there's an essential step to finalize the `URLProtocol`. In `setupWithError`, it's crucial to register our `URLProtocolStub` to ensure its functionality.

```swift
override func setUpWithError() throws {
    try super.setUpWithError()
    URLProtocol.registerClass(URLProtocolStub.self)
}
```

Additionally, in `tearDownWithError`, it's necessary to unregister it and also release the observer closure.

```swift
override func tearDownWithError() throws {
    try super.tearDownWithError()
    URLProtocol.unregisterClass(URLProtocolStub.self)
    requestObserver = nil
}
```

For clarity, we encapsulate these steps into two static functions within `URLProtocolStub`, signifying their specific roles.

```swift
private class URLProtocolStub: URLProtocol {
    ...
    static func startInterceptingRequests() {
        URLProtocol.registerClass(URLProtocolStub.self)
    }
        
    static func stopInterceptingRequests() {
        URLProtocol.unregisterClass(URLProtocolStub.self)
        requestObserver = nil
        stub = nil
    }
    ...
}
```
In the test file, the adjustments are made as follows:

```swift
override func setUpWithError() throws {
    try super.setUpWithError()
    URLProtocolStub.startInterceptingRequests()
}

override func tearDownWithError() throws {
    try super.tearDownWithError()
    URLProtocolStub.stopInterceptingRequests()
}
```

Now let's move back to the `test_getFromURL_performsGETRequestWithURL`. The testing sequence follows these steps:

(1) Configure the request details (GET request).

(2) Establish an expectation instance to anticipate asynchronous execution.

(3) Use `URLProtocolStub.observeRequests` to inject a closure that retrieves a URLRequest. Within this closure, validate the URL and httpMethod, then fulfill the expectation to conclude the test.

(4) Initiate the request.

(5) Await its completion.

```swift
func test_getFromURL_performsGETRequestWithURL() {
    let url = URL(string: "http://any-url.com")!
    var request = URLRequest(url: url)
    request.httpMethod = "GET"
    let exp = expectation(description: "Wait for request")

    URLProtocolStub.observeRequests { request in
        XCTAssertEqual(request.url, url)
        XCTAssertEqual(request.httpMethod, "GET")
        exp.fulfill()
    }
   
    makeSUT().load(request: request) { _ in }
        
    wait(for: [exp], timeout: 1.0)
}
```

Upon running the test, it should pass successfully, demonstrating the efficient workings of our `URLProtocolStub`. 

However, we aim to test additional details, like data, error, and URL responses. How can we accomplish this?

Taking cues from our request callback setup, we'll introduce more properties and methods in `URLProtocolStub` for data storage. We'll integrate a private `Stub` structure that houses the relevant information and a static function serving as an interface for data storage.

```swift
private class URLProtocolStub: URLProtocol {
    private static var stub: Stub?
    private static var requestObserver: ((URLRequest) -> Void)?

    private struct Stub {
        let data: Data?
        let response: URLResponse?
        let error: Error?
    }

    ...
        
    static func stub(data: Data?, response: URLResponse?, error: Error?) {
        stub = Stub(data: data, response: response, error: error)
    }
}
```

Subsequent to this, we'll enhance the `startLoading` function, enabling it to iterate through and verify this stored information.

```swift
override func startLoading() {
            
    if let requestObserver = URLProtocolStub.requestObserver {
        client?.urlProtocolDidFinishLoading(self)
        return requestObserver(request)
    }
            
    if let data = URLProtocolStub.stub?.data {
        client?.urlProtocol(self, didLoad: data)
    }
            
    if let response = URLProtocolStub.stub?.response {
        client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
    }
            
    if let error = URLProtocolStub.stub?.error {
        client?.urlProtocol(self, didFailWithError: error)
    }
            
    client?.urlProtocolDidFinishLoading(self)
}

```
Within the stub, we'll traverse each property, triggering delegate methods corresponding to each scenario.

And remember to clear the stub property inside `stopInterceptingRequests`.

```swift
static func stopInterceptingRequests() {
    URLProtocol.unregisterClass(URLProtocolStub.self)
    requestObserver = nil
    stub = nil
}
```

Returning to the test file, our next task is to script a secondary test, ensuring it flags an error for a failed request.

```swift
func test_getFromURL_failsOnRequestError() {
            
    let requestError = NSError(domain: "any error", code: 1)
        
    URLProtocolStub.stub(data: nil, response: nil, error: requestError)
    let sut = makeSUT()
    let exp = expectation(description: "Wait for completion")

    sut.load(request: anyGetRequest()) { result in
        switch result {
            case .failure(let receivedError):
                XCTAssertNotNil(receivedError)
            default:
                XCTFail("Expected failure, got \(result) instead")
        }
        exp.fulfill()
    }
            
    wait(for: [exp], timeout: 1.0)
}

// MARK: - Helpers
...
private func anyGetRequest() -> URLRequest {
    return .init(url: .init(string: "http://any-url.com")!)
}
```
We can certainly streamline the aforementioned process for better reusability. We'll move the logic into a `resultFor` function. Depending on the result type, we'll devise two supplementary functions specifically for success and failure scenarios.

```swift
private func resultFor(data: Data?, response: URLResponse?, error: Error?) -> APIClient.Result {
    URLProtocolStub.stub(data: data, response: response, error: error)
    let sut = makeSUT()
    let exp = expectation(description: "Wait for completion")
        
    var receivedResult: APIClient.Result!
    sut.load(request: anyGetRequest()) { result in
        receivedResult = result
        exp.fulfill()
    }
        
    wait(for: [exp], timeout: 1.0)
    return receivedResult
}

// For success cases
private func resultValuesFor(data: Data?, response: URLResponse?, error: Error?) -> (data: Data, response: HTTPURLResponse)? {
    let result = resultFor(data: data, response: response, error: error)
        
    switch result {
        case let .success((data, response)):
            return (data, response)
        default:
            XCTFail("Expected success, got \(result) instead")
            return nil
    }
}

// For failure cases   
private func resultErrorFor(data: Data?, response: URLResponse?, error: Error?) -> Error? {
        
let result = resultFor(data: data, response: response, error: error)
        
switch result {
    case let .failure(error):
        return error
    default:
        XCTFail("Expected failure, got \(result) instead")
        return nil
    }
}
```
Consequently, our initial test will be transformed as outlined:

```swift
func test_getFromURL_failsOnRequestError() {
        
    let requestError = NSError(domain: "any error", code: 1)
    let receivedError = resultErrorFor(data: nil, response: nil, error: requestError)
        
    XCTAssertNotNil(receivedError)
}
```
This refactoring empowers us to test a diverse range of response scenarios. We can exhaustively test various combinations of data, response, and error, making our tests more robust. For a comprehensive list of test cases and the complete `URLProtocolStub` code, feel free to check out my GitHub [gist](https://gist.github.com/PinYuanChen/d4bb4c70b4973f3eeea8abb103356260).
