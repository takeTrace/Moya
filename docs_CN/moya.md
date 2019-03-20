# 文档

在您的应用中，Moya从事于高层抽象的工作.
它通过以下管道实现了这一点.

![Pipeline](https://raw.github.com/Moya/Moya/master/web/pipeline.png)

----------------

<p align="center">
    <a href="Targets.md">Targets</a> &bull; <a href="Endpoints.md">Endpoints</a> &bull; <a href="Providers.md">Providers</a> &bull; <a href="Authentication.md">Authentication</a> &bull; <a href="ReactiveSwift.md">ReactiveSwift</a> &bull; <a href="RxSwift.md">RxSwift</a> &bull; <a href="Threading.md">Threading</a> &bull; <a href="Plugins.md">Plugins</a>
</p>

----------------

你不应用直接引用Alamofire. 虽然它是一个很棒的库,但是Moya的观点是你不必处理那些低级的细节。.

(如果你需要使用Alamofire, 你可以传递一个 `SessionManager` 对象实例给到
`MoyaProvider` 构造器.)

如果你想改变Moya的行为，库中可能已经有一种在不修改库的情况下来达到你的目的方法。Moya被设计的超级灵活且满足了每个开发者的需求. 它不是一个实现网络请求的编码性的框架（那是Alamofire的责任），更多的是关于如何考虑网络请求的框架.

记住, 如果在任何时候你有问题, 只要 [open an issue](http://github.com/Moya/Moya/issues/new)
我们会给你一些帮助。

# Targets

Moya的使用始于定义一个target——典型的是定义一个符合`TargetType` 协议的枚举类型。然后,您的APP剩下的只处理那些target。Target是一些你希望在API上采取的动作，比如 "`favoriteTweet(tweetID: String)`"。

这儿有个示例:

```swift
public enum GitHub {
    case zen
    case userProfile(String)
    case userRepositories(String)
    case branches(String, Bool)
}
```

Targets必须遵循 `TargetType`协议。 `TargetType`协议要求一个`baseURL`属性必须在这个枚举中定义，注意它不应该依赖于`self`的值，而应该直接返回单个值（如果您多个base URL，它们独立的分割在枚举和Moya中）。下面开始我们的扩展:

```swift
extension GitHub: TargetType {
    public var baseURL: URL { return URL(string: "https://api.github.com")! }
}
```

这个协议指定了你API端点相对于它base URL的位置（下面有更多的）

```swift
public var path: String {
    switch self {
    case .zen:
        return "/zen"
    case .userProfile(let name):
        return "/users/\(name.urlEscaped)"
    case .userRepositories(let name):
        return "/users/\(name.urlEscaped)/repos"
    case .branches(let repo, _):
        return "/repos/\(repo.urlEscaped)/branches"
    }
}
```

注意我们使用“`_` ”符号，忽略了分支中的第二个关联值。这是因为我们不需要它来定义分支的路径。注意这儿我们使用了String的扩展`urlEscaped`。
这个文档的最后会给出一个实现的示例。

OK, 非常好. 现在我们需要为枚举定义一个`method`, 这儿我们始终使用GET方法,所以这相当的简单:

```swift
public var method: Moya.Method {
    return .get
}
```

非常好. 如果您的一些端点需要POST或者其他的方法，那么您需要使用switch来分别返回合适的值。swith的使用在上面 `path`属性中已经看到过了。

我们的`TargetType`快成形了,但是我们还没有完成。我们需要一个`task`的计算属性。它返回可能带有参数的task类型。

下面是一个示例:

```swift
public var task: Task {
    switch self {
    case .userRepositories:
        return .requestParameters(parameters: ["sort": "pushed"], encoding: URLEncoding.default)
    case .branches(_, let protected):
        return .requestParameters(parameters: ["protected": "\(protected)"], encoding: URLEncoding.default)
    default:
        return .requestPlain
    }
}
```

不像我们先前的`path`属性, 我们不需要关心 `userRepositories` 分支的关联值, 所以我们省略了括号。
让我们来看下 `branches` 分支: 我们使用 `Bool` 类型的关联值(`protected`) 作为请求的参数值，并且把它赋值给了字典中的 `"protected"` 关键字。我们转换了 `Bool` 到 `String`。(Alamofire 没有自动编码`Bool`参数, 所以需要我们自己来完成这个工作).

当我们谈论参数时，这里面隐含了参数需要被如何编码进我们的请求。我们需要通过`.requestParameters`中的`ParameterEncoding`参数来解决这个问题。Moya有 `URLEncoding`, `JSONEncoding`, and `PropertyListEncoding`可以直接使用。您也可以自定义编码，只要遵循`ParameterEncoding`协议即可（比如，`XMLEncoder`）。

 `task` 属性代表你如何发送/接受数据，并且允许你向它添加数据、文件和流到请求体中。这儿有几种`.request` 类型:

- `.requestPlain` 没有任何东西发送
- `.requestData(_:)` 可以发送 `Data` (useful for `Encodable` types in Swift 4)
- `.requestJSONEncodable(_:)`
- `.requestParameters(parameters:encoding:)` 发送指定编码的参数
- `.requestCompositeData(bodyData:urlParameters:)` & `.requestCompositeParameters(bodyParameters:bodyEncoding:urlParameters)` which allow you to combine url encoded parameters with another type (data / parameters)

同时, 有三个上传的类型:

- `.uploadFile(_:)` 从一个URL上传文件,
- `.uploadMultipart(_:)` multipart 上传
- `.uploadCompositeMultipart(_:urlParameters:)` 允许您同时传递 multipart 数据和url参数

还有 两个下载类型:
- `.downloadDestination(_:)` 单纯的文件下载
- `.downloadParameters(parameters:encoding:destination:)` 请求中携带参数的下载。


下面, 注意枚举中的`sampleData`属性。 这是`TargetType`协议的一个必备属性。这个属性值可以用来后续的测试或者为开发者提供离线数据支持。这个属性值依赖于 `self`.

```swift
public var sampleData: Data {
    switch self {
    case .zen:
        return "Half measures are as bad as nothing at all.".data(using: String.Encoding.utf8)!
    case .userProfile(let name):
        return "{\"login\": \"\(name)\", \"id\": 100}".data(using: String.Encoding.utf8)!
    case .userRepositories(let name):
        return "[{\"name\": \"Repo Name\"}]".data(using: String.Encoding.utf8)!
    case .branches:
        return "[{\"name\": \"master\"}]".data(using: String.Encoding.utf8)!
    }
}
```

最后, `headers` 属性存储头部字段，它们将在请求中被发送。

```swift
public var headers: [String: String]? {
    return ["Content-Type": "application/json"]
}
```

在这些配置后, 创建我们的 [Provider](Providers.md) 就像下面这样简单:

```swift
let GitHubProvider = MoyaProvider<GitHub>()
```

URLs的转义
-------------

这个扩展示例，需要您很容易的把常规字符串"like this" 转义成url编码的"like%20this"字符串:

```swift
extension String {
    var urlEscaped: String {
        return addingPercentEncoding(withAllowedCharacters: .urlHostAllowed)!
    }
}
```

# (端点)Endpoints

endpoint是Moya的半个内部数据结构，它最终被用来生成网络请求。 每个endpoint 都存储了下面的数据:

- url.
- HTTP 方法 (`GET`, `POST`, etc).
- HTTP 请求头.
- `Task` 用来区别 `upload`, `download` 和 `request`.
- sample response (为单元测试).

[Providers](Providers.md) 映射 [Targets](Targets.md) 成 Endpoints, 然后映射
Endpoints 到实际的网络请求。

有两种方式与Endpoints交互。

1. 当创建一个provider, 您可以指定一个从`Target` 到 `Endpoint`的映射.
1. 当创建一个provider, 您可以指定一个从`Endpoint` to `URLRequest`的映射.

第一个可能类似如下:

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint in
    let url = URL(target: target).absoluteString
    return Endpoint(url: url, sampleResponseClosure: {.networkResponse(200, target.sampleData)}, method: target.method, task: target.task)
}
```

这实际上也Moya provide的默认实现。如果您需要一些定制或者创建一个在单元测试中返回一个非200HTTP状态的测试provide，这就是您需要自定义的地方。

注意 `URL(target:)` 的初始化, Moya 提供了一个从`TargetType`到`URL`的便利扩展。

第二个使用非常的少见。Moya试图让您不用操心底层细节。但是，如果您需要，它就在那儿。它的使用涉及的更深入些.。

让我们来看一个从Target到EndpointLet的灵活映射的例子。

## 从 Target 到 Endpoint

在这个闭包中，您拥有从`Target` 到 `Endpoint`映射的绝对权利，
您可以改变`task`, `method`, `url`, `headers` 或者 `sampleResponse`。
比如, 我们可能希望将应用程序名称设置到HTTP头字段中，从而用于服务器端分析。

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint in
    let defaultEndpoint = MoyaProvider.defaultEndpointMapping(for: target)
    return defaultEndpoint.adding(newHTTPHeaderFields: ["APP_NAME": "MY_AWESOME_APP"])
}
let provider = MoyaProvider<GitHub>(endpointClosure: endpointClosure)
```

*注意头字段也可以作为[Target](Targets.md)定义的一部分。*

这也就意味着您可以为部分或者所有的endpoint提供附加参数。 比如, 假设 `MyTarget` 除了实际执行身份验证的值之外，其他的所有值都需要有一个身份证令牌，我们可以构造一个类似如下面的
`endpointClosure` 。

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint in
    let defaultEndpoint = MoyaProvider.defaultEndpointMapping(for: target)

    // Sign all non-authenticating requests
    switch target {
    case .authenticate:
        return defaultEndpoint
    default:
        return defaultEndpoint.adding(newHTTPHeaderFields: ["AUTHENTICATION_TOKEN": GlobalAppStorage.authToken])
    }
}
let provider = MoyaProvider<GitHub>(endpointClosure: endpointClosure)
```

太棒了.

请注意，我们可以依赖于Moya的现有行为，而不是替换它。 `adding(newHttpHeaderFields:)` 函数允许您依赖已经存在的Moya代码并添加自定义的值 。

Sample responses 是 `TargetType` 协议的必备部分。然而, 它们仅指定返回的数据。在Target-到-Endpoint的映射闭包中您可以指定更多对单元测试非常有用的细节。


Sample responses 有下面的这些值:

- `.networkError(NSError)` 当网络发送请求失败, 或者未能检索到响应 (比如 ，超时).
- `.networkResponse(Int, Data)` 这个里面 `Int` 是一个状态码， `Data` 是返回的数据.
- `.response(HTTPURLResponse, Data)` 这个里面 `HTTPURLResponse` 是一个 response ， `Data` 是返回的数据. 这个可用来完全的stub一个响应。


## Request 映射

我们先前已经提到过, 这个库的目标不是来提供一个网络访问的代码框架——那是Alamofire的事情。
 Moya 是一种构建网络访问和为定义良好的网络目标提供编译时检查的方式。 您已经看到了如何使用`MoyaProvider`构造器中的`endpointClosure`参数把target映射成endpoint。这个参数让你创建一个  `Endpoint` 实例对象，Moya将会使用它来生成网络API调用。 在某一时刻,
`Endpoint` 必须被转化成 `URLRequest` 从而给到 Alamofire。
这就是 `requestClosure` 参数的作用.

`requestClosure` 是可选的,是最后编辑网络请求的时机 。 它有一个默认值`MoyaProvider.defaultRequestMapping`,
这个值里面仅仅使用了`Endpoint`的 `urlRequest` 属性 .

这个闭包接收一个`Endpoint`实例对象并负责调用把代表Endpoint的request作为参数的`RequestResultClosure`闭包 ( `Result<URLRequest, MoyaError> -> Void`的简写) 。
在这儿，您要做OAuth签名或者别的什么。由于您可以异步调用闭包，您可以使用任何您喜欢的权限认证库，如 ([example](https://github.com/rheinfabrik/Heimdallr.swift))。
//不修改请求，而是简单地将其记录下来。

```swift
let requestClosure = { (endpoint: Endpoint, done: MoyaProvider.RequestResultClosure) in
    do {
        var request = try endpoint.urlRequest()
        // Modify the request however you like.
        done(.success(request))
    } catch {
        done(.failure(MoyaError.underlying(error)))
    }

}
let provider = MoyaProvider<GitHub>(requestClosure: requestClosure)
```

`requestClosure`用来修改`URLRequest`的指定属性或者提供直到创建request才知道的信息（比如，cookie设置）给request是非常有用的。注意上面提到的`endpointClosure` 不是为了这个目的，也不是任何特定请求的应用级映射。

这个闭包参数实际在编辑请求对象时是非常有用的。
`URLRequest` 有很多你可以自定义的属性。比方，你想禁用所有请求的cookie:

```swift
{ (endpoint: Endpoint, done: MoyaProvider.RequestResultClosure) in
    do {
        var request: URLRequest = try endpoint.urlRequest()
        request.httpShouldHandleCookies = false
        done(.success(request))
    } catch {
        done(.failure(MoyaError.underlying(error)))
    }
}
```

您也可以在此完成网络请求的日志输出，因为这个闭包在request发送到网络之前每次都会被调用。

# (供应者)Providers

当使用Moya时, 您通过MoyaProvider实例进行所有API请求,并把指定要调用哪个Endpoint的enum的值传递给它。在你设置了 [Endpoint](Endpoints.md)之后, 基本用法实际上配置完毕了:

```swift
let provider = MoyaProvider<MyService>()
```

在如此简单的设置之后您就可以直接使用了:

```swift
provider.request(.zen) { result in
    // `result` is either .success(response) or .failure(error)
}
```

到此完毕! `request()` 方法返回一个`Cancellable`, 它有一个你可以取消request的公共的方法。 更多关于`Result`类型的信息查看 [Examples](Examples)

记住,  把target和provider放在*哪儿*完全取决于您自己。 您可以查看 [Artsy的实现](https://github.com/artsy/eidolon/blob/master/Kiosk/App/Networking/ArtsyAPI.swift)
的例子.

但是别忘了持有它的一个引用 . 如果它被销毁了你将会在response上看到一个 `-999 "canceled"` 错误 。

## 高级用法

为了解释 `MoyaProvider`所有的配置选项我们将会按照下面的小节一个一个的来解析 。

### （endpoint闭包）endpointClosure:

  `MoyaProvider` 构造器的第一个(可选的)参数是一个
endpoints闭包, 它负责把您的enum值映射成一个`Endpoint`实例对象。 让我们看看它是什么样子的。

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint in
    let url = URL(target: target).absoluteString
    return Endpoint(url: url, sampleResponseClosure: {.networkResponse(200, target.sampleData)}, method: target.method, task: target.task)
}
let provider = MoyaProvider(endpointClosure: endpointClosure)
```

注意在这个`MoyaProvider`的构造器中我们不再有指定泛型 ，因为Swift将会自动从`endpointClosure`的类型中推断出来。 非常灵巧!

您有可能已经注意到了`URL(target:)` 构造器, Moya 提供了一个便利扩展来从任意 `TargetType`中创建 `URL`。

这个`endpointClosure`就像您看到的这样简单. 它其实也是Moya的默认实现， 这个实现存储在 `MoyaProvider.defaultEndpointMapping`.
查看 [Endpoints](Endpoints.md) 文档来查看 _为什么_ 您可能想自定义这个。

### （请求闭包）requestClosure:

下一个初始化参数是`requestClosure`,它分解一个`Endpoint` 成一个实际的 `URLRequest`. 同样的, 查看 [Endpoints](Endpoints.md)
文档了解为什么及如何来做这个 。

### （stub闭包）stubClosure:

下一个选择是来提供一个`stubClosure`。这个闭包返回 `.never` (默认的), `.immediate` 或者可以把stub请求延迟指定时间的`.delayed(seconds)`三个中的一个。 例如, `.delayed(0.2)` 可以把每个stub 请求延迟0.2s. 这个在单元测试中来模拟网络请求是非常有用的。

更棒的是如果您需要对请求进行区别性的stub，那么您可以使用自定义的闭包。

```swift
let provider = MoyaProvider<MyTarget>(stubClosure: { target: MyTarget -> Moya.StubBehavior in
    switch target {
        /* Return something different based on the target. */
    }
})
```

但通常情况下，您希望所有目标都有同样的stub行为。在 `MoyaProvider`中有三个静态方法您可以使用。

```swift
MoyaProvider.neverStub
MoyaProvider.immediatelyStub
MoyaProvider.delayedStub(seconds)
```

所以,在上面的示例上,如果您希望为所有的target立刻进行stub行为，下面的两种方式都可行 。

```swift
let provider = MoyaProvider<MyTarget>(stubClosure: { (_: MyTarget) -> Moya.StubBehavior in return .immediate })
let provider = MoyaProvider<MyTarget>(stubClosure: MoyaProvider.immediatelyStub)
```

### （管理器）manager:

接下来就是`manager`参数. 默认您将会获得一个基本配置的自定义的`Alamofire.Manager`实例对象

```swift
public final class func defaultAlamofireManager() -> Manager {
    let configuration = URLSessionConfiguration.default
    configuration.httpAdditionalHeaders = Alamofire.Manager.defaultHTTPHeaders

    let manager = Alamofire.Manager(configuration: configuration)
    manager.startRequestsImmediately = false
    return manager
}
```

这儿只有一个需要注意的事情: 由于在AF中创建一个`Alamofire.Request`默认会立即触发请求，即使为单元测试进行  "stubbing" 请求也一样。 因此在Moya中, `startRequestsImmediately` 属性被默认设置成了 `false` 。

如果您喜欢自定义自己的 manager, 比如, 添加SSL pinning, 创建一个并且添加到manager,
所有请求将通过自定义配置的manager进行路由.

```swift
let policies: [String: ServerTrustPolicy] = [
    "example.com": .PinPublicKeys(
        publicKeys: ServerTrustPolicy.publicKeysInBundle(),
        validateCertificateChain: true,
        validateHost: true
    )
]

let manager = Manager(
    configuration: URLSessionConfiguration.default,
    serverTrustPolicyManager: ServerTrustPolicyManager(policies: policies)
)

let provider = MoyaProvider<MyTarget>(manager: manager)
```

### 插件:

最后, 您可能也提供一个`plugins`数组给provider。 这些插件会在请求被发送前及响应收到后被执行。 Moya已经提供了一些插件: 一个是 网络活动(`NetworkActivityPlugin`),一个是记录所有的 网络活动 (`NetworkLoggerPlugin`), 还有一个是 [HTTP Authentication](Authentication.md).

例如您可以通过传递 `[NetworkLoggerPlugin()]` 给 `plugins`参考来开启日志记录 。注意查看也可以配置的, 比如，已经存在的 `NetworkActivityPlugin` 需要一个 `networkActivityClosure` 参数. 可配置的插件实现类似这样的:

```swift
public final class NetworkActivityPlugin: PluginType {

    public typealias NetworkActivityClosure = (change: NetworkActivityChangeType) -> ()
    let networkActivityClosure: NetworkActivityClosure

    public init(networkActivityClosure: NetworkActivityClosure) {
        self.networkActivityClosure = networkActivityClosure
    }

    // MARK: Plugin

    /// Called by the provider as soon as the request is about to start
    public func willSend(request: RequestType, target: TargetType) {
        networkActivityClosure(change: .began)
    }

    /// Called by the provider as soon as a response arrives
    public func didReceive(data: Data?, statusCode: Int?, response: URLResponse?, error: ErrorType?, target: TargetType) {
        networkActivityClosure(change: .ended)
    }
}
```

`networkActivityClosure` 是一个当网络请求开始或结束时提供通知的闭包。 这个和 [network activity indicator](https://github.com/thoughtbot/BOTNetworkActivityIndicator)一起来用是非常有用的。
注意这个闭包的签名是 `(change: NetworkActivityChangeType) -> ()`,
所以只有当请求是`.began` 或者`.ended`（您没有提供任何关于网络请求的细节） 时您才会被通知。

# 身份验证

身份验证变化多样。可以通过一些方法对网络请求进行身份验证。让我们来讨论常见的两种。

## 基本的HTTP身份验证

HTTP身份验证是一个 username/password HTTP协议内置的验证方式. 如果您需要使用 HTTP身份验证, 当初始化provider的时候可以使用一个 `CredentialsPlugin`
。

```swift
let provider = MoyaProvider<YourAPI>(plugins: [CredentialsPlugin { _ -> URLCredential? in
        return URLCredential(user: "user", password: "passwd", persistence: .none)
    }
])
```

这个特定的例子显示了HTTP的使用，它验证 _每个_ 请求,
通常这是不必要的。下面的方式可能更好:

```swift
let provider = MoyaProvider<YourAPI>(plugins: [CredentialsPlugin { target -> URLCredential? in
        switch target {
        case .targetThatNeedsAuthentication:
            return URLCredential(user: "user", password: "passwd", persistence: .none)
        default:
            return nil
        }
    }
])
```

## 访问令牌认证
另一个常见的身份验证方法就是通过使用一个访问令牌。
Moya提供一个 `AccessTokenPlugin` 来完成
 [JWT](https://jwt.io/introduction/)的 `Bearer` 认证 和 `Basic` 认证 。

 开始使用`AccessTokenPlugin`之前需要两个步骤.

1. 您需要把 `AccessTokenPlugin` 添加到您的`MoyaProvider`中，就像下面这样:

```Swift
let token = "eyeAm.AJsoN.weBTOKen"
let authPlugin = AccessTokenPlugin { token }
let provider = MoyaProvider<YourAPI>(plugins: [authPlugin])
```

`AccessTokenPlugin` 构造器接收一个`tokenClosure`闭包来负责返回一个可以被添加到request头部的令牌 。

2. 您的 `TargetType` 需要遵循`AccessTokenAuthorizable` 协议:

```Swift
extension YourAPI: TargetType, AccessTokenAuthorizable {
    case targetThatNeedsBearerAuth
    case targetThatNeedsBasicAuth
    case targetDoesNotNeedAuth

    var authorizationType: AuthorizationType {
        switch self {
            case .targetThatNeedsBearerAuth:
                return .bearer
            case .targetThatNeedsBasicAuth:
                return .basic
            case .targetDoesNotNeedAuth:
                return .none
            }
        }
}
```

`AccessTokenAuthorizable` 协议需要您实现一个属性 , `authorizationType`, 是一个枚举值，代表用于请求的头

**Bearer HTTP 认证**
Bearer 请求通过向HTTP头部添加下面的表单来获得授权:

```
Authorization: Bearer <token>
```

**Basic API Key 认证**
Basic 请求通过向HTTP头部添加下面的表单来获得授权

```
Authorization: Basic <token>
```

## OAuth

OAuth 有些麻烦。 它涉及一个多步骤的过程，在不同的api之间通常是不同的。 您 _确实_ 不想自己来做OAuth –
这儿有其他的库为您服务. [Heimdallr.swift](https://github.com/rheinfabrik/Heimdallr.swift),
例如. The trick is just getting Moya and whatever you're using to talk
to one another.

Moya内置了OAuth思想。 使用OAuth的网络请求“签名”本身有时会要求执行网络请求，所以对Moya的请求是一个异步的过程。让我们看看一个例子。

```swift
let requestClosure = { (endpoint: Endpoint, done: MoyaProvider.RequestResultClosure) in
    let request = endpoint.urlRequest // This is the request Moya generates
    YourAwesomeOAuthProvider.signRequest(request, completion: { signedRequest in
        // The OAuth provider can make its own network calls to sign your request.
        // However, you *must* call `done()` with the signed so that Moya can
        // actually send it!
        done(.success(signedRequest))
    })
}
let provider = MoyaProvider<YourAPI>(requestClosure: requestClosure)
```

(注意 Swift能推断出您的 `YourAPI` 类型)

## 在您的Provider子类中处理session刷新

您可以查看在每个请求前session刷新的示例[Examples/SubclassingProvider](Examples/SubclassingProvider.md).
它是基于 [Artsy's networking implementation](https://github.com/artsy/eidolon/blob/master/Kiosk/App/Networking/Networking.swift).
# 身份验证

身份验证变化多样。可以通过一些方法对网络请求进行身份验证。让我们来讨论常见的两种。

## 基本的HTTP身份验证

HTTP身份验证是一个 username/password HTTP协议内置的验证方式. 如果您需要使用 HTTP身份验证, 当初始化provider的时候可以使用一个 `CredentialsPlugin`
。

```swift
let provider = MoyaProvider<YourAPI>(plugins: [CredentialsPlugin { _ -> URLCredential? in
        return URLCredential(user: "user", password: "passwd", persistence: .none)
    }
])
```

这个特定的例子显示了HTTP的使用，它验证 _每个_ 请求,
通常这是不必要的。下面的方式可能更好:

```swift
let provider = MoyaProvider<YourAPI>(plugins: [CredentialsPlugin { target -> URLCredential? in
        switch target {
        case .targetThatNeedsAuthentication:
            return URLCredential(user: "user", password: "passwd", persistence: .none)
        default:
            return nil
        }
    }
])
```

## 访问令牌认证
另一个常见的身份验证方法就是通过使用一个访问令牌。
Moya提供一个 `AccessTokenPlugin` 来完成
 [JWT](https://jwt.io/introduction/)的 `Bearer` 认证 和 `Basic` 认证 。

 开始使用`AccessTokenPlugin`之前需要两个步骤.

1. 您需要把 `AccessTokenPlugin` 添加到您的`MoyaProvider`中，就像下面这样:

```Swift
let token = "eyeAm.AJsoN.weBTOKen"
let authPlugin = AccessTokenPlugin { token }
let provider = MoyaProvider<YourAPI>(plugins: [authPlugin])
```

`AccessTokenPlugin` 构造器接收一个`tokenClosure`闭包来负责返回一个可以被添加到request头部的令牌 。

2. 您的 `TargetType` 需要遵循`AccessTokenAuthorizable` 协议:

```Swift
extension YourAPI: TargetType, AccessTokenAuthorizable {
    case targetThatNeedsBearerAuth
    case targetThatNeedsBasicAuth
    case targetDoesNotNeedAuth

    var authorizationType: AuthorizationType {
        switch self {
            case .targetThatNeedsBearerAuth:
                return .bearer
            case .targetThatNeedsBasicAuth:
                return .basic
            case .targetDoesNotNeedAuth:
                return .none
            }
        }
}
```

`AccessTokenAuthorizable` 协议需要您实现一个属性 , `authorizationType`, 是一个枚举值，代表用于请求的头

**Bearer HTTP 认证**
Bearer 请求通过向HTTP头部添加下面的表单来获得授权:

```
Authorization: Bearer <token>
```

**Basic API Key 认证**
Basic 请求通过向HTTP头部添加下面的表单来获得授权

```
Authorization: Basic <token>
```

## OAuth

OAuth 有些麻烦。 它涉及一个多步骤的过程，在不同的api之间通常是不同的。 您 _确实_ 不想自己来做OAuth –
这儿有其他的库为您服务. [Heimdallr.swift](https://github.com/rheinfabrik/Heimdallr.swift),
例如. The trick is just getting Moya and whatever you're using to talk
to one another.

Moya内置了OAuth思想。 使用OAuth的网络请求“签名”本身有时会要求执行网络请求，所以对Moya的请求是一个异步的过程。让我们看看一个例子。

```swift
let requestClosure = { (endpoint: Endpoint, done: MoyaProvider.RequestResultClosure) in
    let request = endpoint.urlRequest // This is the request Moya generates
    YourAwesomeOAuthProvider.signRequest(request, completion: { signedRequest in
        // The OAuth provider can make its own network calls to sign your request.
        // However, you *must* call `done()` with the signed so that Moya can
        // actually send it!
        done(.success(signedRequest))
    })
}
let provider = MoyaProvider<YourAPI>(requestClosure: requestClosure)
```

(注意 Swift能推断出您的 `YourAPI` 类型)

## 在您的Provider子类中处理session刷新

您可以查看在每个请求前session刷新的示例[Examples/SubclassingProvider](Examples/SubclassingProvider.md).
它是基于 [Artsy's networking implementation](https://github.com/artsy/eidolon/blob/master/Kiosk/App/Networking/Networking.swift).

# RxSwift

Moya 在`MoyaProvider`中提供了一个可选的`RxSwift`实现，它可以做些有趣的事情。我们使用 `Observable`而不使用`request()`及请求完成时的回调闭包。

使用reactive扩展您不需要任何额外的设置。只使用您的 `MoyaProvider`实例对象 。

```swift
let provider = MoyaProvider<GitHub>()
```

简单设置之后, 您就可以使用了：

```swift
provider.rx.request(.zen).subscribe { event in
    switch event {
    case .success(let response):
        // do something with the data
    case .error(let error):
        // handle the error
    }
}
```

您也可以使用 `requestWithProgress` 来追踪您请求的进度 :

```swift
provider.rx.requestWithProgress(.zen).subscribe { event in
    switch event {
    case .next(let progressResponse):
        if let response = progressResponse.response {
            // do something with response
        } else {
            print("Progress: \(progressResponse.progress)")
        }
    case .error(let error):
        // handle the error
    default:
        break
    }
}
```

请务必记住直到signal被订阅之后网络请求才会开始。signal订阅者在网络请求完成前被销毁了，那么这个请求将被取消 。

如果请求正常完成，两件事件将会发生：

1. 这个信号将发送一个值，即一个 `Moya.Response` 实例对象.
2. 信号结束.

如果这个请求产生了一个错误 (通常一个 URLSession 错误),
然后它将发送一个错误. 这个错误的 `code` 就是失败请求的状态码, if any, and the response data, if any.

`Moya.Response` 类包含一个 `statusCode`, 一个 `data`,
和 一个( 可选的) `HTTPURLResponse`. 您可以在 `subscribe ` 或 `map` 回调中随意使用这些值。

为了让事情更加简便, Moya 为`Single` 和 `Observable`提供一些扩展来更容易的处理`MoyaResponses`。

- `filter(statusCodes:)` 指定一范围的状态码。如果响应的状态代码不是这个范围内,会产生一个错误。
- `filter(statusCode:)` 查看指定的一个状态码，如果没找到会产生一个错误。
- `filterSuccessfulStatusCodes()` 过滤 200-范围内的状态码.
- `filterSuccessfulStatusAndRedirectCodes()` 过滤 200-300 范围内的状态码。
- `mapImage()` 尝试把响应数据转化为 `UIImage` 实例
  如果不成功将产生一个错误。
- `mapJSON()` 尝试把响应数据映射成一个JSON对象，如果不成功将产生一个错误。
- `mapString()` 把响应数据转化成一个字符串，如果不成功将产生一个错误。
- `mapString(atKeyPath:)` 尝试把响应数据的key Path 映射成一个字符串，如果不成功将产生一个错误。

在错误的情况下, 错误的 `domain`是 `MoyaErrorDomain`。code
的值是`MoyaErrorCode`的其中一个的`rawValue`值。 只要有可能，会提供underlying错误并且原始响应数据会被包含在`NSError`的字典类型的`userInfo`的data中

# 线程

默认,您所有的请求将会被`Alamofire`放入background线程中, 响应将会在主线程中调用。如果您希望您的响应在不同的线程中调用 , 您可以用一个指定的 `callbackQueue`来初始化您的provider:

```swift
provider = MoyaProvider<GitHub>(callbackQueue: DispatchQueue.global(.utility))
provider.request(.userProfile("ashfurrow")) {
    /* this is called on a utility thread */
}
```

使用 `RxSwift` 或者 `ReactiveSwift` 您可以使用 `observeOn(_:)` 或者 `observe(on:)` 来实现类似的的行为:

## RxSwift
```swift
provider = MoyaProvider<GitHub>()
provider.rx.request(.userProfile("ashfurrow"))
  .map { /* this is called on the current thread */ }
  .observeOn(ConcurrentDispatchQueueScheduler(qos: .utility))
  .map { /* this is called on a utility thread */ }
```

## ReactiveSwift
```swift
provider = MoyaProvider<GitHub>()
provider.reactive.request(.userProfile("ashfurrow"))
  .map { /* this is called on the current thread */ }
  .observe(on: QueueScheduler(qos: .utility))
  .map { /* this is called on a utility thread */ }
```

# 插件

Moya的插件是被用来编辑请求、响应及完成副作用的。 插件调用:

- (`prepare`)  Moya 已经分解 `TargetType` 成 `URLRequest`之后被执行.
  这是请求被发送前进行编辑的一个机会 (例如 添加
  headers).
- (`willSend`) 请求将要发送前被执行. 这是检查请求和执行任何副作用(如日志)的机会。
- (`didReceive`) 接收到一个响应后被执行. 这是一个检查响应和执行副作用的机会。
- (`process`) 在 `completion` 被调用前执行. 这是对`request`的`Result`进行任意编辑的一个机会。

## 内置插件
Moya附带了一些用于常见功能的默认插件: 身份验证, 网络活动指示器管理 和 日志记录.
您可以在构造provider的时候申明插件来使用它:

```swift
let provider = MoyaProvider<GitHub>(plugins: [NetworkLoggerPlugin(verbose: true)])
```

### 身份验证
身份验证插件允许用户给每个请求赋值一个可选的 `URLCredential` 。当收到请求时，没有操作

这个插件可以在 [`Sources/Moya/Plugins/CredentialsPlugin.swift`](../Sources/Moya/Plugins/CredentialsPlugin.swift)中找到

### 网络活动指示器
在iOS网络中一个非常常见的任务就是在网络请求是显示一个网络活动指示器，当请求完成时移除它。提供的插件添加了回调，当请求开始和结束时调用，它可以用来跟踪正在进行的请求数，并相应的显示和隐藏网络活动指示器。

这个插件可以在 [`Sources/Moya/Plugins/NetworkActivityPlugin.swift`](../Sources/Moya/Plugins/NetworkActivityPlugin.swift)中找到

### 日志记录
在开发期间，将网络活动记录到控制台是非常有用的。这可以是任何来自发送和接收请求URL的内容，来记录每个请求和响应的完整的header，方法，请求体。

The provided plugin for logging is the most complex of the provided plugins, and can be configured to suit the amount of logging your app (and build type) require. When initializing the plugin, you can choose options for verbosity, whether to log curl commands, and provide functions for outputting data (useful if you are using your own log framework instead of `print`) and formatting data before printing (by default the response will be converted to a String using `String.Encoding.utf8` but if you'd like to convert to pretty-printed JSON for your responses you can pass in a formatter function, see the function `JSONResponseDataFormatter` in [`Examples/_shared/GitHubAPI.swift`](../Examples/_shared/GitHubAPI.swift) for an example that does exactly that)

这个插件可以在 [`Sources/Moya/Plugins/NetworkLoggerPlugin.swift`](../Sources/Moya/Plugins/NetworkLoggerPlugin.swift)中找到

## 自定义插件

Every time you need to execute some pieces of code before a request is sent and/or immediately after a response, you can create a custom plugin, implementing the `PluginType` protocol.
For examples of creating plugins, see [`docs/Examples/CustomPlugin.md`](Examples/CustomPlugin.md) and [`docs/Examples/AuthPlugin.md`](Examples/AuthPlugin.md).

# 社区项目

我们感谢围绕Moya周围的社区所做的一切工作。如果想让您的作品出现在下面列表中，只要简单的向列表中添加它（如果没有合适的标题，请自行添加一个）并且发起一个pull request。
## Extensions

- [Moya-ObjectMapper](https://github.com/ivanbruel/Moya-ObjectMapper) - Moya的ObjectMapper封装，易于JSON序列化
- [Moya-SwiftyJSONMapper](https://github.com/AvdLee/Moya-SwiftyJSONMapper) - Moya的SwiftyJSON 封装，易于JSON序列化
- [Moya-Argo](https://github.com/wattson12/Moya-Argo) - Moya 的Argo 封装，易于JSON序列化
- [Moya-ModelMapper](https://github.com/sunshinejr/Moya-ModelMapper) - Moya 的ModelMapper 封装，易于JSON序列化
- [Moya-Gloss](https://github.com/spxrogers/Moya-Gloss) - Moya 的Gloss 封装，易于JSON序列化
- [Moya-JASON](https://github.com/DroidsOnRoids/Moya-JASON) - Moya 的JASON 封装，易于JSON序列化
- [Moya-JASONMapper](https://github.com/AvdLee/Moya-JASONMapper) - Moya 的JASON封装，易于JSON序列化
- [Moya-Unbox](https://github.com/RyogaK/Moya-Unbox) - Moya 的Unbox 封装，易于JSON序列化
- [MoyaSugar](https://github.com/devxoul/MoyaSugar) –  Moya语法糖
- [Moya-EVReflection](https://github.com/evermeer/EVReflection/tree/master/Source/Alamofire/Moya) - Moya 的EVReflection 封装，易于JSON序列化 (包括子项目  [RxSwift](https://github.com/evermeer/EVReflection/tree/master/Source/Alamofire/Moya/RxSwift) 和 [ReactiveCocoa](https://github.com/evermeer/EVReflection/tree/master/Source/Alamofire/Moya/ReactiveCocoa))
- [Moya-Marshal](https://github.com/JARMourato/Moya-Marshal) - Moya 的Marshal 封装，易于JSON序列化
- [Moya-Decodable](https://github.com/xiaoyaogaojian/Moya-Decodable) - Decodable bindings for Moya for easier JSON serialization
- [SwiftMoyaCodeGenerator](https://github.com/narlei/SwiftMoyaCodeGenerator) - Extension for [Paw Mac App](https://paw.cloud) to generate Moya files based in rest documentation.


## 库

- [NetClient/Moya](https://github.com/intelygenz/NetClient-iOS) - 基于Swift3的通用的HTTP网络库。

## 应用

- [Drrrible](https://github.com/devxoul/Drrrible) - Dribbble for iOS using ReactorKit
- [Eidolon](https://github.com/artsy/eidolon) - The Artsy Auction Kiosk App
- [Insights for Instagram](https://github.com/adimango/insights-for-instagram) - A simple iOS Instagram's media insights App written in Swift 4
- [Papr](https://github.com/jdisho/Papr/tree/papr-moya-version) - An Unsplash app for iOS.
- [SwiftHub](https://github.com/khoren93/SwiftHub) - Reactive Github iOS client written in RxSwift and MVVM.

# 发布

(_Note: This document is a reference for people with push access to Moya and to [CocoaPods](https://cocoapods.org/pods/Moya)._)

## Before release

Releasing a new version of Moya has been automated as much as possible. There are a few prerequisite steps:

1. [Generate a GitHub personal access token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/)
1. Run the following command:
```ruby
echo "machine api.github.com login {GITHUB_LOGIN} password {GITHUB_TOKEN}" > ~/.netrc
```
Where `{GITHUB_LOGIN}` is your GitHub login and `{GITHUB_TOKEN}` is your personal access token generated in step 1 (or if you had one before). Example:
```ruby
echo "machine api.github.com login ashfurrow password dc14e6ac2b871e7630f56df3d57d2694b576316a" > ~/.netrc
```
This lets the automated release script access the GitHub API authorized as you.
1. Then run `chmod 600 ~/.netrc`.
1. Make sure you have a registered CocoaPods session. To do that you can run command:
```ruby
pod trunk me
```
If you see an error command that you do not have registered session, run command below:
```ruby
pod trunk register you@youremailaddress.com
```

## Release

(_Note: To make a release, you need at least one entry in the `Next` section of the changelog._)

To make a release:

1. Pull latest from master and make sure your git is clean (the script will fail if it's not).
1. Run `rake release["X.Y.Z"]`. (If you use ZSH, use `rake release\["X.Y.Z"\]`)
1. Grab a :tea: or :coffee:.
1. Make sure everything went smoothly.

What you'll need to do manually afterwards (if you released a major version):

1. Update the Swift Package Manager instructions in the Readme to use the release you just made public.

If anything goes wrong, don't panic! Get in touch with someone else who has released, or [Ash](mailto:ash@ashfurrow.com).
