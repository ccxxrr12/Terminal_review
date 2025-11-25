### 一、Windows Socket基础（Winsock）
Windows Socket（简称Winsock）是Windows平台上的网络编程接口，基于BSD Socket扩展，支持TCP/IP等协议。所有套接字操作均需基于Winsock库，核心步骤包括初始化、创建套接字、通信、清理。


#### 1. 环境准备
- **开发工具**：Visual Studio（推荐）、MinGW等。  
- **链接库**：Winsock编程需链接`ws2_32.lib`（VS中可通过`#pragma comment(lib, "ws2_32.lib")`自动链接）。  
- **头文件**：包含`<winsock2.h>`（核心API）和`<ws2tcpip.h>`（扩展函数）。  


#### 2. 核心初始化与清理
所有Winsock程序必须先初始化库，结束后清理，否则会导致资源泄漏。  

```c
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")  // 链接Winsock库

int main() {
    // 1. 初始化Winsock（指定版本2.2）
    WSADATA wsaData;
    int ret = WSAStartup(MAKEWORD(2, 2), &wsaData);  // 版本号2.2为常用版本
    if (ret != 0) {
        printf("WSAStartup失败，错误码：%d\n", WSAGetLastError());
        return 1;
    }

    // 2. 套接字操作（后续内容）...

    // 3. 清理Winsock
    WSACleanup();
    return 0;
}
```

- `WSAStartup`：初始化Winsock库，参数为版本号（`MAKEWORD(主版本, 次版本)`）和`WSADATA`结构体（返回库信息）。  
- `WSACleanup`：释放Winsock资源，必须在程序结束前调用。  


#### 3. 核心数据结构与函数
- **套接字描述符**：`SOCKET`类型（本质是无符号整数），标识一个套接字。  
- **地址结构**：`sockaddr_in`（IPv4）或`sockaddr_in6`（IPv6），用于存储IP和端口。  
  ```c
  struct sockaddr_in {
      short sin_family;         // 地址族（AF_INET表示IPv4）
      u_short sin_port;         // 端口号（网络字节序）
      struct in_addr sin_addr;  // IP地址（网络字节序）
      char sin_zero[8];         // 填充，使结构大小与sockaddr一致
  };
  struct in_addr {
      uint32_t s_addr;          // 32位IPv4地址（网络字节序）
  };
  ```
- **字节序转换**：网络协议使用**大端字节序**，Windows默认**小端字节序**，需通过以下函数转换：  
  - `htons()`：主机字节序→网络字节序（16位，端口）。  
  - `htonl()`：主机字节序→网络字节序（32位，IP地址）。  
  - `ntohs()`/`ntohl()`：网络字节序→主机字节序。  


### 二、流式套接字（SOCK_STREAM）
流式套接字基于**TCP协议**，特点：**面向连接**、**可靠传输**（无丢失/重复）、**字节流**（无边界），适用于文件传输、HTTP等需可靠通信的场景。


#### 1. 通信流程
- **服务器**：创建套接字→绑定（bind）→监听（listen）→接受连接（accept）→收发数据（send/recv）→关闭套接字。  
- **客户端**：创建套接字→连接服务器（connect）→收发数据→关闭套接字。  


#### 2. 服务器示例（TCP）
```c
void tcp_server() {
    // 1. 创建流式套接字（IPv4，TCP）
    SOCKET serverSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (serverSock == INVALID_SOCKET) {
        printf("创建套接字失败：%d\n", WSAGetLastError());
        return;
    }

    // 2. 绑定IP和端口
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8080);  // 端口8080（网络字节序）
    serverAddr.sin_addr.s_addr = INADDR_ANY;  // 绑定所有本地IP
    if (bind(serverSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        printf("绑定失败：%d\n", WSAGetLastError());
        closesocket(serverSock);
        return;
    }

    // 3. 监听连接（最大等待队列长度5）
    if (listen(serverSock, 5) == SOCKET_ERROR) {
        printf("监听失败：%d\n", WSAGetLastError());
        closesocket(serverSock);
        return;
    }
    printf("TCP服务器启动，监听端口8080...\n");

    // 4. 接受客户端连接
    sockaddr_in clientAddr;
    int clientAddrLen = sizeof(clientAddr);
    SOCKET clientSock = accept(serverSock, (SOCKADDR*)&clientAddr, &clientAddrLen);
    if (clientSock == INVALID_SOCKET) {
        printf("接受连接失败：%d\n", WSAGetLastError());
        closesocket(serverSock);
        return;
    }
    printf("客户端连接：%s:%d\n", inet_ntoa(clientAddr.sin_addr), ntohs(clientAddr.sin_port));

    // 5. 收发数据
    char buf[1024];
    int recvLen = recv(clientSock, buf, sizeof(buf)-1, 0);  // 接收数据
    if (recvLen > 0) {
        buf[recvLen] = '\0';
        printf("收到数据：%s\n", buf);

        // 发送响应
        const char* resp = "已收到数据";
        send(clientSock, resp, strlen(resp), 0);
    }

    // 6. 关闭套接字
    closesocket(clientSock);
    closesocket(serverSock);
}
```


#### 3. 客户端示例（TCP）
```c
void tcp_client() {
    // 1. 创建流式套接字
    SOCKET clientSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (clientSock == INVALID_SOCKET) {
        printf("创建套接字失败：%d\n", WSAGetLastError());
        return;
    }

    // 2. 连接服务器
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8080);
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1");  // 服务器IP（本地回环）
    if (connect(clientSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        printf("连接失败：%d\n", WSAGetLastError());
        closesocket(clientSock);
        return;
    }
    printf("已连接到服务器\n");

    // 3. 发送数据
    const char* msg = "Hello TCP Server";
    send(clientSock, msg, strlen(msg), 0);

    // 4. 接收响应
    char buf[1024];
    int recvLen = recv(clientSock, buf, sizeof(buf)-1, 0);
    if (recvLen > 0) {
        buf[recvLen] = '\0';
        printf("收到响应：%s\n", buf);
    }

    // 5. 关闭套接字
    closesocket(clientSock);
}
```


#### 4. 核心函数解析
- `socket(af, type, protocol)`：创建套接字。  
  - `af`：地址族（`AF_INET`=IPv4）。  
  - `type`：`SOCK_STREAM`（流式，TCP）。  
  - `protocol`：`IPPROTO_TCP`（TCP协议）。  
- `bind(sock, addr, len)`：将套接字绑定到指定IP和端口（服务器必做）。  
- `listen(sock, backlog)`：使套接字进入监听状态，`backlog`为最大等待连接数。  
- `accept(sock, clientAddr, len)`：阻塞等待客户端连接，返回新的套接字（与客户端通信）。  
- `connect(sock, serverAddr, len)`：客户端主动连接服务器。  
- `send(sock, buf, len, flags)`：发送数据（`flags`通常为0）。  
- `recv(sock, buf, len, flags)`：接收数据（返回实际接收字节数，0表示连接关闭，`SOCKET_ERROR`表示错误）。  


### 三、数据报套接字（SOCK_DGRAM）
数据报套接字基于**UDP协议**，特点：**无连接**、**不可靠**（可能丢失/重复/乱序）、**数据报边界保留**，适用于实时通信（如视频、语音）、DNS等场景。


#### 1. 通信流程
- 无需“连接”步骤，双方直接通过“发送到目标地址”和“从源地址接收”通信。  
- **服务器**：创建套接字→绑定→接收/发送数据→关闭。  
- **客户端**：创建套接字→（可选绑定）→发送/接收数据→关闭。  


#### 2. 服务器示例（UDP）
```c
void udp_server() {
    // 1. 创建数据报套接字（IPv4，UDP）
    SOCKET serverSock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (serverSock == INVALID_SOCKET) {
        printf("创建套接字失败：%d\n", WSAGetLastError());
        return;
    }

    // 2. 绑定IP和端口
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8081);  // 端口8081
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    if (bind(serverSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        printf("绑定失败：%d\n", WSAGetLastError());
        closesocket(serverSock);
        return;
    }
    printf("UDP服务器启动，监听端口8081...\n");

    // 3. 接收数据（同时获取客户端地址）
    char buf[1024];
    sockaddr_in clientAddr;
    int clientAddrLen = sizeof(clientAddr);
    int recvLen = recvfrom(serverSock, buf, sizeof(buf)-1, 0, 
                          (SOCKADDR*)&clientAddr, &clientAddrLen);
    if (recvLen > 0) {
        buf[recvLen] = '\0';
        printf("收到来自%s:%d的数据：%s\n", 
               inet_ntoa(clientAddr.sin_addr), 
               ntohs(clientAddr.sin_port), 
               buf);

        // 4. 发送响应（指定客户端地址）
        const char* resp = "已收到UDP数据";
        sendto(serverSock, resp, strlen(resp), 0, 
              (SOCKADDR*)&clientAddr, clientAddrLen);
    }

    // 5. 关闭套接字
    closesocket(serverSock);
}
```


#### 3. 客户端示例（UDP）
```c
void udp_client() {
    // 1. 创建数据报套接字
    SOCKET clientSock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (clientSock == INVALID_SOCKET) {
        printf("创建套接字失败：%d\n", WSAGetLastError());
        return;
    }

    // 2. 发送数据到服务器（无需连接，直接指定地址）
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8081);
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1");
    const char* msg = "Hello UDP Server";
    sendto(clientSock, msg, strlen(msg), 0, 
          (SOCKADDR*)&serverAddr, sizeof(serverAddr));

    // 3. 接收服务器响应（获取服务器地址）
    char buf[1024];
    sockaddr_in serverRespAddr;
    int serverAddrLen = sizeof(serverRespAddr);
    int recvLen = recvfrom(clientSock, buf, sizeof(buf)-1, 0, 
                          (SOCKADDR*)&serverRespAddr, &serverAddrLen);
    if (recvLen > 0) {
        buf[recvLen] = '\0';
        printf("收到服务器响应：%s\n", buf);
    }

    // 4. 关闭套接字
    closesocket(clientSock);
}
```


#### 4. 核心函数解析
- `socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)`：创建UDP套接字。  
- `sendto(sock, buf, len, flags, to, tolen)`：发送数据到指定地址（`to`为目标地址）。  
- `recvfrom(sock, buf, len, flags, from, fromlen)`：接收数据，并获取发送方地址（`from`输出源地址）。  


### 四、原始套接字（SOCK_RAW）
原始套接字允许直接操作**底层协议**（如IP、ICMP、TCP、UDP），可构造/解析原始数据包，用于网络诊断（如ping）、协议分析等。但权限要求高（需管理员权限），且受Windows系统限制。


#### 1. 核心特点
- 可发送/接收未经过TCP/IP协议栈处理的原始数据包。  
- 支持`IPPROTO_TCP`、`IPPROTO_UDP`、`IPPROTO_ICMP`等协议类型。  
- Windows XP及以上系统中，普通用户无法创建原始套接字，需以管理员身份运行程序。  


#### 2. 编程流程（以ICMP为例，实现简易ping）
ICMP（互联网控制消息协议）用于网络诊断，ping命令基于ICMP的“回声请求”（类型8）和“回声应答”（类型0）。

```c
#include <iphlpapi.h>  // 包含ICMP相关结构
#pragma comment(lib, "iphlpapi.lib")  // 链接ICMP库

void raw_socket_ping() {
    // 1. 创建原始套接字（ICMP协议）
    SOCKET rawSock = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
    if (rawSock == INVALID_SOCKET) {
        printf("创建原始套接字失败（需管理员权限）：%d\n", WSAGetLastError());
        return;
    }

    // 2. 设置超时（接收超时5秒）
    int timeout = 5000;  // 毫秒
    setsockopt(rawSock, SOL_SOCKET, SO_RCVTIMEO, (char*)&timeout, sizeof(timeout));

    // 3. 构造ICMP回声请求包
    char icmpBuf[1024];
    memset(icmpBuf, 0, sizeof(icmpBuf));

    // ICMP头部（8字节）
    struct icmp_hdr {
        unsigned char type;   // 类型（8=请求，0=应答）
        unsigned char code;   // 代码（0）
        unsigned short checksum;  // 校验和
        unsigned short id;    // 标识符（区分不同ping）
        unsigned short seq;   // 序列号
    }*icmp = (struct icmp_hdr*)icmpBuf;

    icmp->type = 8;  // 回声请求
    icmp->code = 0;
    icmp->id = (unsigned short)GetCurrentProcessId();  // 用进程ID作为标识
    icmp->seq = 1;  // 序列号
    icmp->checksum = 0;  // 先填0，计算后再赋值

    // 填充数据（可选，一般填随机数或固定值）
    char* data = icmpBuf + sizeof(struct icmp_hdr);
    int dataLen = 32;  // 数据部分32字节
    for (int i = 0; i < dataLen; i++) {
        data[i] = 'A' + i % 26;
    }

    // 计算ICMP校验和
    icmp->checksum = checksum((unsigned short*)icmpBuf, sizeof(struct icmp_hdr) + dataLen);

    // 4. 发送到目标IP（如8.8.8.8）
    sockaddr_in destAddr;
    destAddr.sin_family = AF_INET;
    destAddr.sin_addr.s_addr = inet_addr("8.8.8.8");  // 目标IP
    int sendLen = sendto(rawSock, icmpBuf, sizeof(struct icmp_hdr) + dataLen, 0,
                        (SOCKADDR*)&destAddr, sizeof(destAddr));
    if (sendLen == SOCKET_ERROR) {
        printf("发送失败：%d\n", WSAGetLastError());
        closesocket(rawSock);
        return;
    }

    // 5. 接收ICMP应答（包含IP头部+ICMP头部+数据）
    char recvBuf[1024];
    sockaddr_in srcAddr;
    int srcAddrLen = sizeof(srcAddr);
    int recvLen = recvfrom(rawSock, recvBuf, sizeof(recvBuf), 0,
                          (SOCKADDR*)&srcAddr, &srcAddrLen);
    if (recvLen > 0) {
        // IP头部长度（前4位为版本，后4位为头部长度（单位：32位字））
        int ipHeaderLen = (recvBuf[0] & 0x0F) * 4;
        // 跳过IP头部，获取ICMP应答
        struct icmp_hdr* respIcmp = (struct icmp_hdr*)(recvBuf + ipHeaderLen);

        if (respIcmp->type == 0 && respIcmp->id == icmp->id && respIcmp->seq == icmp->seq) {
            printf("收到来自%s的应答，序列号：%d\n", 
                   inet_ntoa(srcAddr.sin_addr), respIcmp->seq);
        }
    } else {
        printf("接收超时或失败：%d\n", WSAGetLastError());
    }

    // 6. 关闭套接字
    closesocket(rawSock);
}

// 辅助函数：计算校验和（RFC 1071标准）
unsigned short checksum(unsigned short* buf, int len) {
    unsigned long sum = 0;
    while (len > 1) {
        sum += *buf++;
        len -= 2;
    }
    if (len == 1) {
        sum += *(unsigned char*)buf;
    }
    sum = (sum >> 16) + (sum & 0xFFFF);
    sum += sum >> 16;
    return (unsigned short)~sum;
}
```


#### 3. 核心注意事项
- **权限**：必须以管理员身份运行程序，否则`socket`函数会返回`WSAEACCES`（10013）错误。  
- **IP头部处理**：若需自定义IP头部，需设置套接字选项`IP_HDRINCL`（告知系统不自动添加IP头）：  
  ```c
  int flag = 1;
  setsockopt(rawSock, IPPROTO_IP, IP_HDRINCL, (char*)&flag, sizeof(flag));
  ```
- **系统限制**：Windows对原始套接字有严格限制（如禁止伪造源IP发送TCP包），不同版本行为可能不同。  


### 五、三种套接字对比
| 类型         | 协议   | 连接性 | 可靠性 | 适用场景                     | 核心函数                     |
|--------------|--------|--------|--------|------------------------------|------------------------------|
| 流式套接字   | TCP    | 面向连接 | 可靠   | 文件传输、HTTP、登录         | `connect`/`listen`/`accept`  |
| 数据报套接字 | UDP    | 无连接  | 不可靠 | 实时通信、DNS、广播          | `sendto`/`recvfrom`          |
| 原始套接字   | 底层协议（IP/ICMP等） | 无连接  | 自定义 | 网络诊断（ping）、协议分析   | 需手动构造/解析数据包        |


### 六、错误处理
所有Winsock函数失败时返回`SOCKET_ERROR`，可通过`WSAGetLastError()`获取具体错误码（如10061表示连接被拒绝，10049表示地址不可用）。

### 七、winsocket函数
好的，我们来深入探讨一下 Winsock 的函数。

Winsock（Windows Sockets API）是 Windows 平台上进行网络编程的核心接口，它提供了一系列函数用于创建套接字、建立连接、数据传输等操作。下面我将为你详细梳理 Winsock 中最常用的函数，包括它们的定义、参数说明和使用示例。  

---

#### **1. WSAStartup**
**功能**：初始化 Winsock 库。  
在使用任何 Winsock 函数之前，必须先调用 `WSAStartup`。

```c
#include <winsock2.h>

int WSAAPI WSAStartup(
  WORD      wVersionRequested,  // 请求的 Winsock 版本
  LPWSADATA lpWSAData           // 指向 WSADATA 结构体的指针
);
```

**参数**：
- `wVersionRequested`：  
  - 高字节为次要版本，低字节为主要版本。  
  - 常用 `MAKEWORD(2, 2)` 请求 2.2 版本。
- `lpWSAData`：  
  - 返回 Winsock 库的详细信息，如版本号、供应商信息等。

**返回值**：
- 成功：`0`
- 失败：返回错误码，可通过 `WSAGetLastError()` 获取详细信息。

**示例**：
```c
WSADATA wsaData;
if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
    printf("WSAStartup 失败: %d\n", WSAGetLastError());
    return 1;
}
```

---

#### **2. WSACleanup**
**功能**：释放 Winsock 库占用的资源。  
在程序退出前，必须调用 `WSACleanup`。

```c
int WSAAPI WSACleanup(void);
```

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`，可通过 `WSAGetLastError()` 获取错误码。

**示例**：
```c
WSACleanup();
```

---

#### **3. socket**
**功能**：创建一个套接字。

```c
SOCKET WSAAPI socket(
  int af,                // 地址族 (Address Family)
  int type,              // 套接字类型
  int protocol           // 协议类型
);
```

**参数**：
- `af`：
  - `AF_INET`：IPv4
  - `AF_INET6`：IPv6
  - `AF_UNSPEC`：未指定
- `type`：
  - `SOCK_STREAM`：流式套接字（TCP）
  - `SOCK_DGRAM`：数据报套接字（UDP）
  - `SOCK_RAW`：原始套接字
- `protocol`：
  - `IPPROTO_TCP`：TCP 协议
  - `IPPROTO_UDP`：UDP 协议
  - `IPPROTO_ICMP`：ICMP 协议
  - `0`：自动选择与 `type` 对应的协议

**返回值**：
- 成功：返回新创建的套接字（非负整数）
- 失败：`INVALID_SOCKET`，可通过 `WSAGetLastError()` 获取错误码。

**示例**：
```c
// 创建 TCP 套接字
SOCKET tcpSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
if (tcpSocket == INVALID_SOCKET) {
    printf("socket 创建失败: %d\n", WSAGetLastError());
    WSACleanup();
    return 1;
}

// 创建 UDP 套接字
SOCKET udpSocket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
```

---

#### **4. bind**
**功能**：将套接字绑定到特定的 IP 地址和端口。

```c
int WSAAPI bind(
  SOCKET         s,                // 套接字
  const sockaddr *name,            // 指向 sockaddr 结构体的指针
  int            namelen           // sockaddr 结构体的大小
);
```

**参数**：
- `s`：
  - 要绑定的套接字。
- `name`：
  - 指向 `sockaddr_in` (IPv4) 或 `sockaddr_in6` (IPv6) 结构体的指针。
- `namelen`：
  - `name` 指向的结构体的大小。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
struct sockaddr_in serverAddr;
serverAddr.sin_family = AF_INET;
serverAddr.sin_port = htons(8080); // 端口 8080
serverAddr.sin_addr.s_addr = INADDR_ANY; // 绑定到所有可用的接口

if (bind(tcpSocket, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
    printf("bind 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
```

---

#### **5. listen**
**功能**：使流式套接字进入监听状态，等待客户端连接。

```c
int WSAAPI listen(
  SOCKET s,      // 套接字
  int    backlog // 等待连接队列的最大长度
);
```

**参数**：
- `s`：
  - 已绑定的流式套接字。
- `backlog`：
  - 等待连接队列的最大长度。
  - 如果队列满了，新的连接请求会被拒绝。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
if (listen(tcpSocket, 5) == SOCKET_ERROR) {
    printf("listen 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("服务器正在监听端口 8080...\n");
```

---

#### **6. accept**
**功能**：接受客户端的连接请求，返回一个新的套接字用于与该客户端通信。

```c
SOCKET WSAAPI accept(
  SOCKET         s,                // 监听套接字
  sockaddr       *addr,            // 指向客户端地址信息的结构体
  int            *addrlen          // 客户端地址结构体的大小
);
```

**参数**：
- `s`：
  - 正在监听的套接字。
- `addr`：
  - 用于存储客户端地址信息。
- `addrlen`：
  - 指向一个整数，该整数指定了 `addr` 结构体的大小。

**返回值**：
- 成功：返回新的套接字
- 失败：`INVALID_SOCKET`

**示例**：
```c
struct sockaddr_in clientAddr;
int clientAddrSize = sizeof(clientAddr);

SOCKET clientSocket = accept(tcpSocket, (SOCKADDR*)&clientAddr, &clientAddrSize);
if (clientSocket == INVALID_SOCKET) {
    printf("accept 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("客户端连接成功: %s:%d\n", inet_ntoa(clientAddr.sin_addr), ntohs(clientAddr.sin_port));
```

---

#### **7. connect**
**功能**：客户端向服务器发起连接请求。

```c
int WSAAPI connect(
  SOCKET         s,                // 套接字
  const sockaddr *name,            // 服务器地址信息
  int            namelen           // 地址信息的大小
);
```

**参数**：
- `s`：
  - 未绑定的套接字。
- `name`：
  - 指向 `sockaddr_in` 或 `sockaddr_in6` 结构体的指针。
- `namelen`：
  - `name` 指向的结构体的大小。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
struct sockaddr_in serverAddr;
serverAddr.sin_family = AF_INET;
serverAddr.sin_port = htons(8080);
serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1"); // 服务器 IP 地址

if (connect(tcpSocket, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
    printf("connect 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("连接到服务器成功!\n");
```

---

#### **8. send**
**功能**：在流式套接字上发送数据。

```c
int WSAAPI send(
  SOCKET     s,            // 已连接的套接字
  const char *buf,         // 指向要发送的数据的指针
  int        len,          // 数据的长度
  int        flags         // 发送标志
);
```

**参数**：
- `s`：
  - 已连接的流式套接字。
- `buf`：
  - 指向要发送的数据的缓冲区。
- `len`：
  - 要发送的数据的字节数。
- `flags`：
  - 发送标志，通常为 `0`。
  - `MSG_OOB`：发送带外数据。

**返回值**：
- 成功：返回实际发送的字节数
- 失败：`SOCKET_ERROR`

**示例**：
```c
const char *sendData = "Hello, Server!";
int bytesSent = send(clientSocket, sendData, strlen(sendData), 0);
if (bytesSent == SOCKET_ERROR) {
    printf("send 失败: %d\n", WSAGetLastError());
    closesocket(clientSocket);
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("发送了 %d 字节的数据: %s\n", bytesSent, sendData);
```

---

#### **9. recv**
**功能**：在流式套接字上接收数据。

```c
int WSAAPI recv(
  SOCKET s,         // 已连接的套接字
  char   *buf,      // 接收数据的缓冲区
  int    len,       // 缓冲区的大小
  int    flags      // 接收标志
);
```

**参数**：
- `s`：
  - 已连接的流式套接字。
- `buf`：
  - 用于存储接收数据的缓冲区。
- `len`：
  - 缓冲区的最大容量。
- `flags`：
  - 接收标志，通常为 `0`。
  - `MSG_PEEK`：查看数据但不从输入队列中移除。

**返回值**：
- 成功：返回实际接收的字节数
- 连接关闭：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
char recvBuf[1024];
int bytesRecv = recv(clientSocket, recvBuf, sizeof(recvBuf), 0);
if (bytesRecv > 0) {
    recvBuf[bytesRecv] = '\0';
    printf("收到 %d 字节的数据: %s\n", bytesRecv, recvBuf);
} else if (bytesRecv == 0) {
    printf("服务器关闭了连接\n");
} else {
    printf("recv 失败: %d\n", WSAGetLastError());
}
```

---

#### **10. sendto**
**功能**：在数据报套接字上发送数据。

```c
int WSAAPI sendto(
  SOCKET         s,            // 套接字
  const char     *buf,         // 指向要发送的数据的指针
  int            len,          // 数据的长度
  int            flags,        // 发送标志
  const sockaddr *to,          // 目标地址信息
  int            tolen         // 目标地址信息的大小
);
```

**参数**：
- `s`：
  - 数据报套接字。
- `buf`：
  - 指向要发送的数据的缓冲区。
- `len`：
  - 要发送的数据的字节数。
- `flags`：
  - 发送标志，通常为 `0`。
- `to`：
  - 指向目标地址信息的结构体。
- `tolen`：
  - `to` 指向的结构体的大小。

**返回值**：
- 成功：返回实际发送的字节数
- 失败：`SOCKET_ERROR`

**示例**：
```c
struct sockaddr_in serverAddr;
serverAddr.sin_family = AF_INET;
serverAddr.sin_port = htons(8080);
serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1");

const char *sendData = "Hello, UDP Server!";
int bytesSent = sendto(udpSocket, sendData, strlen(sendData), 0, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
if (bytesSent == SOCKET_ERROR) {
    printf("sendto 失败: %d\n", WSAGetLastError());
    closesocket(udpSocket);
    WSACleanup();
    return 1;
}
printf("发送了 %d 字节的数据: %s\n", bytesSent, sendData);
```

---

#### **11. recvfrom**
**功能**：在数据报套接字上接收数据。

```c
int WSAAPI recvfrom(
  SOCKET    s,          // 套接字
  char      *buf,       // 接收数据的缓冲区
  int       len,        // 缓冲区的大小
  int       flags,      // 接收标志
  sockaddr  *from,      // 发送方地址信息
  int       *fromlen    // 发送方地址信息的大小
);
```

**参数**：
- `s`：
  - 数据报套接字。
- `buf`：
  - 用于存储接收数据的缓冲区。
- `len`：
  - 缓冲区的最大容量。
- `flags`：
  - 接收标志，通常为 `0`。
- `from`：
  - 指向发送方地址信息的结构体。
- `fromlen`：
  - 指向一个整数，该整数指定了 `from` 结构体的大小。

**返回值**：
- 成功：返回实际接收的字节数
- 失败：`SOCKET_ERROR`

**示例**：
```c
struct sockaddr_in clientAddr;
int clientAddrSize = sizeof(clientAddr);
char recvBuf[1024];

int bytesRecv = recvfrom(udpSocket, recvBuf, sizeof(recvBuf), 0, (SOCKADDR*)&clientAddr, &clientAddrSize);
if (bytesRecv > 0) {
    recvBuf[bytesRecv] = '\0';
    printf("收到来自 %s:%d 的数据: %s\n", inet_ntoa(clientAddr.sin_addr), ntohs(clientAddr.sin_port), recvBuf);
} else {
    printf("recvfrom 失败: %d\n", WSAGetLastError());
}
```

---

#### **12. closesocket**
**功能**：关闭套接字。

```c
int WSAAPI closesocket(
  SOCKET s  // 要关闭的套接字
);
```

**参数**：
- `s`：
  - 要关闭的套接字。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
closesocket(clientSocket);
closesocket(tcpSocket);
```

---

#### **13. getaddrinfo**
**功能**：将主机名或 IP 地址字符串转换为套接字地址结构。

```c
int WSAAPI getaddrinfo(
  const char *nodename,       // 主机名或 IP 地址
  const char *servname,       // 服务名或端口号
  const struct addrinfo *hints, // 指向 addrinfo 结构体的指针
  struct addrinfo **res       // 指向 addrinfo 结构体链表的指针
);
```

**参数**：
- `nodename`：
  - 主机名或 IP 地址。
- `servname`：
  - 服务名或端口号。
- `hints`：
  - 指向一个 `addrinfo` 结构体，用于指定所需的地址族、套接字类型等。
- `res`：
  - 指向一个 `addrinfo` 结构体链表的指针，用于存储结果。

**返回值**：
- 成功：`0`
- 失败：返回非零错误码，可通过 `gai_strerror()` 获取错误信息。

**示例**：
```c
struct addrinfo hints, *result;
memset(&hints, 0, sizeof(hints));
hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;

int ret = getaddrinfo("www.example.com", "80", &hints, &result);
if (ret != 0) {
    printf("getaddrinfo 失败: %s\n", gai_strerror(ret));
    WSACleanup();
    return 1;
}

// 使用 result 中的地址信息进行连接
struct sockaddr_in *serverAddr = (struct sockaddr_in*)result->ai_addr;
printf("服务器 IP 地址: %s\n", inet_ntoa(serverAddr->sin_addr));

freeaddrinfo(result); // 释放内存
```

---

#### **14. inet_pton**
**功能**：将 IP 地址字符串转换为二进制格式。

```c
int WSAAPI inet_pton(
  int   af,          // 地址族
  const char *src,   // IP 地址字符串
  void   *dst        // 存储二进制 IP 地址的缓冲区
);
```

**参数**：
- `af`：
  - `AF_INET`：IPv4
  - `AF_INET6`：IPv6
- `src`：
  - IP 地址字符串。
- `dst`：
  - 用于存储二进制 IP 地址的缓冲区。

**返回值**：
- 成功：`1`
- 失败：`0`（格式错误）或 `-1`（地址族不支持）

**示例**：
```c
struct in_addr addr;
if (inet_pton(AF_INET, "127.0.0.1", &addr) == 1) {
    printf("IP 地址转换成功: %u\n", addr.s_addr);
} else {
    printf("IP 地址格式错误\n");
}
```

---

#### **15. inet_ntop**
**功能**：将二进制 IP 地址转换为字符串格式。

```c
const char *WSAAPI inet_ntop(
  int         af,        // 地址族
  const void *src,       // 二进制 IP 地址
  char       *dst,       // 存储 IP 地址字符串的缓冲区
  socklen_t   size       // 缓冲区的大小
);
```

**参数**：
- `af`：
  - `AF_INET`：IPv4
  - `AF_INET6`：IPv6
- `src`：
  - 二进制 IP 地址。
- `dst`：
  - 用于存储 IP 地址字符串的缓冲区。
- `size`：
  - 缓冲区的大小。

**返回值**：
- 成功：返回指向 `dst` 的指针
- 失败：`NULL`

**示例**：
```c
struct in_addr addr;
addr.s_addr = inet_addr("127.0.0.1");

char ipStr[INET_ADDRSTRLEN];
if (inet_ntop(AF_INET, &addr, ipStr, sizeof(ipStr)) != NULL) {
    printf("IP 地址字符串: %s\n", ipStr);
} else {
    printf("IP 地址转换失败\n");
}
```

---

#### **16. setsockopt**
**功能**：设置套接字的选项。

```c
int WSAAPI setsockopt(
  SOCKET         s,                // 套接字
  int            level,            // 选项级别
  int            optname,          // 选项名称
  const char     *optval,          // 选项值
  int            optlen            // 选项值的大小
);
```

**参数**：
- `s`：
  - 套接字。
- `level`：
  - 选项级别。
  - `SOL_SOCKET`：套接字级别。
  - `IPPROTO_TCP`：TCP 协议级别。
  - `IPPROTO_IP`：IP 协议级别。
- `optname`：
  - 选项名称。
  - `SO_REUSEADDR`：允许重用本地地址。
  - `SO_RCVTIMEO`：接收超时。
  - `SO_SNDTIMEO`：发送超时。
  - `TCP_NODELAY`：禁用 Nagle 算法。
- `optval`：
  - 指向选项值的指针。
- `optlen`：
  - 选项值的大小。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
// 设置接收超时为 5 秒
int recvTimeout = 5000; // 毫秒
if (setsockopt(tcpSocket, SOL_SOCKET, SO_RCVTIMEO, (char*)&recvTimeout, sizeof(recvTimeout)) == SOCKET_ERROR) {
    printf("setsockopt 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}

// 允许重用本地地址
int reuseAddr = 1;
if (setsockopt(tcpSocket, SOL_SOCKET, SO_REUSEADDR, (char*)&reuseAddr, sizeof(reuseAddr)) == SOCKET_ERROR) {
    printf("setsockopt 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
```

---

#### **17. getsockopt**
**功能**：获取套接字的选项值。

```c
int WSAAPI getsockopt(
  SOCKET         s,                // 套接字
  int            level,            // 选项级别
  int            optname,          // 选项名称
  char           *optval,          // 存储选项值的缓冲区
  int            *optlen           // 缓冲区的大小
);
```

**参数**：
- `s`：
  - 套接字。
- `level`：
  - 选项级别。
- `optname`：
  - 选项名称。
- `optval`：
  - 用于存储选项值的缓冲区。
- `optlen`：
  - 指向一个整数，该整数指定了 `optval` 缓冲区的大小。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
// 获取接收超时
int recvTimeout;
int optlen = sizeof(recvTimeout);
if (getsockopt(tcpSocket, SOL_SOCKET, SO_RCVTIMEO, (char*)&recvTimeout, &optlen) == SOCKET_ERROR) {
    printf("getsockopt 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("接收超时: %d 毫秒\n", recvTimeout);
```

---

#### **18. ioctlsocket**
**功能**：控制套接字的 I/O 模式。

```c
int WSAAPI ioctlsocket(
  SOCKET  s,        // 套接字
  long    cmd,      // 控制命令
  u_long  *argp     // 指向参数的指针
);
```

**参数**：
- `s`：
  - 套接字。
- `cmd`：
  - 控制命令。
  - `FIONBIO`：设置套接字为非阻塞模式。
  - `FIONREAD`：获取可读数据的字节数。
- `argp`：
  - 指向一个 `u_long` 类型的指针，用于存储命令的参数。

**返回值**：
- 成功：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
// 设置套接字为非阻塞模式
u_long nonBlocking = 1;
if (ioctlsocket(tcpSocket, FIONBIO, &nonBlocking) == SOCKET_ERROR) {
    printf("ioctlsocket 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}

// 获取可读数据的字节数
u_long bytesAvailable;
if (ioctlsocket(tcpSocket, FIONREAD, &bytesAvailable) == SOCKET_ERROR) {
    printf("ioctlsocket 失败: %d\n", WSAGetLastError());
    closesocket(tcpSocket);
    WSACleanup();
    return 1;
}
printf("可读数据字节数: %lu\n", bytesAvailable);
```

---

#### **19. select**
**功能**：等待一个或多个套接字上的事件。

```c
int WSAAPI select(
  int           nfds,        // 忽略
  fd_set        *readfds,    // 要检查可读性的套接字集合
  fd_set        *writefds,   // 要检查可写性的套接字集合
  fd_set        *exceptfds,  // 要检查异常的套接字集合
  const timeval *timeout     // 超时时间
);
```

**参数**：
- `nfds`：
  - 忽略，设置为 `0`。
- `readfds`：
  - 指向一个 `fd_set` 结构体，包含要检查可读性的套接字。
- `writefds`：
  - 指向一个 `fd_set` 结构体，包含要检查可写性的套接字。
- `exceptfds`：
  - 指向一个 `fd_set` 结构体，包含要检查异常的套接字。
- `timeout`：
  - 指向一个 `timeval` 结构体，指定超时时间。
  - `NULL`：无限等待。

**返回值**：
- 成功：返回准备好的套接字数量
- 超时：`0`
- 失败：`SOCKET_ERROR`

**示例**：
```c
fd_set readSet;
FD_ZERO(&readSet);
FD_SET(tcpSocket, &readSet);

struct timeval timeout;
timeout.tv_sec = 5; // 5 秒
timeout.tv_usec = 0;

int ret = select(0, &readSet, NULL, NULL, &timeout);
if (ret > 0) {
    if (FD_ISSET(tcpSocket, &readSet)) {
        printf("套接字有数据可读\n");
        char recvBuf[1024];
        int bytesRecv = recv(tcpSocket, recvBuf, sizeof(recvBuf), 0);
        if (bytesRecv > 0) {
            recvBuf[bytesRecv] = '\0';
            printf("收到数据: %s\n", recvBuf);
        }
    }
} else if (ret == 0) {
    printf("select 超时\n");
} else {
    printf("select 失败: %d\n", WSAGetLastError());
}
```

---

#### **20. WSAGetLastError**
**功能**：获取上一个 Winsock 函数调用的错误码。

```c
int WSAAPI WSAGetLastError(void);
```

**返回值**：
- 返回上一个 Winsock 函数调用的错误码。

**示例**：
```c
if (socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) == INVALID_SOCKET) {
    int errorCode = WSAGetLastError();
    printf("socket 创建失败，错误码: %d\n", errorCode);
}
```

---

## **总结**

以上是 Winsock 中最常用的 20 个函数，涵盖了套接字的创建、绑定、监听、连接、数据传输等基本操作。

**常用函数组合**：

1. **服务器端（TCP）**：
   ```
   WSAStartup -> socket -> bind -> listen -> accept -> recv/send -> closesocket -> WSACleanup
   ```

2. **客户端（TCP）**：
   ```
   WSAStartup -> socket -> connect -> send/recv -> closesocket -> WSACleanup
   ```

3. **服务器端（UDP）**：
   ```
   WSAStartup -> socket -> bind -> recvfrom -> sendto -> closesocket -> WSACleanup
   ```

4. **客户端（UDP）**：
   ```
   WSAStartup -> socket -> sendto -> recvfrom -> closesocket -> WSACleanup
   ```

---

### 总结
- 流式套接字（TCP）适合需可靠传输的场景，编程流程较复杂（需连接）。  
- 数据报套接字（UDP）适合实时性优先的场景，编程简单（无连接）。  
- 原始套接字适合底层网络操作，需管理员权限，受系统限制较多。  

实践时需注意字节序转换、地址结构填充和错误处理，可结合Wireshark等工具抓包分析数据流向。