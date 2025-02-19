### https://google.map987.us.kg/
# google-image-cloudflare-workers-proxy
使用cloudflare workers谷歌搜图

https://github.com/copilot/c/edfd1dcf-3c89-48d3-8e4c-d71f55a0e141

```
 
 https://www.n.cn/search/acfea815279148e38a2f2bc917af38f5?fr=none
根据您的Nginx配置文件，要实现访问 `http://127.0.0.1:8080/` 直接代理到百度，需修改 location 匹配规则。当前配置仅代理了 `/proxy/baidu` 路径，而根路径 `/` 未配置代理，因此出现404错误。
 
---
 
修改方案（分步骤说明）
 
1. 修改 location 匹配规则 
将原有配置中的：
```nginx
location /proxy/baidu {{
    proxy_pass https://www.baidu.com;
    ...
}}
```
改为：
```nginx
location / {{
    proxy_pass https://www.baidu.com;
    proxy_set_header Host www.baidu.com;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_hide_header X-Frame-Options;
    proxy_ssl_server_name on;  # 若百度启用了SNI，需添加此配置 
}}
```
 
2. 关键配置说明 
- `location /`：匹配根路径请求，所有访问 `http://127.0.0.1:8080/` 的请求都会被代理。
- `proxy_ssl_server_name on`：解决SSL证书的SNI（Server Name Indication）问题，确保Nginx能正确识别百度域名。
- `proxy_set_header`：传递客户端真实IP和域名信息，避免被目标服务器拒绝。
 
3. 最终配置文件示例 
```nginx
server {{
    listen 8080;
    server_name localhost;
 
    location / {{
        proxy_pass https://www.baidu.com;
        proxy_set_header Host www.baidu.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_hide_header X-Frame-Options;
        proxy_ssl_server_name on;
    }}
 
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {{
        root /data/data/com.termux/files/usr/share/nginx/html;
    }}
}}
```
 
---
 
验证与调试 
1. 重载Nginx配置：
   ```bash
   nginx -s reload 
   ```
2. 测试访问：
   - 浏览器访问 `http://127.0.0.1:8080` 应显示百度首页。
3. 排查问题：
   - 若出现502错误，检查Nginx与百度服务器的网络连通性（如DNS解析、防火墙）。
   - 查看Nginx错误日志：
     ```bash
     tail -f /path/to/nginx/logs/error.log 
     ```
 
---
 
扩展场景配置（按需选择）
1. 保留原有 `/proxy/baidu` 路径：
   ```nginx
   location / {{
       proxy_pass https://www.baidu.com;
       ...
   }}
 
   location /proxy/baidu {{
       proxy_pass https://www.baidu.com/s?wd=nginx;
       ...
   }}
   ```
   此时：
   - `http://127.0.0.1:8080/` 代理到百度首页 
   - `http://127.0.0.1:8080/proxy/baidu` 代理到百度搜索"nginx"的结果页 
 
2. 添加缓存优化：
   ```nginx
   proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=baidu_cache:10m;
   location / {{
       proxy_cache baidu_cache;
       proxy_pass https://www.baidu.com;
       ...
   }}
   ```
 
---
 
注意事项 
1. 合规性：代理第三方网站需遵守目标网站的`robots.txt`和服务条款，避免法律风险。
2. 性能：高并发代理可能导致Nginx资源消耗增加，建议限制连接数或启用缓存。
3. 安全性：若需对外网开放，应配置防火墙规则限制访问IP范围

#user  nobody; worker_processes  1;  #error_log  logs/error.log; #error_log  logs/error.log  notice; #error_log  logs/error.log  info;  #pid        logs/nginx.pid;   events {     worker_connections  1024; }   http {     include       mime.types;     default_type  application/octet-stream;      #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '     #                  '$status $body_bytes_sent "$http_referer" '     #                  '"$http_user_agent" "$http_x_forwarded_for"';      #access_log  logs/access.log  main;      sendfile        on;     #tcp_nopush     on;      #keepalive_timeout  0;     keepalive_timeout  65;      #gzip  on;      server {         listen       8080;         server_name  localhost;          #charset koi8-r;          #access_log  logs/host.access.log  main;                  location /proxy/baidu {         proxy_pass https://www.baidu.com;          proxy_set_header Host www.baidu.com;          proxy_set_header X-Real-IP $remote_addr;         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;         # 移除X-Frame-Options限制（如果目标网站允许）         proxy_hide_header X-Frame-Options;     }          #error_page  404              /404.html;          # redirect server error pages to the static page /50x.html         #         error_page   500 502 503 504  /50x.html;         location = /50x.html {             root   /data/data/com.termux/files/usr/share/nginx/html;         }          # proxy the PHP scripts to Apache listening on 127.0.0.1:80         #         #location ~ \.php$ {         #    proxy_pass   http://127.0.0.1;         #}          # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000         #         #location ~ \.php$ {         #    root           html;         #    fastcgi_pass   127.0.0.1:9000;         #    fastcgi_index  index.php;         #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;         #    include        fastcgi_params;         #}          # deny access to .htaccess files, if Apache's document root         # concurs with nginx's one         #         #location ~ /\.ht {         #    deny  all;         #}     }       # another virtual host using mix of IP-, name-, and port-based configuration     #     #server {     #    listen       8000;     #    listen       somename:8080;     #    server_name  somename  alias  another.alias;      #    location / {     #        root   html;     #        index  index.html index.htm;     #    }     #}       # HTTPS server     #     #server {     #    listen       443 ssl;     #    server_name  localhost;      #    ssl_certificate      cert.pem;     #    ssl_certificate_key  cert.key;      #    ssl_session_cache    shared:SSL:1m;     #    ssl_session_timeout  5m;      #    ssl_ciphers  HIGH:!aNULL:!MD5;     #    ssl_prefer_server_ciphers  on;      #    location / {     #        root   html;     #        index  index.html index.htm;     #    }     #}  }  如何实现访问http://127.0.0.1:8080/实现访问百度，现在是404页面。 




 
 
以下是整合反向代理与顶部菜单功能的完整Nginx配置方案，采用结构化分层实现：
 
```nginx

```
 
需同步创建菜单样式文件 `/data/data/com.termux/files/usr/share/nginx/custom/menu.css`：
 
```css
/* 固定定位顶部菜单 */

```
 
关键实现原理：
1. 通过`sub_filter`实现HTML内容注入 
2. 独立CSS文件服务保障样式加载 
3. 固定定位确保菜单全局可见 
4. 响应头处理解决内容压缩问题 
5. 动态调整body间距避免内容遮挡 
 
测试建议：
1. 检查nginx是否包含`ngx_http_sub_module`
2. 使用开发者工具验证CSS加载情况 
3. 测试不同分辨率下的菜单表现 
4. 验证代理页面功能完整性 
 
扩展优化方向：
- 添加JavaScript实现动态菜单 
- 响应式布局适配移动端 
- 集成用户登录状态模块 
- 增加菜单过渡动画效果 
- 多语言支持实现 
 
该方案在保持反向代理功能完整性的基础上，通过内容注入技术实现了非侵入式菜单集成，适用于各种被代理页面的统一导航需求。 

```
```
Skip to content
Navigation Menu

Copilot Chat
HTML page with iframe and search controls
https://c.map987.us.kg/https://google.com.hk/search?q=TV+%E3%82%A2%E3%83%8B%E3%83%A1+-eeo.today&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg&tbs=qdr:w&imgsz=8mp
实现一个页面，iframe插入上面的html完整在本html中，
也就是一个html实际上是可以放在另外一个html中的一部分的，比如live2d做成一个html，放在页面左下角，

然后实现一个面板，可以显示 参数q，可以更改q的参数然后再次搜索显示新的html，可以选择 imgsz 为 2mp 4… 8 15 20 40 70 ，或者为空，可以选择 tbs=qdr:w 的w为d w m y 或者默认空
Let me help you create an HTML file that incorporates these requirements. I'll create a page with an iframe and controls to modify the search parameters.


index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
This HTML file includes:

A control panel at the top with:

A text input for the search query (q parameter)
A dropdown to select image size (imgsz parameter)
A dropdown to select time range (tbs=qdr parameter)
An "Update Search" button
An iframe below the control panel that displays the search results

Functionality:

The page preserves the original proxy URL structure (c.map987.us.kg)
You can modify the search query, image size, and time range
The "Update Search" button will reload the iframe with the new parameters
The iframe takes up 80% of the viewport height
All parameters are properly encoded
The initial values are set to match your example URL
To use this:

Save the file as index.html
Open it in a web browser
You can modify the search parameters using the control panel
Click "Update Search" to apply the changes
The page is responsive and will work on both desktop and mobile devices. The control panel has a clean, user-friendly interface with proper spacing and styling. The iframe will resize according to the window size while maintaining proper proportions.

这样单独html不行，我现在部署到cloudflare workers 中 ，源代码是 addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  try {
      const url = new URL(request.url);

      // 如果访问根目录，返回HTML
      if (url.pathname === "/") {
          return new Response(getRootHtml(), {
              headers: {
                  'Content-Type': 'text/html; charset=utf-8'
              }
          });
      }

      // 从请求路径中提取目标 URL
      let actualUrlStr = decodeURIComponent(url.pathname.replace("/", ""));

      // 判断用户输入的 URL 是否带有协议
      actualUrlStr = ensureProtocol(actualUrlStr, url.protocol);

      // 保留查询参数
      actualUrlStr += url.search;

      // 创建新 Headers 对象，排除以 'cf-' 开头的请求头
      const newHeaders = filterHeaders(request.headers, name => !name.startsWith('cf-'));

      // 创建一个新的请求以访问目标 URL
      const modifiedRequest = new Request(actualUrlStr, {
          headers: newHeaders,
          method: request.method,
          body: request.body,
          redirect: 'manual'
      });

      // 发起对目标 URL 的请求
      const response = await fetch(modifiedRequest);
      let body = response.body;

      // 处理重定向
      if ([301, 302, 303, 307, 308].includes(response.status)) {
          body = response.body;
          // 创建新的 Response 对象以修改 Location 头部
          return handleRedirect(response, body);
      } else if (response.headers.get("Content-Type")?.includes("text/html")) {
          body = await handleHtmlContent(response, url.protocol, url.host, actualUrlStr);
      }

      // 创建修改后的响应对象
      const modifiedResponse = new Response(body, {
          status: response.status,
          statusText: response.statusText,
          headers: response.headers
      });

      // 添加禁用缓存的头部
      setNoCacheHeaders(modifiedResponse.headers);

      // 添加 CORS 头部，允许跨域访问
      setCorsHeaders(modifiedResponse.headers);

      return modifiedResponse;
  } catch (error) {
      // 如果请求目标地址时出现错误，返回带有错误消息的响应和状态码 500（服务器错误）
      return jsonResponse({
          error: error.message
      }, 500);
  }
}

// 确保 URL 带有协议
function ensureProtocol(url, defaultProtocol) {
  return url.startsWith("http://") || url.startsWith("https://") ? url : defaultProtocol + "//" + url;
}

// 处理重定向
function handleRedirect(response, body) {
  const location = new URL(response.headers.get('location'));
  const modifiedLocation = `/${encodeURIComponent(location.toString())}`;
  return new Response(body, {
      status: response.status,
      statusText: response.statusText,
      headers: {
          ...response.headers,
          'Location': modifiedLocation
      }
  });
}

// 处理 HTML 内容中的相对路径
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
  const originalText = await response.text();
  const regex = new RegExp('((href|src|action)=["\'])/(?!/)', 'g');
  let modifiedText = replaceRelativePaths(originalText, protocol, host, new URL(actualUrlStr).origin);

  return modifiedText;
}

// 替换 HTML 内容中的相对路径
function replaceRelativePaths(text, protocol, host, origin) {
  const regex = new RegExp('((href|src|action)=["\'])/(?!/)', 'g');
  return text.replace(regex, `$1${protocol}//${host}/${origin}/`);
}

// 返回 JSON 格式的响应
function jsonResponse(data, status) {
  return new Response(JSON.stringify(data), {
      status: status,
      headers: {
          'Content-Type': 'application/json; charset=utf-8'
      }
  });
}

// 过滤请求头
function filterHeaders(headers, filterFunc) {
  return new Headers([...headers].filter(([name]) => filterFunc(name)));
}

// 设置禁用缓存的头部
function setNoCacheHeaders(headers) {
  headers.set('Cache-Control', 'no-store');
}

// 设置 CORS 头部
function setCorsHeaders(headers) {
  headers.set('Access-Control-Allow-Origin', '*');
  headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  headers.set('Access-Control-Allow-Headers', '*');
}

// 返回根目录的 HTML
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <link href="https://cdnjs.cloudflare.com/ajax/libs/materialize/1.0.0/css/materialize.min.css" rel="stylesheet">
  <title>Proxy Everything</title>
  <link rel="icon" type="image/png" href="https://img.icons8.com/color/1000/kawaii-bread-1.png">
  <meta name="Description" content="Proxy Everything with CF Workers.">
  <meta property="og:description" content="Proxy Everything with CF Workers.">
  <meta property="og:image" content="https://img.icons8.com/color/1000/kawaii-bread-1.png">
  <meta name="robots" content="index, follow">
  <meta http-equiv="Content-Language" content="zh-CN">
  <meta name="copyright" content="Copyright © ymyuuu">
  <meta name="author" content="ymyuuu">
  <link rel="apple-touch-icon-precomposed" sizes="120x120" href="https://img.icons8.com/color/1000/kawaii-bread-1.png">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0, user-scalable=no">
  <style>
      body, html {
          height: 100%;
          margin: 0;
      }
      .background {
          background-image: url('https://imgapi.cn/bing.php');
          background-size: cover;
          background-position: center;
          height: 100%;
          display: flex;
          align-items: center;
          justify-content: center;
      }
      .card {
          background-color: rgba(255, 255, 255, 0.8);
          transition: background-color 0.3s ease, box-shadow 0.3s ease;
      }
      .card:hover {
          background-color: rgba(255, 255, 255, 1);
          box-shadow: 0px 8px 16px rgba(0, 0, 0, 0.3);
      }
      .input-field input[type=text] {
          color: #2c3e50;
      }
      .input-field input[type=text]:focus+label {
          color: #2c3e50 !important;
      }
      .input-field input[type=text]:focus {
          border-bottom: 1px solid #2c3e50 !important;
          box-shadow: 0 1px 0 0 #2c3e50 !important;
      }
  </style>
</head>
<body>
  <div class="background">
      <div class="container">
          <div class="row">
              <div class="col s12 m8 offset-m2 l6 offset-l3">
                  <div class="card">
                      <div class="card-content">
                          <span class="card-title center-align"><i class="material-icons left">link</i>Proxy Everything</span>
                          <form id="urlForm" onsubmit="redirectToProxy(event)">
                              <div class="input-field">
                                  <input type="text" id="targetUrl" placeholder="在此输入目标地址" required>
                                  <label for="targetUrl">目标地址</label>
                              </div>
                              <button type="submit" class="btn waves-effect waves-light teal darken-2 full-width">跳转</button>
                          </form>
                      </div>
                  </div>
              </div>
          </div>
      </div>
  </div>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/materialize/1.0.0/js/materialize.min.js"></script>
  <script>
      function redirectToProxy(event) {
          event.preventDefault();
          const targetUrl = document.getElementById('targetUrl').value.trim();
          const currentOrigin = window.location.origin;
          window.open(currentOrigin + '/' + encodeURIComponent(targetUrl), '_blank');
      }
  </script>
</body>
</html>`;
}，也就是 https://c.map987.us.kg/https://google.com.hk/search?q=TV+%E3%82%A2%E3%83%8B%E3%83%A1+-eeo.today&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg&tbs=qdr:w&imgsz=8mp实际返回https://google.com.hk/search?q=TV+%E3%82%A2%E3%83%8B%E3%83%A1+-eeo.today&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa…  的内容，现在有没有办法在访问时候，不只是显示原网站，而是把上述html筛选的html添加到页面上去，我说的是主页面 
明白了，您想要在代理的页面上添加一个控制面板。我们需要修改 handleHtmlContent 函数来注入我们的控制面板。以下是修改后的代码：


worker.js
v1
// ... 其他代码保持不变 ...

// 修改 handleHtmlContent 函数
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
    const originalText = await response.text();
这段代码的主要变更：

在 handleHtmlContent 函数中添加了控制面板的 HTML 和 JavaScript 代码
控制面板固定在页面顶部
包含了搜索框、图片大小选择和时间范围选择
自动从当前 URL 中提取并设置当前的查询参数
点击 "Update Search" 按钮时会使用新的参数重新加载页面
控制面板的样式使用内联 CSS 确保不受原页面样式影响
使用较高的 z-index 确保控制面板始终显示在最上层
当用户访问代理的 Google 图片搜索页面时，会在页面顶部看到这个控制面板，可以：

修改搜索关键词
选择图片大小
选择时间范围
点击 "Update Search" 按钮应用新的搜索条件
控制面板会自动从当前 URL 中提取现有的参数并显示，使用户可以基于当前搜索进行调整。这样的实现方式不会影响原页面的功能，同时提供了便捷的搜索参数调整界面。

要使用这个修改，只需要将原来的 worker.js 代码替换为上面的代码即可。其他部分（如根路径的 HTML 页面等）都保持不变。

问题是 部署在 getRootHtml还是说每个具体页面比较好，如果在主页面 就是选择 imgsz=8mp，然后页面动态显示 https://c.map987.us.kg/https://google.com.hk/search?q=TV+%E3%82%A2%E3%83%8B%E3%83%A1+-eeo.today&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg&tbs=qdr:w&imgsz=8mp
实现一个页面，iframe插入上面的html完整在本html中，
也就是一个html实际上是可以放在另外一个html中的一部分的，比如live2d做成一个html，放在页面左下角，

然后实现一个面板，可以显示 参数q，可以更改q的参数然后再次搜索显示新的html，可以选择 imgsz 为 2mp 4… 8 15 20 40 70 ，或者为空，可以选择 tbs=qdr:w 的w为d w m y 或者默认空 动态显示出可点击的链接然后让我跳转
我明白了，您想要在主页面(getRootHtml)中实现这个功能，这样更合理。我们来修改 getRootHtml 函数：


worker.js
v2
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
Message interrupted
Copilot was interrupted before it could finish this message.
顶部的栏目，在手机版本没有显示全，还有就是应该默认填充为q 为关键词 TV+%E3%82%A2%E3%83%8B%E3%83%A1+-eeo.today，同时最好实时显示实际上跳转的网址
我理解您要改进的几个方面：

改进移动端适配
默认填充搜索关键词
实时显示将要跳转的完整URL
我来修改 getRootHtml 函数：


worker.js
v3
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
主要改进：

移动端适配改进：

使用 flex-wrap 和 min-width 确保控制项在手机上垂直堆叠
调整了padding和字体大小
优化了各个控件的响应式布局
默认值设置：

searchQuery 输入框默认值设为 "TV アニメ -eeo.today"
imageSize 默认选中 "8MP"
timeRange 默认选中 "一周内"
URL预览功能：

添加了 preview-url 区域显示完整的代理URL
实时更新：当任何参数改变时都会更新预览URL
使用 word-break: break-all 确保长URL正确换行
界面优化：

添加了更清晰的分组和标签
改进了控件间距和对齐
使用半透明白色背景提高可读性
添加了圆角和阴影效果
功能改进：

页面加载时自动执行首次搜索
添加了独立的加载按钮
iframe 容器美化，增加了边框和圆角
性能优化：

减少了不必要的DOM操作
优化了事件监听器的使用
使用方法保持不变，但现在界面在手机上更易用，默认值已经设置好，并且可以实时预览将要访问的URL。用户可以在修改任何参数时看到URL的变化，然后点击"加载搜索结果"按钮来执行搜索。

不仅仅是root页面，同时之前// ... 其他代码保持不变 ...

// 修改 handleHtmlContent 函数
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
    const originalText = await response.text();
    let modifiedText = replaceRelativePaths(originalText, protocol, host, new URL(actualUrlStr).origin);

    // 注入控制面板
    const controlPanel = `
    <div id="proxyControlPanel" style="
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        background: rgba(240, 240, 240, 0.95);
        padding: 10px;
        z-index: 999999;
        box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        display: flex;
        align-items: center;
        gap: 10px;
        font-family: Arial, sans-serif;
    ">
        <input type="text" id="searchQuery" value="${getQueryParam(actualUrlStr, 'q')}" 
            style="flex: 1; padding: 5px; border: 1px solid #ccc; border-radius: 3px;">
        
        <select id="imageSize" style="padding: 5px; border: 1px solid #ccc; border-radius: 3px;">
            <option value="">All sizes</option>
            <option value="2mp">2MP</option>
            <option value="4mp">4MP</option>
            <option value="8mp">8MP</option>
            <option value="15mp">15MP</option>
            <option value="20mp">20MP</option>
            <option value="40mp">40MP</option>
            <option value="70mp">70MP</option>
        </select>

        <select id="timeRange" style="padding: 5px; border: 1px solid #ccc; border-radius: 3px;">
            <option value="">Any time</option>
            <option value="d">Past 24 hours</option>
            <option value="w">Past week</option>
            <option value="m">Past month</option>
            <option value="y">Past year</option>
        </select>

        <button onclick="updateSearch()" style="
            padding: 5px 15px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 3px;
            cursor: pointer;
        ">Update Search</button>
    </div>

    <script>
    function updateSearch() {
        const query = encodeURIComponent(document.getElementById('searchQuery').value);
        const imageSize = document.getElementById('imageSize').value;
        const timeRange = document.getElementById('timeRange').value;

        let url = '${protocol}//${host}/https://google.com.hk/search?q=' + query + 
            '&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg';

        if (timeRange) {
            url += '&tbs=qdr:' + timeRange;
        }
        if (imageSize) {
            url += (timeRange ? ',' : '&tbs=') + 'imgsz:' + imageSize;
        }

        window.location.href = url;
    }

    // 设置当前值
    window.addEventListener('load', function() {
        const currentUrl = new URL(window.location.href);
        const tbs = new URLSearchParams(currentUrl.search).get('tbs') || '';
        
        if (tbs.includes('qdr:')) {
            const timeValue = tbs.match(/qdr:([dwmy])/)?.[1];
            if (timeValue) document.getElementById('timeRange').value = timeValue;
        }
        
        if (tbs.includes('imgsz:')) {
            const sizeValue = tbs.match(/imgsz:([^,&]*)/)?.[1];
            if (sizeValue) document.getElementById('imageSize').value = sizeValue;
        }
    });
    </script>`;

    // 在 </body> 标签前注入控制面板
    modifiedText = modifiedText.replace('</body>', controlPanel + '</body>');

    return modifiedText;
}

// 辅助函数：从 URL 中获取查询参数
function getQueryParam(urlStr, param) {
    try {
        const url = new URL(urlStr);
        return decodeURIComponent(url.searchParams.get(param) || '');
    } catch (e) {
        return '';
    }
}

// ... 其他代码保持不变 ... ，这个部分也需要显示链接和适配手机页面再顶部
好的，我来提供完整的代码，包括改进后的 handleHtmlContent 函数和相关辅助函数：


worker.js
v4
// 修改 handleHtmlContent 函数
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
    const originalText = await response.text();
    let modifiedText = replaceRelativePaths(originalText, protocol, host, new URL(actualUrlStr).origin);
主要改进：

移动端适配优化：

使用 CSS flexbox 和媒体查询实现响应式布局
在移动设备上控件垂直堆叠
调整了padding和字体大小
界面优化：

添加了内联样式表，更好地组织样式代码
改进了控件的视觉外观
添加了hover效果
优化了间距和对齐
URL预览功能：

添加了预览URL区域
实时更新预览URL
优化了URL的显示格式
功能改进：

添加了Enter键支持
所有控件值变化时实时更新预览URL
中文化界面文字
优化了控件的布局和大小
性能优化：

使用事件委托
优化了DOM操作
添加了错误处理
可用性改进：

更大的点击区域
更清晰的视觉反馈
更好的键盘支持
这个版本的控制面板会在任何代理页面的顶部显示，并且能够很好地适应各种屏幕尺寸。同时保持了原有的功能，但提供了更好的用户体验。

主页面里 加载搜索结果不是主要的，应该是跳转到对应链接，当然 加载搜索结果这个按钮也保存，还有就是更新搜索也应该是跳转
明白了，我来修改主页面的代码，增加跳转按钮，并修改更新搜索的行为：


worker.js
v5
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
Message interrupted
Copilot was interrupted before it could finish this message.
主页面里 加载搜索结果不是主要的，应该是跳转到对应链接，当然 加载搜索结果这个按钮也保存，还有就是更新搜索也应该是跳转
好的，我明白了。我会修改主页面和handleHtmlContent中的控制面板，让"更新搜索"也变成跳转行为，同时保留加载功能作为辅助选项。这里是修改后的代码：


worker.js
v6
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
主要改进：

所有的搜索更新都默认为新窗口打开
统一了主页和内嵌控制面板的行为模式
改进了UI设计，使其在移动端更易用
优化了预览URL的显示方式，使其可点击
简化了按钮操作，主要突出"新窗口打开"功能
优化了事件处理和性能
改进了移动端适配
统一了样式和交互体验
使用方式：

主页面：可以设置参数后选择"新窗口打开搜索页面"或"在当前页面加载"
搜索结果页面的控制面板：修改参数后可以直接点击预览URL或使用"新窗口打开"按钮跳转
所有的更新操作都会生成新的URL，并优先使用新窗口打开的方式，保持原页面状态。

下
Message interrupted
Copilot was interrupted before it could finish this message.
现在点击没有反应，背景图也没有加载 
我们来修复这些问题。主要是 materialize.css 的引用和一些功能没有正确实现。以下是修复后的主页面代码：


worker.js
v7
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
主要修复和改进：

移除了 Materialize CSS 依赖，改用原生 CSS
使用渐变背景替代图片背景，提高加载速度和稳定性
优化了所有点击事件的实现
改进了移动端适配
简化了 CSS 类名和结构
增加了更多的视觉反馈（hover 效果等）
优化了按钮和输入框的样式
改进了错误处理
增加了更多的空间和间距
优化了字体大小和颜色
功能特点：

可以直接输入搜索关键词
可以选择图片大小和时间范围
显示完整的代理 URL
提供两种跳转方式：新窗口打开和当前页面加载
支持回车键快速搜索
URL 预览区域可点击跳转
所有参数变化时实时更新 URL
现在界面应该可以正常工作，没有外部依赖，并且在各种设备上都能正常显示和使用。

Ask Copilot

我们来修复这些问题。主要是 materialize.css 的引用和一些功能没有正确实现。以下是修复后的主页面代码： worker.js function getRootHtml() { return `<!DOCTYPE html> <html lang="zh-CN"> <head> <meta charset="UTF-8"> 主要修复和改进： 移除了 Materialize CSS 依赖，改用原生 CSS 使用渐变背景替代图片背景，提高加载速度和稳定性 优化了所有点击事件的实现 改进了移动端适配 简化了 CSS 类名和结构 增加了更多的视觉反馈（hover 效果等） 优化了按钮和输入框的样式 改进了错误处理 增加了更多的空间和间距 优化了字体大小和颜色 功能特点： 可以直接输入搜索关键词 可以选择图片大小和时间范围 显示完整的代理 URL 提供两种跳转方式：新窗口打开和当前页面加载 支持回车键快速搜索 URL 预览区域可点击跳转 所有参数变化时实时更新 URL 现在界面应该可以正常工作，没有外部依赖，并且在各种设备上都能正常显示和使用。
```

```
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  try {
      const url = new URL(request.url);

      // 如果访问根目录，返回HTML
      if (url.pathname === "/") {
          return new Response(getRootHtml(), {
              headers: {
                  'Content-Type': 'text/html; charset=utf-8'
              }
          });
      }

      // 从请求路径中提取目标 URL
      let actualUrlStr = decodeURIComponent(url.pathname.replace("/", ""));

      // 判断用户输入的 URL 是否带有协议
      actualUrlStr = ensureProtocol(actualUrlStr, url.protocol);

      // 保留查询参数
      actualUrlStr += url.search;

      // 创建新 Headers 对象，排除以 'cf-' 开头的请求头
      const newHeaders = filterHeaders(request.headers, name => !name.startsWith('cf-'));

      // 创建一个新的请求以访问目标 URL
      const modifiedRequest = new Request(actualUrlStr, {
          headers: newHeaders,
          method: request.method,
          body: request.body,
          redirect: 'manual'
      });

      // 发起对目标 URL 的请求
      const response = await fetch(modifiedRequest);
      let body = response.body;

      // 处理重定向
      if ([301, 302, 303, 307, 308].includes(response.status)) {
          body = response.body;
          // 创建新的 Response 对象以修改 Location 头部
          return handleRedirect(response, body);
      } else if (response.headers.get("Content-Type")?.includes("text/html")) {
          body = await handleHtmlContent(response, url.protocol, url.host, actualUrlStr);
      }

      // 创建修改后的响应对象
      const modifiedResponse = new Response(body, {
          status: response.status,
          statusText: response.statusText,
          headers: response.headers
      });

      // 添加禁用缓存的头部
      setNoCacheHeaders(modifiedResponse.headers);

      // 添加 CORS 头部，允许跨域访问
      setCorsHeaders(modifiedResponse.headers);

      return modifiedResponse;
  } catch (error) {
      // 如果请求目标地址时出现错误，返回带有错误消息的响应和状态码 500（服务器错误）
      return jsonResponse({
          error: error.message
      }, 500);
  }
}

// 确保 URL 带有协议
function ensureProtocol(url, defaultProtocol) {
  return url.startsWith("http://") || url.startsWith("https://") ? url : defaultProtocol + "//" + url;
}

// 处理重定向
function handleRedirect(response, body) {
  const location = new URL(response.headers.get('location'));
  const modifiedLocation = `/${encodeURIComponent(location.toString())}`;
  return new Response(body, {
      status: response.status,
      statusText: response.statusText,
      headers: {
          ...response.headers,
          'Location': modifiedLocation
      }
  });
}

// 处理 HTML 内容中的相对路径
// ... 其他代码保持不变 ...

// 修改 handleHtmlContent 函数
// 修改 handleHtmlContent 函数


// 修改 handleHtmlContent 函数中的控制面板部分
async function handleHtmlContent(response, protocol, host, actualUrlStr) {
    const originalText = await response.text();
    let modifiedText = replaceRelativePaths(originalText, protocol, host, new URL(actualUrlStr).origin);

    const controlPanel = `
    <div id="proxyControlPanel" style="
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        background: rgba(255, 255, 255, 0.98);
        padding: 12px;
        z-index: 999999;
        box-shadow: 0 2px 8px rgba(0,0,0,0.15);
        font-family: Arial, sans-serif;
    ">
        <style>
            #proxyControlPanel * {
                box-sizing: border-box;
            }
            .control-wrapper {
                display: flex;
                flex-wrap: wrap;
                gap: 10px;
                margin-bottom: 10px;
            }
            .control-item {
                flex: 1;
                min-width: 200px;
            }
            .preview-url {
                font-family: monospace;
                font-size: 12px;
                padding: 8px;
                background: #f5f5f5;
                border-radius: 4px;
                word-break: break-all;
                margin: 8px 0;
                cursor: pointer;
            }
            .preview-url:hover {
                background: #e9ecef;
            }
            @media (max-width: 768px) {
                .control-item {
                    min-width: 100%;
                }
                #proxyControlPanel {
                    padding: 8px;
                }
            }
            .search-input {
                width: 100%;
                padding: 8px;
                border: 1px solid #ddd;
                border-radius: 4px;
                font-size: 14px;
            }
            .search-button {
                padding: 8px 16px;
                background: #1a73e8;
                color: white;
                border: none;
                border-radius: 4px;
                cursor: pointer;
                font-size: 14px;
                white-space: nowrap;
            }
            .search-button:hover {
                background: #1557b0;
            }
        </style>
        <div class="control-wrapper">
            <div class="control-item">
                <input type="text" id="searchQuery" class="search-input" 
                    value="${getQueryParam(actualUrlStr, 'q')}" 
                    placeholder="搜索关键词">
            </div>
            <div class="control-item">
                <select id="imageSize" class="search-input">
                    <option value="">不限大小</option>
                    <option value="2mp">2MP</option>
                    <option value="4mp">4MP</option>
                    <option value="8mp">8MP</option>
                    <option value="15mp">15MP</option>
                    <option value="20mp">20MP</option>
                    <option value="40mp">40MP</option>
                    <option value="70mp">70MP</option>
                </select>
            </div>
            <div class="control-item">
                <select id="timeRange" class="search-input">
                    <option value="">不限时间</option>
                    <option value="d">24小时内</option>
                    <option value="w">一周内</option>
                    <option value="m">一月内</option>
                    <option value="y">一年内</option>
                </select>
            </div>
            <div class="control-item" style="flex: 0 0 auto;">
                <button onclick="jumpToNewSearch()" class="search-button">新窗口打开</button>
            </div>
        </div>
        <div class="preview-url" id="previewUrl" onclick="jumpToNewSearch()"></div>
    </div>

    <script>
    function generateSearchUrl() {
        const query = encodeURIComponent(document.getElementById('searchQuery').value);
        const imageSize = document.getElementById('imageSize').value;
        const timeRange = document.getElementById('timeRange').value;

        let url = '${protocol}//${host}/https://google.com.hk/search?q=' + query + 
            '&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg';

        if (timeRange) {
            url += '&tbs=qdr:' + timeRange;
        }
        if (imageSize) {
            url += (timeRange ? ',' : '&tbs=') + '&imgsz=' + imageSize;
        }

        return url;
    }

    function updatePreviewUrl() {
        const url = generateSearchUrl();
        document.getElementById('previewUrl').innerHTML = 
            '<div style="color: #666; margin-bottom: 4px;">点击链接直接跳转：</div>' + url;
    }

    function jumpToNewSearch() {
        window.open(generateSearchUrl(), '_blank');
    }

    // 设置当前值并添加事件监听
    window.addEventListener('load', function() {
        const currentUrl = new URL(window.location.href);
        const tbs = new URLSearchParams(currentUrl.search).get('tbs') || '';
        
        if (tbs.includes('qdr:')) {
            const timeValue = tbs.match(/qdr:([dwmy])/)?.[1];
            if (timeValue) document.getElementById('timeRange').value = timeValue;
        }
        
        if (tbs.includes('imgsz=')) {
            const sizeValue = tbs.match(/imgsz=([^,&]*)/)?.[1];
            if (sizeValue) document.getElementById('imageSize').value = sizeValue;
        }

        // 添加事件监听
        ['searchQuery', 'imageSize', 'timeRange'].forEach(id => {
            document.getElementById(id).addEventListener('input', updatePreviewUrl);
            document.getElementById(id).addEventListener('change', updatePreviewUrl);
        });

        // 添加回车键支持
        document.getElementById('searchQuery').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                jumpToNewSearch();
            }
        });

        // 初始更新预览URL
      //  updatePreviewUrl();
    });
    </script>`;

    // 在 </body> 标签前注入控制面板
    modifiedText = modifiedText.replace('</body>', controlPanel + '</body>');

    return modifiedText;
}
// 辅助函数：从 URL 中获取查询参数
function getQueryParam(urlStr, param) {
    try {
        const url = new URL(urlStr);
        return decodeURIComponent(url.searchParams.get(param) || '');
    } catch (e) {
        return '';
    }
}

// ... 其他代码保持不变 ...

// 替换 HTML 内容中的相对路径
function replaceRelativePaths(text, protocol, host, origin) {
  const regex = new RegExp('((href|src|action)=["\'])/(?!/)', 'g');
  return text.replace(regex, `$1${protocol}//${host}/${origin}/`);
}

// 返回 JSON 格式的响应
function jsonResponse(data, status) {
  return new Response(JSON.stringify(data), {
      status: status,
      headers: {
          'Content-Type': 'application/json; charset=utf-8'
      }
  });
}

// 过滤请求头
function filterHeaders(headers, filterFunc) {
  return new Headers([...headers].filter(([name]) => filterFunc(name)));
}

// 设置禁用缓存的头部
function setNoCacheHeaders(headers) {
  headers.set('Cache-Control', 'no-store');
}

// 设置 CORS 头部
function setCorsHeaders(headers) {
  headers.set('Access-Control-Allow-Origin', '*');
  headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  headers.set('Access-Control-Allow-Headers', '*');
}

// 返回根目录的 HTML
function getRootHtml() {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Google图片搜索代理</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
    <style>
        body, html {
            height: 100%;
            margin: 0;
            font-family: Arial, sans-serif;
        }
        .page-container {
            min-height: 100vh;
            background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
            padding: 20px;
        }
        .search-panel {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
            max-width: 1000px;
            margin: 20px auto;
        }
        .panel-title {
            text-align: center;
            margin-bottom: 30px;
            color: #333;
            font-size: 24px;
        }
        .control-group {
            display: flex;
            flex-wrap: wrap;
            gap: 15px;
            margin-bottom: 20px;
        }
        .control-item {
            flex: 1;
            min-width: 250px;
        }
        .input-field {
            margin-bottom: 15px;
        }
        .input-field label {
            display: block;
            margin-bottom: 5px;
            color: #666;
        }
        .input-field input,
        .input-field select {
            width: 100%;
            padding: 8px 12px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-size: 14px;
            box-sizing: border-box;
        }
        .input-field input:focus,
        .input-field select:focus {
            border-color: #1a73e8;
            outline: none;
        }
        .url-display {
            background: #f8f9fa;
            padding: 15px;
            border-radius: 6px;
            margin: 15px 0;
            font-family: monospace;
            font-size: 13px;
            word-break: break-all;
            border: 1px solid #e9ecef;
            cursor: pointer;
            transition: background 0.3s;
        }
        .url-display:hover {
            background: #e9ecef;
        }
        .button-group {
            display: flex;
            gap: 10px;
            margin-top: 20px;
        }
        .btn {
            flex: 1;
            padding: 12px 20px;
            border: none;
            border-radius: 4px;
            font-size: 14px;
            cursor: pointer;
            transition: background-color 0.3s;
            color: white;
            text-align: center;
            text-decoration: none;
        }
        .btn-primary {
            background: #1a73e8;
        }
        .btn-primary:hover {
            background: #1557b0;
        }
        .btn-secondary {
            background: #5f6368;
        }
        .btn-secondary:hover {
            background: #4a4d51;
        }
        @media (max-width: 768px) {
            .page-container {
                padding: 10px;
            }
            .control-item {
                min-width: 100%;
            }
            .button-group {
                flex-direction: column;
            }
            .btn {
                width: 100%;
            }
        }
    </style>
</head>
<body>
    <div class="page-container">
        <div class="search-panel">
            <h1 class="panel-title">Google 图片搜索代理</h1>
            <div class="control-group">
                <div class="control-item">
                    <div class="input-field">
                        <label for="searchQuery">搜索关键词</label>
                        <input type="text" id="searchQuery" value="TV アニメ -eeo.today">
                    </div>
                </div>
                <div class="control-item">
                    <div class="input-field">
                        <label for="imageSize">图片大小</label>
                        <select id="imageSize">
                            <option value="">不限大小</option>
                            <option value="2mp">2MP</option>
                            <option value="4mp">4MP</option>
                            <option value="8mp" selected>8MP</option>
                            <option value="15mp">15MP</option>
                            <option value="20mp">20MP</option>
                            <option value="40mp">40MP</option>
                            <option value="70mp">70MP</option>
                        </select>
                    </div>
                </div>
                <div class="control-item">
                    <div class="input-field">
                        <label for="timeRange">时间范围</label>
                        <select id="timeRange">
                            <option value="">不限时间</option>
                            <option value="d">24小时内</option>
                            <option value="w" selected>一周内</option>
                            <option value="m">一月内</option>
                            <option value="y">一年内</option>
                        </select>
                    </div>
                </div>
            </div>
            
            <div class="url-display" id="previewUrl" onclick="jumpToUrl()"></div>
            
            <div class="button-group">
                <button onclick="jumpToUrl()" class="btn btn-primary">
                    在新窗口打开
                </button>
                <button onclick="loadInCurrentPage()" class="btn btn-secondary">
                    在当前页面加载
                </button>
            </div>
        </div>
    </div>

    <script>
        function generateSearchUrl() {
            const query = encodeURIComponent(document.getElementById('searchQuery').value);
            const imageSize = document.getElementById('imageSize').value;
            const timeRange = document.getElementById('timeRange').value;

            let baseUrl = 'https://google.com.hk/search?q=' + query + 
                '&tbm=isch&start=0&sa=N&lite=0&source=lnms&tbm=isch&sa=X&ei=XosDVaCXD8TasATItgE&ved=0CAcQ_AUoAg';

            if (timeRange) {
                baseUrl += '&tbs=qdr:' + timeRange;
            }
            if (imageSize) {
                baseUrl += (timeRange ? '' : '&tbs=') + '&imgsz=' + imageSize;
            }

            return window.location.origin + '/' + encodeURIComponent(baseUrl);
        }

        function updatePreviewUrl() {
            const url = generateSearchUrl();
            const previewElement = document.getElementById('previewUrl');
            previewElement.innerHTML = '<div style="color: #666; margin-bottom: 8px;">点击下方链接跳转：</div>' + url;
        }

        function jumpToUrl() {
            window.open(generateSearchUrl(), '_blank');
        }

        function loadInCurrentPage() {
            window.location.href = generateSearchUrl();
        }

        // 为所有输入元素添加事件监听
        ['searchQuery', 'imageSize', 'timeRange'].forEach(id => {
            const element = document.getElementById(id);
            element.addEventListener('input', updatePreviewUrl);
            element.addEventListener('change', updatePreviewUrl);
        });

        // 添加回车键支持
        document.getElementById('searchQuery').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                jumpToUrl();
            }
        });

        // 初始化显示URL
        updatePreviewUrl();
    </script>
</body>
</html>`;
}
```

https://github.com/ProgCZ/code-cloud-a/blob/master/2020/05/cf-workers-mirrors/index.jshttps://github.com/ymyuuu/Cloudflare-Workers-Proxy
