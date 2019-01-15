---
layout: post
title: 第三方类库应该如何进行错误处理
category: 技术
date: 2019-1-14 23:10
---

## 从一个 BestHTTP 的 Bug 说起
今天在 Unity 游戏中碰到一个奇怪的问题，差不多费了半天的时间才定位出问题，略有所感。
在游戏中我们使用了 BestHTTP 这个第三方的类库来发送 HTTP 请求，并且，业务逻辑上首先发送一个 http://api.host1.com/gets 这样一个接口，
该接口会返回另外一个 URL Host 比如：http://api.host2.com/，然后，后续所有的 API 的 host 都基于该新返回的 host2, 再拼接上其余的 URL Path。

在 BestHTTP 内部，有一个函数 GetRequestPathAndQueryURL 是负责解析 URL 的，如下：

```C#

public static string GetRequestPathAndQueryURL(this Uri uri)
{
    string requestPathAndQuery = uri.GetComponents(UriComponents.PathAndQuery, UriFormat.UriEscaped);

    // http://forum.unity3d.com/threads/best-http-released.200006/page-26#post-2723250
    if (string.IsNullOrEmpty(requestPathAndQuery))
        requestPathAndQuery = "/";

    return requestPathAndQuery;
}
```

直接调用 GetRequestPathAndQueryURL 的函数的内部处理如下：

```C#
internal void SendOutTo(Stream stream)
{
    try
    {
        string requestPathAndQuery = urrentUri.GetRequestPathAndQueryURL();

    }
    finally
    {
        if (UploadStream != null && DisposeUploadStream)
            UploadStream.Dispose();
    }
}

```

SendOutTo 的上层调用者，如下：
```C#

try
{
#if !NETFX_CORE
     Client.NoDelay = CurrentRequest.TryToMinimizeTCPLatency;
#endif
     CurrentRequest.SendOutTo(Stream);

     sentRequest = true;
}
catch (Exception ex)
{
    Close();

    if (State == HTTPConnectionStates.TimedOut ||
        State == HTTPConnectionStates.AbortRequested)
        throw new Exception("AbortRequested");

    // We will try again only once
    if (!alreadyReconnected && !CurrentRequest.DisableRetry)
    {
        alreadyReconnected = true;
        cause = RetryCauses.Reconnect;
    }
    else // rethrow exception
        throw ex;
}

```

由于服务器开发人员不小心在返回的 URL 字符串的开始处多加了一个空格，既" http://api.host2.com"，导致 `GetRequestPathAndQueryURL()` 函数在解析时会抛出异常。
但遗憾的是 BestHTTP 并没有正确的处理这个异常，既在 `SendOutTo()` 函数中，只有 `try{} finally {}`, 并没有 `catch（）`。
这样，GetRequestPathAndQueryURL 所抛出的异常会被更上一层的函数捕获处理。而更上层的函数则只是把网络连接关闭，
并抛出了另外一个与 URL 解析异常毫不相干的异常：`throw new Exception("AbortRequested")` 。


从现象上来看，游戏中的第一个请求 request 发出和 response 返回处理都是正常的，而第二个请求则直接被终止了，使用抓包工具发现，第二个请求连发出都不曾发出就失败了。
当出现这个 Bug 时，根据异常信息，我的第一反应是怀疑发送请求的管理队列在状态处理方面是不是有Bug，根本没有想到会是 URL 解析错误的问题。

更难办的是，上面所列出的代码都被 BestHTTP 封装在一个 DLL 中，根本无法在其内部加断点调试，也就很难发现那个被 "狸猫换太子" 的 URL 解析失败的异常。
而且，该错误只有在 Windows 平台下才有这个Bug，在 Mac 平台上没有问题。
最后，我是使用了 BestHTTP 的源码，才最终发现这个 Bug 的原因的，花费了好长时间。


## 第三方如何进行错误处理
从这个例子我们可以看到，之所以出现如此难查的问题，根本原因是因为 BestHTTP 在对异常的处理上有问题，它不应该把异常"狸猫换太子"，而是应该把最直接的异常信息打印出来。
在错误处理时，应该秉持"快速失败"原则，哪个地方有问题，就在哪个地方失败，这样方便开发者快速定位错误代码。特别是第三方库，由于其封装问题，很有可能不会以源代码的形式提供，
而是以 DLL 等函数库的方式提供，对于调试更加不便，更应该"快速失败"，并打印出足够多的 Log 信息。




 

