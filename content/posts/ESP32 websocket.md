# ESP32 websocket



## 启动webserver

创建http服务监听端口

初始化一些全局变量

```c
start_webserver(void){

    httpd_handle_t server = NULL;
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    httpd_start(&server, &config);

    static const httpd_uri_t ws = {
        .uri = "/ws",
        .method = HTTP_GET,
        .handler = handler,
        .user_ctx = NULL,
        .is_websocket = true};

    httpd_register_uri_handler(server, &ws);
    //httpd_register_uri_handler 注册 URL
    
    // 初始化广播所需的全局状态
    //把这个server给全局变量
    s_server_handle = server;
    if (!s_ws_lock)
    {
        //!s_ws_lock判断锁有没有被创建
        s_ws_lock = xSemaphoreCreateMutex();
    }
    for (int i = 0; i < WS_MAX_CLIENTS; ++i)
    {
        s_client_fds[i] = -1;
    }
    ESP_LOGI(TAG, "WebSocket server ready");
    return server;
}
```

## handler函数

然后我访问的时候就会进入到handler函数

​	fd是底层 TCP 连接的整数句柄。类型是 int，>0 表示有效，断开后会被系统关闭，值可能以后复用。数字本身没业务含义，只是底层 socket 句柄（文件描述符）。句柄(Handle)：一种间接引用资源的整数或指针，自己没有实际数据，只用来让系统在内部找到对应资源。在 ESP-IDF / lwIP 中，TCP 连接创建后分配一个文件描述符 fd（>0）。你保存 fd，就能用 httpd_ws_send_frame_async(server_handle, fd, …) 等函数定位并向该连接发送数据。

```c
static esp_err_t handler(httpd_req_t *req)
{
    //第一次访问时的运行
    if (req->method == HTTP_GET)
    {
        //fd是底层 TCP 连接的整数句柄
        int fd = httpd_req_to_sockfd(req);
        ws_client_add(fd); // 记录客户端
        ESP_LOGI(TAG, "Handshake done, fd=%d", fd);
        ws_broadcast("client %d joined", fd);
        return ESP_OK;
    }
    httpd_ws_frame_t ws_pkt;
    uint8_t *buf = NULL;
    memset(&ws_pkt, 0, sizeof(httpd_ws_frame_t));
    ws_pkt.type = HTTPD_WS_TYPE_TEXT;
    /* Set max_len = 0 to get the frame len */
    esp_err_t ret = httpd_ws_recv_frame(req, &ws_pkt, 0);
    if (ret != ESP_OK)
    {
        ws_client_remove(httpd_req_to_sockfd(req));
        ESP_LOGE(TAG, "get frame len failed %d", ret);
        return ret;
    }
    ESP_LOGI(TAG, "frame len is %d", ws_pkt.len);
    if (ws_pkt.len)
    {
        /* ws_pkt.len + 1 is for NULL termination as we are expecting a string */
        buf = calloc(1, ws_pkt.len + 1);
        if (buf == NULL)
        {
            ESP_LOGE(TAG, "Failed to calloc memory for buf");
            return ESP_ERR_NO_MEM;
        }
        ws_pkt.payload = buf;
        /* Set max_len = ws_pkt.len to get the frame payload */
        ret = httpd_ws_recv_frame(req, &ws_pkt, ws_pkt.len);
        if (ret != ESP_OK)
        {
            ESP_LOGE(TAG, "httpd_ws_recv_frame failed with %d", ret);
            free(buf);
            return ret;
        }
        ESP_LOGI(TAG, "Got packet with message: %s", ws_pkt.payload);
    }
    ESP_LOGI(TAG, "Packet type: %d", ws_pkt.type);
    if (ws_pkt.type == HTTPD_WS_TYPE_TEXT &&
        strcmp((char *)ws_pkt.payload, "Trigger async") == 0)
    {
        free(buf);
        return trigger_async_send(req->handle, req);
    }

    ret = httpd_ws_send_frame(req, &ws_pkt);
    if (ret != ESP_OK)
    {
        ESP_LOGE(TAG, "httpd_ws_send_frame failed with %d", ret);
    }
    free(buf);
    return ret;
}
```

## 添加客户端

```c
static void ws_client_add(int fd)
{
    if (!s_ws_lock)
        return;
    xSemaphoreTake(s_ws_lock, portMAX_DELAY);
    for (int i = 0; i < WS_MAX_CLIENTS; ++i)
    {
        if (s_client_fds[i] == fd)
        { // 已存在
            xSemaphoreGive(s_ws_lock);
            return;
        }
        if (s_client_fds[i] <= 0)
        {
            s_client_fds[i] = fd;
            break;
        }
    }
    xSemaphoreGive(s_ws_lock);
}
```



calloc 与 malloc 的区别:

- 初始化
  - malloc(size): 只分配内存,内容未初始化(是脏数据)。
  - calloc(n, size): 分配 n*size 字节,并将全部字节置为 0。
- 形参
  - malloc 只有字节数一个参数。
  - calloc 有元素个数和单个元素大小两个参数,更适合“数组/结构体数组”。
- 溢出检查
  - malloc 需要你自己计算 n*size,可能溢出。
  - calloc 一般会在内部做乘法溢出检查(实现相关),溢出时返回 NULL,更安全。
- 性能
  - calloc 需要清零,在 ESP-IDF 这类无页懒分配的系统上确实要实际写零,会比 malloc 略慢; 但省去了手动 memset。





目前修改增加了两种频道

一种是之前的video频道，外加一个传文字内容的的吧