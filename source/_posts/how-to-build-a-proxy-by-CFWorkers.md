---
title: how-to-build-a-proxy-by-CFWorkers
date: 2020-11-22 23:42:41
tags: techs
---
## 背景
Cloudflare是著名的CDN提供商，有着庞大的边缘网络。![cloudflare边缘网络]('www.cloudflare.com/resources/images/slt3lc6tev37/6DtsbzZuXhp10jg8oJhoqj/de2a4824fae910e96e0bb6987d42a7be/illustration_network-map_animation.gif') 他们提供了一项构建无服务器应用程序的CFWorkers,可以把JS脚本快速部署到CF的边缘网络上来搞事情。

也就是说我们可以通过CFWorkers搭建反向代理来实现~~科学上网~~。

第一次见到这个操作还是某个sd群友的[谷歌反代站](gogoogle.ml)不过因为不可抗力因素这个反代站已经关闭了......所以建议学会反代的同学自己偷着乐就可以了。而且有的网站禁止反代，比如[P站](www.pixiv.net);有的网站反代没用，比如[GMAIL](mail.google.com)，因为涉及到账号登陆。不过[反代个wiki](wiki.xzbloggers.cn)啥的还是很香的

## 准备
- Cloudflare账户
- 一个域名（可要可不要，workers默认部署的子域太长）

## 实现
首先登录你的Cloudflare账户，选择页面右侧的

```
Workers
构建无服务器应用程序

```
在新页面单击

```

创建Worker

```
接着在左侧脚本框内输入以下代码

``` JS
// Website you intended to retrieve for users.
const upstream = 'www.example.com'

// Custom pathname for the upstream website.
const upstream_path = '/'

// Website you intended to retrieve for users using mobile devices.
const upstream_mobile = 'www.example.com'

// Countries and regions where you wish to suspend your service.
const blocked_region = ['KP', 'SY', 'PK', 'CU']

// IP addresses which you wish to block from using your service.
const blocked_ip_address = ['0.0.0.0', '127.0.0.1']

// Whether to use HTTPS protocol for upstream address.
const https = true

// Whether to disable cache.
const disable_cache = true

// Replace texts.
const replace_dict = {
    '$upstream': '$custom_domain',
    '//www.example.com': ''
}

addEventListener('fetch', event => {
    event.respondWith(fetchAndApply(event.request));
})

async function fetchAndApply(request) {

    const region = request.headers.get('cf-ipcountry').toUpperCase();
    const ip_address = request.headers.get('cf-connecting-ip');
    const user_agent = request.headers.get('user-agent');

    let response = null;
    let url = new URL(request.url);
    let url_hostname = url.hostname;

    if (https == true) {
        url.protocol = 'https:';
    } else {
        url.protocol = 'http:';
    }

    if (await device_status(user_agent)) {
        var upstream_domain = upstream;
    } else {
        var upstream_domain = upstream_mobile;
    }

    url.host = upstream_domain;
    if (url.pathname == '/') {
        url.pathname = upstream_path;
    } else {
        url.pathname = upstream_path + url.pathname;
    }

    if (blocked_region.includes(region)) {
        response = new Response('Access denied: WorkersProxy is not available in your region yet.', {
            status: 403
        });
    } else if (blocked_ip_address.includes(ip_address)) {
        response = new Response('Access denied: Your IP address is blocked by WorkersProxy.', {
            status: 403
        });
    } else {
        let method = request.method;
        let request_headers = request.headers;
        let new_request_headers = new Headers(request_headers);

        new_request_headers.set('Host', url.hostname);
        new_request_headers.set('Referer', url.hostname);

        let original_response = await fetch(url.href, {
            method: method,
            headers: new_request_headers
        })

        let original_response_clone = original_response.clone();
        let original_text = null;
        let response_headers = original_response.headers;
        let new_response_headers = new Headers(response_headers);
        let status = original_response.status;
        
        if (disable_cache) {
            new_response_headers.set('Cache-Control', 'no-store');
        }

        new_response_headers.set('access-control-allow-origin', '*');
        new_response_headers.set('access-control-allow-credentials', true);
        new_response_headers.delete('content-security-policy');
        new_response_headers.delete('content-security-policy-report-only');
        new_response_headers.delete('clear-site-data');
        
        if(new_response_headers.get("x-pjax-url")) {
            new_response_headers.set("x-pjax-url", response_headers.get("x-pjax-url").replace("//" + upstream_domain, "//" + url_hostname));
        }
        
        const content_type = new_response_headers.get('content-type');
        if (content_type.includes('text/html') && content_type.includes('UTF-8')) {
            original_text = await replace_response_text(original_response_clone, upstream_domain, url_hostname);
        } else {
            original_text = original_response_clone.body
        }
        
        response = new Response(original_text, {
            status,
            headers: new_response_headers
        })
    }
    return response;
}

async function replace_response_text(response, upstream_domain, host_name) {
    let text = await response.text()

    var i, j;
    for (i in replace_dict) {
        j = replace_dict[i]
        if (i == '$upstream') {
            i = upstream_domain
        } else if (i == '$custom_domain') {
            i = host_name
        }

        if (j == '$upstream') {
            j = upstream_domain
        } else if (j == '$custom_domain') {
            j = host_name
        }

        let re = new RegExp(i, 'g')
        text = text.replace(re, j);
    }
    return text;
}


async function device_status(user_agent_info) {
    var agents = ["Android", "iPhone", "SymbianOS", "Windows Phone", "iPad", "iPod"];
    var flag = true;
    for (var v = 0; v < agents.length; v++) {
        if (user_agent_info.indexOf(agents[v]) > 0) {
            flag = false;
            break;
        }
    }
    return flag;
}
```

在右侧选项卡选择预览选项卡并刷新检查是否反代成功。
成功后保存并退出到上一层页面，重命名一个好记的名字，并打开左侧

```

已在workers.dev子域中提供

```
好了，一个基于CFWorkers的反代站已经做好了，把它收藏在浏览器收藏夹里就可以方便地调用了。

## 使用客制化域名

首先你得准备好一个域名，不妨假设是 www.example.com 。在域名的注册商那儿把该域名的DNS服务器改为Cloudflare的DNS服务器。具体请参见如何在Cloudflare中添加站点。
然后，在