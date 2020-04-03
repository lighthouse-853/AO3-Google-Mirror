# 853号灯塔计划 AO3等网站的反向代理镜像搭建
## 目录
1. 简介
2. 利用Cloudflare Workers搭建反向代理
3. 高级功能实现
4. 绑定自定义域名
5. 编辑防火墙规则开启验证码/地区限制
6. 利用Cloudflare Access 进行访问控制（按需开启）
## 简介
这一方案基于Cloudflare边缘计算网络上的无服务器应用Workers，无需服务器部署nginx和域名（按需）
利用Cloudflare Workers 支持的JS反向代理搭建AO3镜像/Google/中文维基百科镜像

详细教程实现地址：https://blog.lighthouse-853.icu/2020/04/cloudflare-worker-Mirror-AO3-Wikipedia.html <br>
AO3镜像：https://3.lighthouse-853.icu/ <br>
中文维基百科镜像：https://blog.lighthouse-853.icu/2020/04/Wikipedia-Mirror.html <br>
*#为避免配额过快消耗，此镜像只允许中国大陆访问，首次访问需要接受Captcha验证码质询* <br>
*#配合火狐浏览器的ESNI功能与DNS over HTTPS效果更佳*


## 利用Cloudflare Workers搭建反向代理

在Cloudflare中Open一个Workers项目，粘贴如下的Javascript

    // 反代目标网站（AO3域名较简单）.
    const upstream = 'archiveofourown.org'

    // 反代目标网站的移动版.
    const upstream_mobile = 'archiveofourown.org'

    // 访问区域黑名单（按需设置）.
    const blocked_region = ['TK']

    // IP地址黑名单（按需设置）.
    const blocked_ip_address = ['0.0.0.0', '127.0.0.1']

    // 路径替换.
    const replace_dict = {
    '$upstream': '$custom_domain',
    '//archiveofourown.org': ''
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
    let url_host = url.host;

    if (url.protocol == 'http:') {
        url.protocol = 'https:'
        response = Response.redirect(url.href);
        return response;
    }

    if (await device_status(user_agent)) {
        var upstream_domain = upstream;
    } else {
        var upstream_domain = upstream_mobile;
    }

    url.host = upstream_domain;

    if (blocked_region.includes(region)) {
        response = new Response('Access denied: WorkersProxy is not available in your region yet.', {
            status: 403
        });
    } else if(blocked_ip_address.includes(ip_address)){
        response = new Response('Access denied: Your IP address is blocked by WorkersProxy.', {
            status: 403
        });
    } else{
        let method = request.method;
        let request_headers = request.headers;
        let new_request_headers = new Headers(request_headers);

        new_request_headers.set('Host', upstream_domain);
        new_request_headers.set('Referer', url.href);

        let original_response = await fetch(url.href, {
            method: method,
            headers: new_request_headers
        })

        let original_response_clone = original_response.clone();
        let original_text = null;
        let response_headers = original_response.headers;
        let new_response_headers = new Headers(response_headers);
        let status = original_response.status;

        new_response_headers.set('cache-control' ,'public, max-age=14400')
        new_response_headers.set('access-control-allow-origin', '*');
        new_response_headers.set('access-control-allow-credentials', true);
        new_response_headers.delete('content-security-policy');
        new_response_headers.delete('content-security-policy-report-only');
        new_response_headers.delete('clear-site-data');

        const content_type = new_response_headers.get('content-type');
        if (content_type.includes('text/html') && content_type.includes('UTF-8')) {
            original_text = await replace_response_text(original_response_clone, upstream_domain, url_host);
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


    async function device_status (user_agent_info) {
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
之后就可以使用Cloudflare workers分配的 https://XXX.yourproject.workers.dev 直接访问
## 高级功能实现
### 绑定自定义域名
*需要将自定义域名添加到Cloudflare管理，不能使用其他DNS或Partner接入，否则可能无法颁发SSL证书*
1. 进入**网站管理**页面内的Workers选项卡，选择**添加路由**，输入Https://ao3.your-domain.com/* 绑定域名到刚刚创建的Workers项目 （不要漏掉***Https***与最后的星号）
2. 在DNS解析中添加CNAME记录，将ao3.your-domain.com指向XXX.yourproject.workers.dev，并**点亮橙云**
### 编辑Cloudflare防火墙规则开启验证码/地区限制
#### 添加验证码避免配额消耗
在防火墙规则中添加表达式 <br>  

    (http.host eq "ao3.your-domain.com")
选择操作“质询（Captcha）”或“JS质询” ***(建议选择前者)***
#### 开启访问地区限制，并拒绝机器人和高风险IP访问
在防火墙规则中添加表达式 <br>  

    (not ip.geoip.country in {"CN" } and http.host eq "ao3.your-domain.com") or (cf.client.bot and http.host eq "ao3.your-domain.com") or (cf.threat_score ge 25 and http.host eq "ao3.your-domain.com")

选择操作“阻止”
### 利用Cloudflare Access 进行访问控制（按需开启）
*#需要绑定付款方式，信用卡或Paypal进行验证*    
在Access中添加ao3.your-domain.com，可以选择只让指定用户访问此镜像，验证方式支持邮箱接收一次性验证码/Github OAuth

![Cloudflare Access规则]（http://github.com/lighthouse-853/Lighthouse-Mirror/raw/master/images_folder/cloudflareAccess.png）
具体配置方式参考Cloudflare Support：https://developers.cloudflare.com/access/setting-up-access/configuring-access-policies/
## 致敬
***屏蔽维基百科，将是对审查者的极大讽刺，因为我们不是他们宣称的任何一种审查对象；审查维基百科等于承认，恰恰是中立的事实数据令其担惊受怕。我们不是政治宣传机器，我们不是网络赌博，我们不是色情。我们是一部百科全书。    ——维基百科计划的创始人 Jimmy Wales ***
