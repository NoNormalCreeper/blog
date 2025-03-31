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

我自作聪明地查找之后直接清除掉了数据，想着应该不会第二次再查找了。然而这在后面抓耳挠腮的时候并不是主要的 bug。
