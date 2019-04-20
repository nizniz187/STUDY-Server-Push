# STUDY-ServerPush
Study note for server push.

## A. Comet Server Push
1. 在 Client Request 後，與 Server 保持持久連結，以方便 Server 在連結期間主動傳遞資料。
2. **Long-Polling**：延後 Server Response 的時間，利用這段延遲來減少 Request 數量。適用於資料更新不頻繁的場合。實作技術：AJAX / Script tag。
3. **HTTP Streaming**：連結後使用串流模式傳遞資料，Client 可在連結期間不斷接收 Server 資料，直到 Server 傳遞結束。實作技術：iframe / AJAX / SSE。可能因 proxy 或防火牆的資料緩存機制，造成資料回應上的延遲。
4. 伺服器端可實作非同步回應，來處理資源佔據的效能問題。

> **Reference**
> - **[Comet | Wikipedia](https://en.wikipedia.org/wiki/Comet_(programming))**
> - [Comet: Low Latency Data for the Browser](http://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)
> - [非同步 Long Polling](https://openhome.cc/Gossip/ServletJSP/LongPolling.html)

## A-1. HTTP Streaming 實作
1. 傳統的 Request / Response 機制為有限長度的傳輸，並在傳輸完成後才對整塊完整資料進行處理。Streaming 的概念是不限定長度，將資料分塊傳輸並即時處理，使得總傳輸量可超過接收方的記憶體容量限制。
2. 分塊傳輸的概念適用於 Request 與 Response。Request streaming 對上傳檔案特別有用，但實務上支援此功能的 server 不多。
3. HTTP/1.1：「**Transfer-Encoding: chunked**」；HTTP/2：不須額外設定（IE11 部分支援）
4. 若資料非連續傳輸，須注意 **timeout** 問題（後端可能須用特定方式 output stream，前端如用 AJAX 則可設定 timeout=0）
5. 實作面最大的問題在於 **buffering**；一次請求回應所流經的所有 Client / Server 都有各自的 buffering 機制與限制。若要 streaming 正確運作，一般不是把預設的 buffering 機制關閉，就是將 buffer size 設定成與 chunk size 相同。
6. 瀏覽器在開始解析 Response 資料以前，有個資料接收的 buffer 限制。每個瀏覽器的 buffer 限制不同，通常為 1KB。
7. 資料分塊愈多，切割與解析的成本就愈高。另外若接收方無法立刻處理資料，則資料緩衝的成本與複雜度也會提高。

> **Reference**
> - **[HTTP Streaming (or Chunked vs Store & Forward)](https://gist.github.com/CMCDragonkai/6bfade6431e9ffb7fe88)**
> - [Browser Buffer Limit | StackOverflow](https://stackoverflow.com/questions/16909227/using-transfer-encoding-chunked-how-much-data-must-be-sent-before-browsers-s/16909228#16909228)
> - [分塊傳輸編碼 | Wikipedia](https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)
> - [Browser Support for HTTP/2 | Can I use...](https://caniuse.com/#search=http2)
> - [XMLHttpRequest.timeout | MDN](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/timeout)
> - [使用Tomcat实现基于iframe streaming的Comet聊天室](https://blog.csdn.net/xiao__gui/article/details/38487117)

### A-1-1. JSON Streaming (JS)
1. 此技術主要也是應用於大型資料傳輸上
2. 如何運用 AJAX 在後端以串流回傳資料的同時，同步進行處理
3. Oboe.js & Highland.js 可幫助處理 JSON 串流資料，兩者都適用於 Node Server 和 Client Browser
4. NodeJS 對 streaming 的支援性相當好。

> **Reference**
> - [JSON Streaming | Wikipedia](https://en.wikipedia.org/wiki/JSON_streaming)
> - **[[譯] 使用 stream 的方式處理 JSON 傳輸](https://andyyou.github.io/2017/05/22/better-json-through-stream/)**
> - [Oboe.js](http://oboejs.com/)
> - [Hignland.js](https://highlandjs.org/)

### A-1-2. HTTP Streaming Media
影音類多媒體串流，使得客戶端得以：
1. 一邊下載一邊播放，
2. 跳至指定位置下載並播放。

> **Reference**
> - [串流媒體 (Stream Media)](https://www.moneydj.com/kmdj/wiki/wikiviewer.aspx?keyid=9dbe3247-8dbf-4b7e-a5b8-9ead881f7c1d)
> - [HTTP Streaming: What You Need to Know](https://www.streamingmedia.com/Articles/Editorial/Featured-Articles/HTTP-Streaming-What-You-Need-to-Know-65749.aspx)

### A-1-3. Server-Sent Event
1. 最新實作 HTTP Streaming 的 HTML5 API。類似訂閱機制：瀏覽器透過 SSE 來向 Server 訂閱一個資料來源；一旦有更新時，Server 即會主動通知瀏覽器。
2. 與 WebSocket 相似，但為 Server > Client 單向。具備 WebSocket 所沒有的特性：自動重新連線、事件 ID、傳送任意事件。
3. **IE / Edge 不支援**
4. SSE 定義了一些特別的資料傳輸格式，所有要從伺服器透過 SSE 傳輸的資料都要符合它所定義的格式：「data: ... \n\n」。
5. 安全性：在使用 SSE 接收訊息時，應該要檢查訊息中的 e.origin 是否跟應用程式正確的來源相符合。

> **Reference**
> - **[HTML5 的 Server-Sent Events 串流使用教學](https://blog.gtwang.org/web-development/stream-updates-with-server-sent-events/)**
> - [Browser Support for Server-Sent Events | Can I use...](https://caniuse.com/#feat=eventsource)
> - [非同步 Server-Sent Event](https://openhome.cc/Gossip/ServletJSP/ServerSentEvent.html)

### A-1-4. HTTP/2 Server Push
1. 主要用於 Server 主動推送該網頁所需的其他資源，比較像是 preload 的概念。
2. 需 Server 支援實作。需在 Server 端預先定義好須主動推送哪些資源，這也代表**所有資源須在同一台 Server 上**。
3. 若瀏覽器已存在該資源的緩存，反而浪費資源。因此一般只針對第一次訪問的使用者進行 Server Push。

> **Reference**
> - [HTTP/2 服务器推送（Server Push）教程](http://www.ruanyifeng.com/blog/2018/03/http2_server_push.html)
> - [HTTP2 Server Push的研究](http://www.alloyteam.com/2017/01/http2-server-push-research/)
> - [Node HTTP/2 Server Push 从了解到放弃](https://juejin.im/post/5ae3f17ef265da0ba062e7e3)
> - **[A Comprehensive Guide To HTTP/2 Server Push](https://www.smashingmagazine.com/2017/04/guide-http2-server-push/)**
> - [Browser Support for HTTP/2 | caniuse](https://caniuse.com/#search=http2)

#### A-1-4-1. HTTP Basic
1. OSI Model
2. HTTP 3-way handshake

> **Reference**
> - [OSI 模型 | Wikipedia](https://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B)
> - [解析 HTTP 封包，瞭解 Session ID 和 3-way handshake](http://j796160836.pixnet.net/blog/post/29912977-%5Bhttp%E7%9B%B8%E9%97%9C%5D-%E8%A7%A3%E6%9E%90http%E5%B0%81%E5%8C%85%EF%BC%8C%E7%9E%AD%E8%A7%A3session-id%E5%92%8C3-way-han)

#### A-1-4-2. HTTP/2 Multiplexing
1. HTTP/1.1 可並行請求資源，但 TCP 三方交握成本仍高，請並行數量有上限。HTTP/1.1 keep-alive 能減少交握，但需串行。HTTP/2 Multiplexing 可解決上述問題。
2. Request / Response 能在同一個 TCP 連線中：a) 多個打散並行傳輸、b) 兩者並行傳輸。
3. 過去的網站載入優化處理在此架構下可能變得不再適用（例如：將資源壓成一包來降低 Request 數量。）

> **Reference**
> - [HTTP/2 | Wikipedia](https://zh.wikipedia.org/wiki/HTTP/2)
> - **[Request and response multiplexing | Introduction to HTTP/2](https://developers.google.com/web/fundamentals/performance/http2/#request_and_response_multiplexing)**

## B. WebSocket
1. 全雙工傳輸
2. 網路頻寬與延遲都比 HTTP 協定小很多
3. 各大瀏覽器支援良好（IE10+ 支援）
4. Client 端實作相對簡單；Server 端亦須支援 WebSocket

> **Reference**
> - **[WebSocket 通訊協定簡介：比較 Polling、Long-Polling 與 Streaming 的運作原理](https://blog.gtwang.org/web-development/websocket-protocol/)**
> - [What is WebSocket?](https://www.pubnub.com/learn/glossary/what-is-websocket/)
> - [Browser Support for Web Sockets | Can I use...](https://caniuse.com/#search=websocket)

### B-1. WebSocket v.s. HTTP
1. 無論是傳輸時間、資料大小，或每秒請求數，WebSocket 都有絕對的優勢，而這在傳輸頻繁、Server 同時連線 loading 重的情境下尤其明顯。
2. 建立 WebSocket 連線需耗時 ~190ms，在多數情況下不會造成嚴重的 overhead。
3. HTTP 的優勢：可快取、資料安全一致性高、錯誤處理情境支援性高，適合同步事件處理。

> **Reference**
> - **[HTTP vs Websockets: A performance comparison](https://blog.feathersjs.com/http-vs-websockets-a-performance-comparison-da2533f13a77)**
> - [When to use a HTTP call instead of a WebSocket (or HTTP 2.0)](https://blogs.windows.com/buildingapps/2016/03/14/when-to-use-a-http-call-instead-of-a-websocket-or-http-2-0/#7iv0EEjqjhEygIDx.97)

### B-2. WebSocket 實作

> **Reference**
> - [製作 WebSocket 客戶端應用程式 | MDN](https://developer.mozilla.org/zh-TW/docs/WebSockets/Writing_WebSocket_client_applications)

#### B-2-1. WebSocket - Timeout
1. Server / Proxy 可能會有預設的閒置自動關閉連線設定。
2. Client 端需自行設定排程來實作 timeout 機制 / 斷線自動重連。

> **Reference**
> - [處理 Websocket 超時](http://www.jstips.co/zh_tw/javascript/working-with-websocket-timeout/)
> - [websocket closing connection automatically | StackOverflow](https://stackoverflow.com/a/28932266)

#### B-2-2. WebSocket - Keep Alive
1. WebSocket 使用 PING/PONG 訊息傳遞機制來保持連線：由 Server 傳送 PING 給 Client，Client 再回應 PONG。若 Client 無回應，則 Server 自動關閉連線。
2. WebSocket API 目前不支援由 Client 發起 PING/PONG。雖然瀏覽器可能自行實作發起 PING/PONG，但大部分的 PING/PONG 是由 Server 發起的。
3. PING/PONG 機制的實作方式視瀏覽器和 Server 而有所不同。可自訂方法來自行實作。
4. PING/PONG 的間隔最好小於 30 秒，因為在各個環境中（Chrome / Firefox WebSocket Idle Timeout、TCP Timeout）許多超時的實作設定是 30 秒。20 秒是個不錯的選擇。

> **Reference**
> - [Sending and receiving heartbeat messages](https://django-websocket-redis.readthedocs.io/en/latest/heartbeats.html)
> - [Sending websocket ping/pong frame from browser | StackOverflow](https://stackoverflow.com/a/10586583)
> - **[websocket 的 ping 和 pong 以及 ping 的最佳间隔时间](https://www.crifan.com/websocket_ping_pong_best_interval_time/)**

#### C. WebHook
1. 主要用於 Server - Server 之間的訂閱通知回呼機制
2. 使用 HTTP 協定
3. 無法跨防火牆或在瀏覽器中使用

> **Reference**
> - [Webhook | Wikipedia](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E9%92%A9%E5%AD%90)
> - [測試 webhook 不再煩惱：ngrok](https://blog.techbridge.cc/2018/05/24/ngrok/)

## ★ Wrap-Up
### Server Push | TK
1. 各種實現 Server Push 的方式
2. 各方式的優劣比較
3. 需求分析

> **Link**
> - [Server Push | Google Slide](https://docs.google.com/presentation/d/1dHdkasOrkfDwE9pKDV5q7HE1AqD3FO0f0ad6teqnnto/edit?usp=sharing)
> - Medium

### WebSocket KeepAlive / Timeout | TK
