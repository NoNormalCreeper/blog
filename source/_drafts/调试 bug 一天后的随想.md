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

这里出现了第一个问题，由于第二行的 `fiber.Ctx.Request()` 返回的是一个 `*fasthttp.Request` 对象，但 `HandleSignInCallback()` 接受的参数是一个标准库的 `*http.Request` 对象，因此需要转化。我写了一行注释，简单让 Copilot 补全了一下，他给我补上了这些代码：

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
		ctx.String(http.StatusInternalServerError, err.Error())
		return
	}
}
```

看起来很棒，跑了一下完全可以跑通了。

开始一个未认证错误吓我一跳，提示的是 OAuth 里的错误，\`redirect_uri\` did not match any of the client's registered \`redirect_uris\`.

然后发现只是我漏填了一个 `/auth` ，地址直接填了 `/oidc/callback`，修正后一下子就.......还是失败了，但是错误信息变得看似清晰实则扑朔迷离了起来。


 ![](https://pics.r1kka.one/file/1743474249583_img_v3_02kt_1f6da1d6-6e23-4de4-81fc-1d1e15ec85cg.jpg)







**施工中........**
