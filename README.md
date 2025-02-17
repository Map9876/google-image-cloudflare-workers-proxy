# google-image-cloudflare-workers-proxy
使用cloudflare workers谷歌搜图

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
