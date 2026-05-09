# 连接断开与重连机制详解

## 🎯 核心问题

**当`MSG_PEEK`返回0（连接关闭）时，服务器会自动重新连接客户端吗？**

### ❌ 答案：不会！

服务器**永远不会主动重连客户端**，原因如下：

---

## 📊 架构分析

### rtsproxy的角色定位

```
┌─────────────┐                    ┌──────────────┐
│   客户端     │                    │  rtsproxy    │
│  (VLC/播放器) │                    │   (服务器)    │
└──────┬──────┘                    └──────┬───────┘
       │                                  │
       │  1. TCP SYN (主动发起连接)        │
       │ ───────────────────────────────> │
       │                                  │ 2. accept() 接受连接
       │                                  │ 3. 创建新会话
       │                                  │
       │  4. RTSP请求 (OPTIONS/DESCRIBE)  │
       │ ───────────────────────────────> │
       │                                  │ 5. 转发到上游服务器
       │                                  │
       │  6. RTP流媒体数据                │
       │ <─────────────────────────────── │
       │                                  │
       │  7. 连接断开 (FIN/RST)           │
       │ ───────────────────────────────> │
       │                                  │ 8. 清理资源
       │                                  │ 9. ❌ 不会主动重连
       │                                  │
       │  10. 如需重连，客户端必须          │
       │  重新发起TCP连接                  │
       │ ───────────────────────────────> │
```

**关键点**：
- ✅ rtsproxy是**被动服务器**，只接受incoming connections
- ❌ rtsproxy**不会主动**去连接客户端
- ✅ 重连的责任完全在**客户端侧**

---

## 🔍 详细流程分析

### 场景1：协议探测阶段连接断开

```cpp
// main.cpp L305-L309
else if (n == 0) {
    // Connection closed by peer during protocol detection
    Logger::debug("[SERVER] Client disconnected during protocol detection");
    loop.remove(client_fd);  // ⭐ 从epoll移除
    close(client_fd);         // ⭐ 关闭fd
    // ❌ 没有重连逻辑
}
```

**发生了什么**：
1. 客户端TCP连接建立
2. 但在发送任何数据之前就断开了
3. `recv(MSG_PEEK)`返回0
4. 服务器清理资源
5. **会话结束，不会重连**

---

### 场景2：RTSP会话中上游连接失败

```cpp
// rtsp_client.cpp L107-L110
if (rtsp_fd_ < 0) {
    Logger::error("[RTSP] Connect to upstream failed.");
    if (on_closed_callback_) on_closed_callback_();  // ⭐ 通知上层
    return;
}
```

**回调链**：
```
RTSPClient检测到上游连接失败
    ↓
调用 on_closed_callback_()
    ↓
触发 rtsp_handle.cpp L92-L98 的lambda
    ↓
loop->remove_client_from_map(client_fd)  // 清理客户端映射
    ↓
RTSPMitmClient析构函数被调用
    ↓
所有资源被释放（fd、socket、内存等）
    ↓
❌ 会话结束，不会重连
```

**上游连接失败的原因**：
- 上游服务器宕机
- 网络不通
- 防火墙阻止
- URL错误

**结果**：整个会话终止，客户端需要重新发起请求

---

### 场景3： streaming过程中上游断开

```cpp
// rtsp_mitm_client.cpp L1164-L1170
void RTSPMitmClient::on_upstream_readable()
{
    char buf[8192];
    ssize_t n = recv(upstream_fd_, buf, sizeof(buf) - 1, 0);
    if (n <= 0) {
        if (n == 0 || (errno != EAGAIN && errno != EWOULDBLOCK)) {
            Logger::debug("[MITM] Upstream closed connection");
            close_all();  // ⭐ 关闭所有连接
        }
        return;
    }
    ...
}
```

**close_all()的行为**：
```cpp
void RTSPMitmClient::close_all()
{
    // 关闭下游（客户端）连接
    if (downstream_fd_ >= 0) {
        loop_->remove(downstream_fd_);
        ::close(downstream_fd_);
    }
    
    // 关闭上游（服务器）连接
    if (upstream_fd_ >= 0) {
        loop_->remove(upstream_fd_);
        ::close(upstream_fd_);
    }
    
    // 关闭RTP/RTCP socket
    // 关闭timer fd
    // ...
    
    // 触发on_closed_回调
    if (on_closed_) {
        on_closed_();  // ⭐ 清理client_ptr_map
    }
}
```

**结果**：
- ✅ 所有相关资源被清理
- ✅ 客户端会收到连接断开的通知
- ❌ **服务器不会尝试重连上游或客户端**

---

## 💡 为什么设计成这样？

### 原因1：无状态设计

```
rtsproxy是一个无状态的代理服务器：
- 每个连接都是独立的
- 不维护客户端列表
- 不记录客户端历史
- 断开=结束，不会重试

优点：
✅ 简单可靠
✅ 资源占用低
✅ 不会有"僵尸连接"
✅ 易于调试和维护
```

### 原因2：客户端更了解需求

```
只有客户端知道：
- 是否需要重连
- 何时重连
- 重连间隔多久
- 重连多少次后放弃

服务器不应该替客户端做决定
```

### 原因3：避免资源泄漏

```
如果服务器自动重连：
❌ 需要维护重连队列
❌ 需要管理重连超时
❌ 可能导致大量无效连接
❌ 增加复杂度
❌ 可能耗尽文件描述符

当前设计：
✅ 断开即清理
✅ 资源立即释放
✅ 简单高效
```

---

## 🔄 正确的重连模式

### 客户端侧的重连逻辑（推荐）

```python
# Python示例：客户端重连逻辑
import time
import socket

def connect_with_retry(server_ip, server_port, max_retries=10, retry_delay=5):
    """带重试的连接函数"""
    for attempt in range(1, max_retries + 1):
        try:
            print(f"Attempt {attempt}/{max_retries}: Connecting...")
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(10)
            sock.connect((server_ip, server_port))
            print("Connected successfully!")
            return sock
        except Exception as e:
            print(f"Connection failed: {e}")
            if attempt < max_retries:
                print(f"Retrying in {retry_delay} seconds...")
                time.sleep(retry_delay)
            else:
                print("Max retries reached. Giving up.")
                raise

# 使用示例
try:
    sock = connect_with_retry("192.168.1.1", 8554)
    # 发送RTSP请求
    sock.send(b"OPTIONS rtsp://... RTSP/1.0\r\n...")
    # 接收响应
    response = sock.recv(4096)
except Exception as e:
    print(f"Failed to connect: {e}")
```

### VLC播放器的自动重连

```
VLC内置了自动重连机制：

1. 检测到连接断开
2. 等待几秒（可配置）
3. 重新发起TCP连接
4. 重新发送RTSP请求
5. 恢复播放

配置方法：
工具 → 偏好设置 → 输入/编解码器 → 网络缓存
调整"网络缓存(ms)"和重试参数
```

---

## 📝 特殊情况：上游服务器的重连

虽然rtsproxy不会重连**客户端**，但在某些情况下会重试**上游连接**：

### 情况1：SETUP失败时的协议降级

```cpp
// rtsp_client.cpp L288-L294
if (status == 461 && current_request_.method == RtspMethod::SETUP && !setup_retry_with_tcp_) {
    Logger::warn("[RTSP] Upstream rejected UDP SETUP (461). Retrying with TCP Interleaved...");
    setup_retry_with_tcp_ = true;
    send_rtsp_setup();  // ⭐ 重新发送SETUP，但改用TCP模式
    continue;
}
```

**这不是真正的重连**，而是：
- ✅ 同一个TCP连接
- ✅ 同一个会话
- ✅ 只是改变了传输模式（UDP → TCP）

### 情况2：心跳保活

```cpp
// rtsp_mitm_client.cpp L877-L884
void RTSPMitmClient::handle_timer(uint32_t events)
{
    uint64_t exp;
    read(timer_fd_, &exp, sizeof(exp));
    // Send a GET_PARAMETER to upstream as keepalive.
    std::string ka = "GET_PARAMETER " + ctx_.rtsp_url + " RTSP/1.0\r\n"
                     "CSeq: 99\r\n"
                     "Session: " + ctx_.session_id + "\r\n"
                     "\r\n";
    to_upstream_q_.push_back(ka);  // ⭐ 发送心跳保持连接
    ...
}
```

**这是保活机制**，不是重连：
- ✅ 防止上游服务器因超时而断开
- ✅ 维持现有连接
- ❌ 不创建新连接

---

## 🎯 总结对比

| 场景 | 谁会重连 | 如何重连 | 说明 |
|------|---------|---------|------|
| 客户端断开 | **客户端** | 重新发起TCP连接 | 服务器被动接受 |
| 上游服务器断开 | **客户端** | 重新请求URL | 整个会话重新开始 |
| SETUP失败 | **rtsproxy** | 重试SETUP（改TCP） | 同一连接内重试 |
| 网络抖动 | **客户端** | 自动重连 | 取决于客户端实现 |
| 心跳超时 | **rtsproxy** | 发送GET_PARAMETER | 保活，非重连 |

---

## ✅ 最佳实践建议

### 对于客户端开发者

```javascript
// JavaScript示例：健壮的重连逻辑
class RTSPClient {
    constructor(url) {
        this.url = url;
        this.maxRetries = 5;
        this.retryDelay = 3000; // 3秒
        this.retryCount = 0;
    }
    
    async connect() {
        while (this.retryCount < this.maxRetries) {
            try {
                await this.establishConnection();
                this.retryCount = 0; // 成功后重置
                return;
            } catch (error) {
                this.retryCount++;
                console.log(`Retry ${this.retryCount}/${this.maxRetries}`);
                
                if (this.retryCount >= this.maxRetries) {
                    throw new Error('Max retries exceeded');
                }
                
                // 指数退避：3s, 6s, 12s, 24s...
                const delay = this.retryDelay * Math.pow(2, this.retryCount - 1);
                await this.sleep(delay);
            }
        }
    }
    
    onConnectionLost() {
        console.log('Connection lost, attempting reconnect...');
        this.retryCount = 0;
        this.connect();
    }
}
```

### 对于rtsproxy运维

```bash
# 监控连接状态
watch -n 1 'netstat -tlnp | grep 8554'

# 查看日志中的断开信息
logread | grep "disconnected"

# 检查是否有异常的大量断开
logread | grep "disconnected" | wc -l

# 如果发现频繁断开，检查：
# 1. 网络稳定性
# 2. 客户端配置
# 3. 上游服务器状态
# 4. 防火墙规则
```

---

## 🔧 如果需要服务器端重连（高级）

如果你确实需要rtsproxy具备上游重连能力，可以考虑以下方案：

### 方案1：应用层重连（推荐）

```cpp
// 伪代码：在RTSPMitmClient中添加重连逻辑
class RTSPMitmClient {
private:
    int reconnect_attempts_ = 0;
    static const int MAX_RECONNECT = 3;
    
    void on_upstream_disconnected() {
        if (reconnect_attempts_ < MAX_RECONNECT) {
            reconnect_attempts_++;
            Logger::info("Attempting reconnection " + 
                        std::to_string(reconnect_attempts_));
            
            // 延迟重连（指数退避）
            int delay = 1000 * (1 << (reconnect_attempts_ - 1));
            usleep(delay * 1000);
            
            // 重新连接上游
            connect_upstream();
        } else {
            Logger::error("Max reconnection attempts reached");
            close_all();
        }
    }
};
```

### 方案2：外部看门狗

```bash
#!/bin/bash
# watchdog.sh - 外部监控脚本

PROXY_URL="rtsp://192.168.1.1:8554/stream"
MAX_RETRIES=5

for i in $(seq 1 $MAX_RETRIES); do
    if timeout 10 ffprobe "$PROXY_URL" > /dev/null 2>&1; then
        echo "Stream is healthy"
        exit 0
    else
        echo "Stream check failed, attempt $i/$MAX_RETRIES"
        sleep 5
    fi
done

echo "Stream is down after $MAX_RETRIES attempts"
# 可以触发告警或重启服务
```

---

## 🎉 结论

**回到你的问题**：

> 如果MSG_PEEK返回0（连接关闭）还会自动重新连接吗？

**答案**：
- ❌ **不会**自动重连
- ✅ 这是**正确的设计**
- ✅ 重连责任在**客户端**
- ✅ 服务器只做**资源清理**

这种设计保证了：
- ✅ 简单可靠
- ✅ 资源高效
- ✅ 职责清晰
- ✅ 易于维护

如果需要重连功能，应该在**客户端**实现，而不是在服务器端。
