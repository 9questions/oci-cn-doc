# CDN CloudFlare 接入 OCI 对象存储

## 1 OCI 中创建对象存储

<figure><img src=".gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

### 1.1 创建对象存储(示例中为默认配置)

<figure><img src=".gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

### 1.2 CDN 接入对象存储访问类型

CDN 厂商接入的两种方式：

* **在普通访问模式下，OCI 对象存储需要配置为公共访问类型才能被CDN访问，默认创建的对象存储是私有的**
* **在私有模式下，配置 Pre-Authenticated 的访问方式，允许 CDN 接入 OCI 对象存储**

<figure><img src=".gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

修改为公共访问，允许其他CDN厂商链接

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

## 2 OCI对象存储上传测试图片

在 对象存储 页面中点击 upload，进行文件的上传

<figure><img src=".gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

上传文件

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

## 3 配置cloudflare 加入 OCI 对象存储

### 3.1 登录已注册站点

### 3.2 配置DNS CNAME 解析到 OCI 对象存储

法兰克福 endpoint ： [objectstorage.eu-frankfurt-1.oraclecloud.com](http://objectstorage.eu-frankfurt-1.oraclecloud.com)

其他参考region参考：[https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/](https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/)

<figure><img src=".gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

### 3.3 添加 Workers 服务

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

选用HTTP处理程序

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

### 3.4 编写处理请求服务代码

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  // url.host = 'YOUR_OBJECT_STORAGE_BUCKET_ENDPOINT' // OCI 对象存储 endpoint
  url.host = 'objectstorage.eu-frankfurt-1.oraclecloud.com'  // 使用法兰克福节点
  const newRequest = new Request(url, request)
  return fetch(newRequest)
}
```

<figure><img src=".gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

### 3.5 配置站点缓存页面规则

<figure><img src=".gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

示例中规则是对s3.xxx-chang.com域名下的所有内容进行缓存

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

### 3.6 配置站点的路由规则

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

配置当前CDN域名 到 指定的路由规则

<figure><img src=".gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

## 4 通过CDN域名访问对象存储



```
格式：
https://<域名>/n/<namespace>/b/<bucket>/o/<文件名>

示例 - 普通模式：
https://s3.xxx-chang.com/n/sehubjapacprod/b/bucket-20230418-1654/o/achicken.jpg

示例 - Pre-Authenticated模式：
https://s3.xxx-chang.com/p/b5B0q4auDc_g8u-Bk9taoLGZox94GsUfAnz-CSe9dCKRxAW8H5IkpRWoDRlSOUze/n/sehubjapacprod/b/bucket-20230419-1036/o/chicken.jpg
```

## 5 通过 CloudFlare worker 替换 content-type 类型

### 5.1 背景

通过 SDK ( upload\_obj ) 上传至对象存储的文件，如果不指定 content-type 默认会被指定为 octet-stream；目前已知此 content-type 类型会导致前端矢量图(.svg后缀)的展示失效，并且 oci 对象存储不支持自适应。

### 5.2 解决方式

客户目前正在使用cloudFlare进行缓存，通过 CloudFlare worker 实现对 svg 后缀文件的 content-type 进行替换，统一替换为image/svg+xml。

### 5.3 替换

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // const response = await fetch(request)

  const url = new URL(request.url)  
  // url.host = 'objectstorage.eu-frankfurt-1.oraclecloud.com'  // 使用法兰克福节点
  url.host = '<命名空间>.compat.objectstorage.eu-frankfurt-1.oraclecloud.com'
  const newRequest = new Request(url, request)

  // 发起实际请求
  const response = await fetch(newRequest)
  // 获取请求路径
  const path = url.pathname
   // 检查文件后缀是否为 '.svg'
   if (path.endsWith('.svg')) {
    // 修改内容类型为 'image/svg+xml'
    const modifiedHeaders = new Headers(response.headers)
    modifiedHeaders.set('Content-Type', 'image/svg+xml')

    // 返回经过修改的响应
    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: modifiedHeaders
    })
  }
 // 直接返回原始响应
 return response

```

## 6 参考

[https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/](https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/)
