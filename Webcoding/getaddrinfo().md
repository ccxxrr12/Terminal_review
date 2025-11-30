# Winsock 中 getaddrinfo() 函数详解
`getaddrinfo()` 是 Winsock（Windows 套接字）中用于**域名/服务名到套接字地址转换**的核心函数，替代了老旧的 `gethostbyname()`/`getservbyname()` 等函数，支持 IPv4/IPv6 双栈、线程安全，且符合 RFC 3493 标准，是现代网络编程中解析地址的首选方式。

## 一、核心作用
将**主机名（如 `www.baidu.com`）/IP 字符串（如 `192.168.1.1`）** 和**服务名（如 `80`/`http`）/端口号** 转换为通用的 `sockaddr` 套接字地址结构，无需区分 IPv4/IPv6，直接用于套接字的 `connect()`/`bind()` 等操作。

## 二、函数原型（Winsock 版本）
需包含头文件：`#include <winsock2.h>`、`#include <ws2tcpip.h>`（Windows 特有，Linux 仅需 `<netdb.h>`）。

```c
int WSAAPI getaddrinfo(
  const char*       node,        // 主机名/IP 地址（如"www.baidu.com"、"127.0.0.1"）
  const char*       service,     // 端口号/服务名（如"80"、"http"、"53"）
  const struct addrinfo* hints,  // 筛选条件（指定地址族、套接字类型等）
  struct addrinfo**  res         // 输出：指向结果链表的指针（需手动释放）
);
```

### 参数详解
| 参数       | 说明                                                                 |
|------------|----------------------------------------------------------------------|
| `node`     | 可选：<br>- 主机名（如 `google.com`）、IPv4 字符串（`192.168.0.1`）、IPv6 字符串（`::1`）；<br>- 传 `NULL` 时，结合 `hints` 可用于绑定本机所有地址（如服务器端）。 |
| `service`  | 可选：<br>- 数字端口号（如 `"8080"`）、知名服务名（如 `"ftp"`/`"ssh"`，对应/etc/services）；<br>- 传 `NULL` 则地址结构中端口字段为 0（需手动赋值）。 |
| `hints`    | 可选：<br>- 指向 `addrinfo` 结构体，用于指定筛选条件（如只返回 IPv4 地址、只返回 TCP 类型）；<br>- 传 `NULL` 则默认返回 `AF_UNSPEC`（IPv4/IPv6 均可）、`SOCK_STREAM` 类型。 |
| `res`      | 输出参数：<br>- 函数成功后，指向 `addrinfo` 结构体链表的头节点；<br>- 链表中每个节点对应一个可用的套接字地址（需遍历使用）。 |

## 三、关键结构体：addrinfo
`getaddrinfo()` 的输入（`hints`）和输出（`res`）均基于该结构体，定义如下：
```c
struct addrinfo {
  int              ai_flags;     // 标志（如 AI_PASSIVE 用于服务器绑定）
  int              ai_family;    // 地址族（AF_INET=IPv4、AF_INET6=IPv6、AF_UNSPEC=任意）
  int              ai_socktype;  // 套接字类型（SOCK_STREAM=TCP、SOCK_DGRAM=UDP）
  int              ai_protocol;  // 协议（IPPROTO_TCP、IPPROTO_UDP，0=自动匹配）
  size_t           ai_addrlen;   // ai_addr 的长度（字节）
  char*            ai_canonname; // 主机规范名（可选）
  struct sockaddr* ai_addr;      // 指向 sockaddr_in（IPv4）/sockaddr_in6（IPv6）
  struct addrinfo* ai_next;      // 链表下一个节点（NULL 表示末尾）
};
```

### 常用 `ai_flags` 标志
| 标志          | 作用                                                                 |
|---------------|----------------------------------------------------------------------|
| `AI_PASSIVE`  | 用于服务器端：`node` 传 `NULL` 时，`ai_addr` 会设为 `0.0.0.0`（IPv4）/`::`（IPv6），表示绑定本机所有地址。 |
| `AI_CANONNAME`| 要求返回主机的规范名（填充到 `ai_canonname`）。|
| `AI_NUMERICHOST` | 强制 `node` 为数字 IP 字符串（如 `127.0.0.1`），避免域名解析（提升效率/避免DNS依赖）。 |
| `AI_NUMERICSERV` | 强制 `service` 为数字端口号，避免解析服务名。|

## 四、返回值
- **成功**：返回 `0`；
- **失败**：返回非 0 错误码（可通过 `gai_strerror()` 函数转换为可读字符串，Windows 下需用 `gai_strerrorA()`/`gai_strerrorW()` 区分编码）。

常见错误码：
- `EAI_NONAME`：主机名/服务名不存在；
- `EAI_AGAIN`：DNS 解析临时失败（重试即可）；
- `EAI_FAIL`：DNS 解析永久失败；
- `EAI_FAMILY`：`hints.ai_family` 指定的地址族不支持。

## 五、使用流程（Windows 特有）
Winsock 编程需先初始化套接字库，完整流程如下：
1. 调用 `WSAStartup()` 初始化 Winsock；
2. 初始化 `hints` 结构体（建议用 `memset` 清零，避免随机值）；
3. 调用 `getaddrinfo()` 解析地址；
4. 遍历 `res` 链表，使用 `ai_addr` 进行 `connect()`/`bind()` 等操作；
5. 调用 `freeaddrinfo()` 释放 `res` 链表（避免内存泄漏）；
6. 调用 `WSACleanup()` 清理 Winsock。

## 六、完整示例代码
### 示例 1：客户端解析域名并连接（TCP）
```c
#include <stdio.h>
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")  // 链接Winsock库

int main() {
    // 1. 初始化Winsock
    WSADATA wsaData;
    int ret = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (ret != 0) {
        printf("WSAStartup失败：%d\n", ret);
        return -1;
    }

    // 2. 初始化hints（筛选IPv4+TCP）
    struct addrinfo hints = {0};
    hints.ai_family = AF_INET;       // 只返回IPv4地址
    hints.ai_socktype = SOCK_STREAM; // TCP套接字
    hints.ai_protocol = IPPROTO_TCP; // TCP协议

    // 3. 解析地址（百度80端口）
    struct addrinfo* res = NULL;
    ret = getaddrinfo("www.baidu.com", "80", &hints, &res);
    if (ret != 0) {
        printf("getaddrinfo失败：%s\n", gai_strerrorA(ret));
        WSACleanup();
        return -1;
    }

    // 4. 遍历结果链表，尝试连接
    SOCKET sock = INVALID_SOCKET;
    struct addrinfo* p = NULL;
    for (p = res; p != NULL; p = p->ai_next) {
        // 创建套接字
        sock = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (sock == INVALID_SOCKET) {
            printf("socket失败：%d\n", WSAGetLastError());
            continue;
        }
        // 连接服务器
        ret = connect(sock, p->ai_addr, (int)p->ai_addrlen);
        if (ret == SOCKET_ERROR) {
            closesocket(sock);
            sock = INVALID_SOCKET;
            continue;
        }
        break; // 连接成功，退出循环
    }

    if (sock == INVALID_SOCKET) {
        printf("连接失败：%d\n", WSAGetLastError());
        freeaddrinfo(res);
        WSACleanup();
        return -1;
    }

    printf("连接百度80端口成功！\n");

    // 5. 释放资源
    closesocket(sock);
    freeaddrinfo(res); // 必须释放res链表
    WSACleanup();

    return 0;
}
```

### 示例 2：服务器端绑定本机所有地址（TCP）
```c
#include <stdio.h>
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);

    // 初始化hints（AI_PASSIVE用于服务器绑定）
    struct addrinfo hints = {0};
    hints.ai_flags = AI_PASSIVE;    // 被动模式（绑定本机所有地址）
    hints.ai_family = AF_UNSPEC;    // 支持IPv4/IPv6
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    struct addrinfo* res = NULL;
    int ret = getaddrinfo(NULL, "8888", &hints, &res); // node=NULL，绑定0.0.0.0/::
    if (ret != 0) {
        printf("getaddrinfo失败：%s\n", gai_strerrorA(ret));
        WSACleanup();
        return -1;
    }

    // 创建并绑定套接字
    SOCKET listenSock = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    bind(listenSock, res->ai_addr, (int)res->ai_addrlen);
    listen(listenSock, 5);

    printf("服务器已绑定8888端口，等待连接...\n");

    // 释放资源
    closesocket(listenSock);
    freeaddrinfo(res);
    WSACleanup();

    return 0;
}
```

## 七、核心优势
1. **跨协议兼容**：统一支持 IPv4/IPv6，无需修改代码即可适配双栈网络；
2. **线程安全**：相比旧函数 `gethostbyname()`（非线程安全），可安全用于多线程程序；
3. **灵活筛选**：通过 `hints` 精准控制返回结果（地址族、套接字类型、协议）；
4. **标准化**：遵循 RFC 3493，Windows/Linux 接口一致（仅初始化/清理步骤不同）。

## 八、常见注意事项
1. **内存泄漏**：`getaddrinfo()` 分配的 `res` 链表必须通过 `freeaddrinfo()` 释放，否则会导致内存泄漏；
2. **hints 初始化**：必须用 `memset(&hints, 0, sizeof(hints))` 清零，否则 `ai_flags`/`ai_family` 可能包含随机值，导致解析异常；
3. **错误处理**：`gai_strerror()` 需传入 `getaddrinfo()` 的返回值，而非 `WSAGetLastError()`；
4. **服务名解析**：`service` 传服务名（如 `"http"`）时，依赖系统的 `services` 配置文件，建议直接传数字端口号（更稳定）；
5. **IPv6 适配**：若需支持 IPv6，需确保系统开启 IPv6，且 `hints.ai_family` 设为 `AF_INET6` 或 `AF_UNSPEC`。

## 九、与旧函数的对比
| 函数          | 缺点                          | `getaddrinfo()` 优势                  |
|---------------|-------------------------------|---------------------------------------|
| `gethostbyname()` | 仅支持 IPv4、非线程安全、无筛选 | 支持 IPv4/IPv6、线程安全、灵活筛选    |
| `getservbyname()` | 仅解析服务名、需手动拼接端口   | 直接整合端口/服务名解析，无需额外处理 |

综上，`getaddrinfo()` 是 Winsock 网络编程中地址解析的“标准方案”，几乎所有现代套接字程序都应优先使用该函数。