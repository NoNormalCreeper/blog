---
abbrlink: ''
categories: []
date: '2025-04-01T00:16:29.572860+08:00'
tags: []
title: bug-day
updated: '2025-04-01T00:19:19.248+08:00'
---
# 调试 bug 一天后的随想

今天在尝试写一个 OIDC 认证的中间件和一些 API，使用 GoFiber 作为 web 框架，去接入 Logto 的 OIDC 服务。

“这应该还是比较简单的吧！”我在 Bing 里面输入了这些框架相关的关键词，看着一个个弹出来的链接，一边滑动鼠标滚轮一边想。

进入 Logto 文档，熟练地找到了 Quick Start 标签，一眼找到了 Go 的图标。“有官方 SDK 啊，那肯定稳了”，我高兴了起来。

![](https://pics.r1kka.one/file/1743438390415_%E5%9B%BE%E7%89%87.png)

文档首先指导我创建一个会话存储，用于存储会话的临时 key 等信息，让我实现两个接口即可：

```go
type Storage interface {
	GetItem(key string) string
	SetItem(key, value string)
}
```

可惜官方给的案例和我的不太相同，因此我直接用内存里存 map 实现了简易的的存储和读取接口。

```go
type SimpleStorage struct {
	storage map[string]string
}

// NewSimpleStorage 创建一个新的 SimpleStorage 实例
func NewSimpleStorage() *SimpleStorage {
	return &SimpleStorage{
		storage: make(map[string]string),
	}
}

func (s *SimpleStorage) GetItem(key string) string {
	// 从内存中获取值，获取后自动清除 map 里的对应数据
	if value, ok := s.storage[key]; ok {
		delete(s.storage, key)
		log.Printf("从内存中获取值: %s = %s", key, value)
		return value
	}
	// 如果没有找到，返回空字符串
	return ""
}

func (s *SimpleStorage) SetItem(key, value string) {
	// 将值存储到内存中
	log.Printf("存储值到内存: %s = %s", key, value)
	s.storage[key] = value
}
```

我自作聪明地查找之后直接清除掉了数据，想着应该不会第二次再查询值了。然而这在后面抓耳挠腮的时候并不是主要的 bug。

在注册路由的地方添加了两个路由，开始实现处理方法。

```go
// RegisterRoutes 注册所有认证相关路由
func (h *AuthHandler) RegisterRoutes(app *fiber.App, userHandler *UserHandler) {
	auth := app.Group("/auth")

	authStrict := auth.Group("/", StrictRateLimiter())
	authStrict.Get("/oidc/login", h.OidcLogin)
	authStrict.Get("/oidc/callback", h.OidcCallback)
}
```

其中，`/oidc/login` 的路由很好实现，官方文档中给出了 SDK 示例，有一个内置的 `SignIn(callbackUrl string) error` 方法，照着抄传入回调地址就完事：

```go
func (h *AuthHandler) OidcLogin(ctx *fiber.Ctx) error {
	backendUrl := os.Getenv("BACKEND_URL")
	signInUri, err := h.logtoClient.SignIn(backendUrl + "/auth/oidc/callback")
	if err != nil {
		log.Printf("OIDC登录失败: %v", err)
		// 错误响应....
	}

	err = ctx.Redirect(signInUri, fiber.StatusFound)
	if err != nil {
		log.Printf("重定向失败: %v", err)
		// 错误响应....
	}

	return nil
}
```

回调地址 `/oidc/callback` 的实现，我也是抄的官方文档，也有一个方法 `HandleSignInCallback(request *http.Request) error `直接负责处理回调请求。因为我的应用并不需要处理重定向到首页和存储 Cookie，只负责返回一个 JWT，所以也十分简单：

```go
func (h *AuthHandler) OidcCallback(ctx *fiber.Ctx) error {
	err := h.logtoClient.HandleSignInCallback(ctx.Request())
	if err != nil {
		ctx.String(http.StatusInternalServerError, err.Error())
		return
	}
}
```

这里出现了第一个问题，由于第二行的 `fiber.Ctx.Request()` 返回的是一个 `*fasthttp.Request` 对象，但 `HandleSignInCallback()` 接受的参数是一个标准库的 `*http.Request` 对象，因此需要转化。我写了一行注释，简单让 Copilot 补全了一下，他给我补上了一些代码，全部实现：

```go
func (h *AuthHandler) OidcCallback(ctx *fiber.Ctx) error {
    // Convert the fiber/fasthttp request to a standard net/http request
    req := &http.Request{
        Method: string(ctx.Method()),
        URL: &url.URL{
            Scheme:   string(ctx.Protocol()),
            Host:     string(ctx.Hostname()),
            Path:     string(ctx.Path()),
            RawQuery: string(ctx.Request().URI().QueryString()),
        },
        Header: make(http.Header),
    }

    // Copy headers
    ctx.Request().Header.VisitAll(func(key, value []byte) {
        req.Header.Add(string(key), string(value))
    })
    err := h.logtoClient.HandleSignInCallback(req)
    if err != nil {
        log.Printf("处理OIDC回调失败: %v", err)
        // 错误响应...
        return
    }
}
```

看起来是手动把参数填进去，很棒，跑了一下完全可以跑通了。

开始一个未认证错误吓我一跳，提示的是 OAuth 里的错误，\`redirect_uri\` did not match any of the client's registered \`redirect_uris\`.

然后发现只是我漏填了一个 `/auth` ，地址直接填了 `/oidc/callback`，修正后一下子就.......还是失败了，但是错误信息变得看似清晰实则扑朔迷离了起来。

![](https://pics.r1kka.one/file/1743474249583_img_v3_02kt_1f6da1d6-6e23-4de4-81fc-1d1e15ec85cg.jpg)

我检查了各个地方的 `callback uri` 和 `redirect uri`，发现都是一样的，输出调试日志发现也是一样的。

![](https://pics.r1kka.one/file/1743474815050_%E5%9B%BE%E7%89%87.png)

在 Google 搜了一圈居然没搜到相关的报错，查了 SDK 文档也没有说明。实际上文档只是在 pkg.go.dev 里面列出了各个结构和方法的定义而已，几乎没有作出说明，最头疼的就是这种。

这时我看到 pkg.go.dev 里面的版本是 v2.0.0，而我这里的版本仍是 v1，于是我进行了升级，并使用了新的函数 `SignInWithRedirectUri(callbackUrl)`，结果无济于事，该报错还是报错。

注：后续发现这个方法内部的实现如下，只是对原先的方法进行了一个包装，其实效果是一样的

```go
func (logtoClient *LogtoClient) SignInWithRedirectUri(redirectUri string) (string, error) {
	return logtoClient.SignIn(&SignInOptions{
		RedirectUri: redirectUri,
	})
}
```

丢给 AI 改了一圈，也只是让我输出调试日志和让我改一些莫名其妙的小错误，一直没有帮助。卡了两三个小时在改这些地方和搜没用的资料上。

用 debugger 打断点看到各个地方的 URL 也没什么差异，我还去搜了一下 redirect uri 和 callback uri 的差别，也无果。

一直到了晚上，纠结得头疼，感觉是 SDK 哪里给我自作聪明做了奇怪的验证的问题，但又实在是不想抛弃 SDK 去手搓请求，还是决定还是去看看 SDK 的源码，希望能找到问题。

Ctrl+Click 进入 `SignIn(options *SignInOptions) (string, error)` 方法，看了下好像没发现哪里有问题；又看了下 callback 路由实现里的 `HandleSignInCallback(request *http.Request) error` 方法，很大一堆也没看出头绪来。这时我一不小心把 ESC 按成了 F1，一个全局搜索框出现在我眼前。

我突然想到可以搜索，便把错误信息粘贴了进去，菊花转了几圈后，一个 `Err` 开头的变量出现了，让我顿时感觉找对了地方：`ErrCallbackUriNotMatchRedirectUri`

查看定义：

```go
var ErrCallbackUriNotMatchRedirectUri = errors.New("callback uri not match redirect uri")
```

这不正是我要找的吗！查看 usage，只有两个，其中一个还是测试文件。这下稳了！点进引用，我在对应的方法第一行就看到了这个错误：

```go
func VerifyAndParseCodeFromCallbackUri(callbackUri, redirectUri, state string) (string, error) {
	if !strings.HasPrefix(callbackUri, redirectUri) {
		return "", ErrCallbackUriNotMatchRedirectUri
	}
	// ...
```

果然是 SDK 自作聪明做的验证..........我把中间的 `return` 语句临时注释掉，报错变成了 token 格式错误，我想可能是前面查询完一次就自动清除的 `SimpleStorage` 爆雷了。先不管他，总不能一直注释掉吧，还是得看看问题出在哪。

再跳转到这个函数的引用，也是除了测试外就一个，发现就是上面的 `HandleSignInCallback(...)`，可能当时没仔细看，没看出来罢。

```go
func (logtoClient *LogtoClient) HandleSignInCallback(request *http.Request) error {
	signInSession := SignInSession{}
	parseSignInSessionErr := json.Unmarshal([]byte(logtoClient.storage.GetItem(StorageKeySignInSession)), &signInSession)

	if parseSignInSessionErr != nil {
		return parseSignInSessionErr
	}

	callbackUri := GetOriginRequestUrl(request)
	code, retrieveCodeErr := core.VerifyAndParseCodeFromCallbackUri(callbackUri, signInSession.RedirectUri, signInSession.State)
	if retrieveCodeErr != nil {
		return retrieveCodeErr
	}
	// ...
```

其中：

```go
func GetOriginRequestUrl(request *http.Request) string {
	return getRequestProtocol(request) + "://" + request.Host + request.RequestURI
}
```

各个地方都打个断点看看吧，果然发现了不对的地方。

![](https://pics.r1kka.one/file/1743477073363_%E5%9B%BE%E7%89%87.png)

此处（`VerifyAndParseCodeFromCallbackUri`）里的参数似乎不太准确，`callbackUri` 是不准确的，怪不得会验证不通过。这个变量来自于 `GetOriginRequestUrl(request)` 函数的处理，这个函数没问题，那就是.........传入的 `request` 参数的问题了？

又打了断点查看 `request` 参数，这下真相大白了。

![](https://pics.r1kka.one/file/1743477532696_%E5%9B%BE%E7%89%87.png)

该对象的 `RequestURI` 属性为空，因此，传入 `GetOriginRequestUrl(request *http.Request)` 之后，取出来的 `request.RequestURI` 也为空，所以返回值就是 `<portocal>://<host>` 了，没有后面的 `path`。这也就说明了为什么上面的 `callbackUri` 有残缺。

那就改呗。这次我不敢相信 copilot 了，直接在 Bing 上搜索转化方法，结果还是 AI 生成的结果。依然没有设置 `RequestURI` 参数，于是我只好自己添加上去了。

```go
// Convert the fiber/fasthttp request to a standard net/http request
func convertFastHTTPToHTTPRequest(fastReq *fasthttp.Request) (*http.Request, error) {
	// Create a new http.Request
	req, err := http.NewRequest(
		string(fastReq.Header.Method()),
		fastReq.URI().String(),
		bytes.NewReader(fastReq.Body()),
	)
	if err != nil {
		return nil, err
	}

	// 此处我自己添加.... Set the request URL
	req.RequestURI = string(fastReq.RequestURI())

	// Copy headers
	fastReq.Header.VisitAll(func(key, value []byte) {
		req.Header.Set(string(key), string(value))
	})

	return req, nil
}
```

改完后能跑通了，问题变成了 token 格式错误，拉出调试器一看，果然是调用了两次从存储中取值的方法，于是我把我的储存实现从读取一次自动清除改为了 60 分钟后自动删除，这下应该不会出问题了。修改后的定义和实现如下：

```go
type SimpleStorage struct {
	storage map[string]StorageItem
}

func (s *SimpleStorage) Cleanup() {
	for key, item := range s.storage {
		if time.Now().After(item.ExpiresAt) {
			delete(s.storage, key)
			log.Printf("清除过期的存储项: %s", key)
		}
	}
}

func (s *SimpleStorage) GetItem(key string) string {
	if value, ok := s.storage[key]; ok {
		log.Printf("从内存中获取值: %s = %s", key, value.Value)
		return value.Value
	}
	s.Cleanup()

	log.Printf("未找到内存中的值: %s", key)
	return ""
}

func (s *SimpleStorage) SetItem(key, value string) {
	expiration := time.Now().Add(1 * time.Hour)
	s.storage[key] = StorageItem{
		Value:     value,
		ExpiresAt: expiration,
	}
	log.Printf("存储值到内存: %s = %s", key, value)
	s.Cleanup()
}
```

至此接口的实现都没有问题了，最后添加上获取用户信息并生成 JWT 令牌，明天再写保护其他 API 中间件吧，累了。

我心满意足地躺上了床，又感觉这个 bug 是那种找了一天找不出来，最后发现很简单的类型。虽然有找到问题的满足，但也有对自己死活意识不到问题，开始走了那么多弯路的不甘（）

