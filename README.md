# n8n 自動化工具 docker 安裝以及簡介教學 🤖

* [Youtube Tutorial - n8n 自動化工具 docker 安裝以及簡介教學](https://youtu.be/uCj14FJu6EI)

n8n 是個開源的視覺化工作流程自動化工具（可以自架設）,

可以像組積木一樣，透過拖拉方式建立自定義的自動化流程，同時也支援透過程式碼撰寫 Python 和 JavaScript 來擴充功能。

## 使用 docker 安裝 n8n

這邊假設使用者已經安裝好 docker,

[n8n - docker-compose](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/)

先建立 `.env` 檔案

```cmd
mkdir n8n-compose
cd n8n-compose
```

```env
DOMAIN_NAME=example.com

# The subdomain to serve from
SUBDOMAIN=n8n

# The above example serve n8n at: https://n8n.example.com

# Optional timezone to set which gets used by Cron and other scheduling nodes
# New York is the default value if not set
GENERIC_TIMEZONE=Asia/Taipei

# The email address to use for the TLS/SSL certificate creation
SSL_EMAIL=user@example.com
```

建立一個資料夾保存 n8n data

```cmd
mkdir local-files
```

`docker-compose.yml` 如下

```yml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: n8nio/n8n:stable
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

volumes:
  n8n_data:
  traefik_data:
```

如果你在你的雲空間, 直接設定好你的 DNS, 就可以一次完成了.

### 關於 Traefik 🚦

traefik 服務是一個現代化的反向代理和負載平衡器，類似於 Nginx。

它特別擅長與 Docker 整合，可以自動偵測新的容器服務並為其設定路由和 SSL 憑證。

如果你已經在主機上使用了 Nginx 或其他反向代理，可以考慮調整設定，移除 traefik 服務，

並手動設定你的反向代理將流量轉發到 n8n 容器的 5678 埠。

最後執行 `docker compose up -d` 就完成了.

瀏覽你的 [http://127.0.0.1:5678](http://127.0.0.1:5678) 可以開始把玩 n8n.

但要注意 🔒

`n8n is only accessible using secure HTTPS, not over plain HTTP.`

n8n 官方建議且許多功能（特別是 Webhook）需要在安全的 HTTPS 環境下運作。

如果你在本機使用 HTTP 訪問(本機)，可能會在測試某些節點或功能時遇到問題。

因此，建議最終將 n8n 部署到具有有效 HTTPS 憑證的環境中。

也就是如果你不是 HTTPS, 測試起來可能會遇到一些問題, 所以我會建議直接佈署到雲空間並且處理好 HTTPS.

前面說到 traefik, 我自己的雲端主機是使用 nginx 了, 所以我有再調整移除 traefik, 改用 Nginx 也是沒問題的.

## 範例介紹

一般來說，大型語言模型本身不具備即時上網搜尋或執行外部動作的能力。

透過 n8n 中的 AI Agent 功能, 可以賦予 AI 模型額外能力,

使用工具 (Tools): 例如讓 AI 能夠執行網路搜尋、讀取檔案、呼叫 API 等。

規劃與執行: AI 可以根據目標，自行規劃需要使用哪些工具、按什麼順序執行，來完成更複雜的任務。

![alt tag](https://i.imgur.com/Js6MeDx.png)

本身 AI 不具有上網的能力, 但是當透過 AI Agent 之後, 整個功能就會變得很強大

![alt tag](https://i.imgur.com/vogYlTG.png)

範例 [AI_Agent_in_n8n.json](AI_Agent_in_n8n.json) 直接複製貼上即可.

#### 測試建議

在測試 AI Agent 功能時，建議優先使用 OpenAI 的模型，因為其對於 Function Calling / Tool Using 的支援通常較為成熟，

使用其他模型可能會遇到預期外的錯誤或限制, 因為我發現使用其他的 AI 會有其他的錯誤.

也就是說, AI Agent 比較像 Function Calling, 和之後要介紹的 MCP 又不太一樣.

## MCP Server

如果要使用 MCP 中的 MCP Server Trigger (n8n Version `1.88.0` 才有).

通常會使用反向代理( 我使用 Nginx ),

Nginx 記得一定要設定 `proxy_buffering off;` :exclamation: :exclamation: (範例如下)

不然你會發現你的 MCP 一直連不上.

```conf
 location / {
    proxy_pass http://n8n:5678;
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
  }
```

這邊說明一下什麼是 `proxy_buffering` 的作用 🤔 這個指令控制 Nginx 是否要緩衝 (buffer) 從後端伺服器.

`on` (預設值): 當 `proxy_buffering` 是 `on` 時，Nginx 會先嘗試從後端讀取 完整 的回應，

將其存儲在記憶體或暫存檔中，然後再 一次性 地將這個完整的回應發送給客戶端。

這樣做的好處是，如果客戶端網路慢，不會長時間佔用後端 n8n 的連線；

壞處是客戶端需要等待 Nginx 接收完所有數據才能開始收到內容，且會消耗 Nginx 伺服器的記憶體/磁碟空間.

`off` 當 `proxy_buffering` 是 `off` 時，

Nginx 會 即時地 將從後端（n8n）收到的數據塊 (chunk) 直接轉發給客戶端，

不做完整的緩衝。收到多少，就轉發多少。

接著說明 n8n 的 MCP Server(與 `proxy_buffering off;` 的關聯)

n8n 有一些功能需要 長時間連接 並且 即時地 將數據從後端推送到前端（瀏覽器）。

SSE Trigger 節點： 如果你的工作流程使用 SSE Trigger (**SSE - Server-Sent Events**) 來接收事件。

為什麼需要 `proxy_buffering off;`

對於像 SSE 這樣的技術，數據是 持續不斷地、一小塊一小塊地 從伺服器發送到客戶端的。

如果 Nginx 使用預設的 `proxy_buffering on;`

它會一直等待 n8n 發送完 所有 的 SSE 數據（對於一個持續的事件流來說，可能永遠不會結束），

然後才轉發給瀏覽器。這就導致瀏覽器完全收不到即時的更新，違背了 SSE 的目的。

只有設置為 `proxy_buffering off;`

Nginx 才能在收到 n8n 發來的一小塊 SSE 數據時，立刻轉發給瀏覽器，從而實現了 n8n 編輯器中的即時反饋效果。

其他注意事項可參考 [Limitations](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#limitations)

## 社群節點 🧩

除了官方提供的節點之外, 社群有提供了非常多的第三方節點可以使用

[community-node-package](https://www.npmjs.com/search?q=keywords%3An8n-community-node-package)

可以自行選擇需要的安裝.

![alt tag](https://i.imgur.com/L4hZgFT.png)

之後如果我有使用我再和大家分享.

## Donation

文章都是我自己研究內化後原創，如果有幫助到您，也想鼓勵我的話，歡迎請我喝一杯咖啡  :laughing:

綠界科技ECPAY ( 不需註冊會員 )

![alt tag](https://payment.ecpay.com.tw/Upload/QRCode/201906/QRCode_672351b8-5ab3-42dd-9c7c-c24c3e6a10a0.png)

[贊助者付款](http://bit.ly/2F7Jrha)

歐付寶 ( 需註冊會員 )

![alt tag](https://i.imgur.com/LRct9xa.png)

[贊助者付款](https://payment.opay.tw/Broadcaster/Donate/9E47FDEF85ABE383A0F5FC6A218606F8)

## 贊助名單

[贊助名單](https://github.com/twtrubiks/Thank-you-for-donate)

## License

MIT license
