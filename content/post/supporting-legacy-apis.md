+++
date = "2016-06-14T19:46:32-07:00"
draft = false
title = "Supporting Legacy APIs"
subtitle = "for fun and profit"

+++

I've started learning Elixir, so I've been on the prowl for great blog content to learn from. I just found [this](https://blog.fourk.io/replace-your-production-api-with-elixir-today-4426a8903642) article, and it's hitting a lot of the right buttons for me. I love that he's up-front about the fact that most of us can't big-bang release a production API in a new language! I love that he introduces a way to progressively migrate over!

I love it so much that I've done this myself pretty recently in Go (with a twist)! Maybe this idea will be useful for others.

<!--more-->

I introduced progressive migration to support old versions of mobile apps. Like most developers, when releasing a new version of an app I tend to improve the server API at the same time. But for mobile apps, the existing pesky old versions can't be ignored. So I reverse proxy.

```go
var oldVersionProxy = httputil.NewSingleHostReverseProxy(
    "https://old.api.com",
)

func NewVersionRouter() http.Handler {
    router := new(httprouter.Router)

    router.GET("/api/foo", FooHandler)
    router.POST("/api/bar", BarHandler)

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        handle, params, _ := router.Lookup(r.Method, r.URL.Path)
        if handle != nil {
            handle(w, r, params)
            return
        }

        oldVersionProxy.ServeHTTP(w, r)
    })
}
```

There's not that much to this code in particular. First it sets up a router and sets some routes. Then instead of using it directly, it calls the Lookup function to find a matching route. If it doesn't find one, it falls back to a reverse proxy.

You could do this just as well with an http.ServeMux. Run oldVersionProxy in a "/" handler as the fallback mechanism. I just generally use httprouter, so this is the code I have.

This enables you to migrate an API server route-by-route, just like in the Elixir post. But I said there was a twist!

The key is that when updating an API, you usually swap out endpoint implementations one-for-one. A new endpoint will perform the same job as it's predecessor, just maybe with different request or response formats. So here's the twist: `FooHandler` and `BarHandler` above are just shims. They accept an old request format, translate it to the new corresponding request, and send that back to the new API server. Then they translate the new response to the old version before sending that back out to the client.

How is this helpful? **You can now delete old code while still supporting an old API**. Route all requests to your new API server and run it though the new (no doubt improved) logic. Those hitting the routes of old versions are translated as necessary.
