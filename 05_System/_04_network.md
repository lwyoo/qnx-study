# QNX TCP/IP 네트워킹 프로그래밍

## 1. QNX 네트워크 스택 개요

### 스택 구조

```
Application (Socket API)
    │
    ├─→ TCP/UDP
    ├─→ IP/ICMP
    └─→ ARP/Link Layer
         │
         └─→ Ethernet Driver
```

### 특징

```
특징                설명
─────────────────────────────────────
POSIX 호환         표준 socket() 호출
실시간 성능        결정적 지연시간
우선순위 큐        QoS 지원
멀티캐스트         멀티캐스트 지원
IPv4/IPv6         듀얼 스택
```

---

## 2. 기본 Socket API

### Socket 생성

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

int socket(int domain, int type, int protocol);

// 예제: TCP 소켓 생성
int sock = socket(AF_INET,       // IPv4
                  SOCK_STREAM,   // TCP
                  IPPROTO_TCP);  // TCP 프로토콜

if (sock < 0) {
    perror("socket");
}
```

### Socket 종류

| Type | 설명 | 특징 |
|------|------|------|
| **SOCK_STREAM** | TCP | 연결지향, 신뢰성 |
| **SOCK_DGRAM** | UDP | 비연결, 빠름 |
| **SOCK_RAW** | Raw IP | 저수준 접근 |

---

## 3. TCP Server 구현

### 기본 구조

```
1. socket() - 소켓 생성
2. bind() - 포트에 바인딩
3. listen() - 연결 대기
4. accept() - 클라이언트 수락
5. read()/write() - 데이터 송수신
6. close() - 소켓 닫기
```

### 간단한 Echo Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 5555
#define BACKLOG 5
#define BUFFER_SIZE 1024

int main() {
    int server_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len;
    char buffer[BUFFER_SIZE];
    int nbytes;
    
    // 1. 소켓 생성
    server_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock < 0) {
        perror("socket");
        return EXIT_FAILURE;
    }
    
    // 2. 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);
    
    // 3. Bind
    if (bind(server_sock, (struct sockaddr *)&server_addr, 
             sizeof(server_addr)) < 0) {
        perror("bind");
        return EXIT_FAILURE;
    }
    
    printf("Server: Bound to port %d\n", PORT);
    
    // 4. Listen
    if (listen(server_sock, BACKLOG) < 0) {
        perror("listen");
        return EXIT_FAILURE;
    }
    
    printf("Server: Listening for connections...\n");
    
    // 5. Accept 루프
    while (1) {
        client_len = sizeof(client_addr);
        
        client_sock = accept(server_sock, 
                            (struct sockaddr *)&client_addr,
                            &client_len);
        
        if (client_sock < 0) {
            perror("accept");
            continue;
        }
        
        printf("Server: Client connected from %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));
        
        // 6. Echo 루프
        while (1) {
            nbytes = read(client_sock, buffer, BUFFER_SIZE);
            
            if (nbytes <= 0) {
                break;  // 클라이언트 연결 종료
            }
            
            printf("Server: Received %d bytes\n", nbytes);
            
            // 동일한 데이터 전송
            if (write(client_sock, buffer, nbytes) < 0) {
                perror("write");
                break;
            }
        }
        
        close(client_sock);
        printf("Server: Client disconnected\n");
    }
    
    close(server_sock);
    return EXIT_SUCCESS;
}
```

---

## 4. TCP Client 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 5555
#define BUFFER_SIZE 1024

int main(int argc, char *argv[]) {
    int sock;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];
    char message[BUFFER_SIZE];
    int nbytes;
    
    if (argc < 2) {
        printf("Usage: %s <server_ip>\n", argv[0]);
        return EXIT_FAILURE;
    }
    
    // 1. 소켓 생성
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        return EXIT_FAILURE;
    }
    
    // 2. 서버 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    
    if (inet_pton(AF_INET, argv[1], 
                  &server_addr.sin_addr) <= 0) {
        perror("inet_pton");
        return EXIT_FAILURE;
    }
    
    // 3. Connect
    if (connect(sock, (struct sockaddr *)&server_addr,
                sizeof(server_addr)) < 0) {
        perror("connect");
        return EXIT_FAILURE;
    }
    
    printf("Client: Connected to server\n");
    
    // 4. 메시지 송수신
    strcpy(message, "Hello from client!");
    
    nbytes = write(sock, message, strlen(message));
    printf("Client: Sent %d bytes\n", nbytes);
    
    nbytes = read(sock, buffer, BUFFER_SIZE);
    printf("Client: Received: %.*s\n", nbytes, buffer);
    
    close(sock);
    return EXIT_SUCCESS;
}
```

### 컴파일 및 실행

```bash
# 컴파일
qcc -o echo_server echo_server.c
qcc -o echo_client echo_client.c

# 터미널 1: 서버 실행
./echo_server
# 출력: Server: Listening for connections...

# 터미널 2: 클라이언트 실행
./echo_client 127.0.0.1
# 출력:
# Client: Connected to server
# Client: Sent 19 bytes
# Client: Received: Hello from client!
```

---

## 5. UDP (비연결) 통신

### UDP Server

```c
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    int sock;
    struct sockaddr_in addr, client_addr;
    socklen_t client_len;
    char buffer[256];
    int nbytes;
    
    sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(5555);
    
    bind(sock, (struct sockaddr *)&addr, sizeof(addr));
    
    printf("UDP Server: Waiting for messages...\n");
    
    while (1) {
        client_len = sizeof(client_addr);
        
        nbytes = recvfrom(sock, buffer, sizeof(buffer), 0,
                         (struct sockaddr *)&client_addr,
                         &client_len);
        
        printf("Received: %.*s\n", nbytes, buffer);
        
        // 응답 전송
        sendto(sock, "ACK", 3, 0,
               (struct sockaddr *)&client_addr,
               client_len);
    }
    
    close(sock);
    return 0;
}
```

### UDP Client

```c
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

int main() {
    int sock;
    struct sockaddr_in addr;
    char message[] = "Hello UDP!";
    char buffer[256];
    int nbytes;
    
    sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(5555);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    
    sendto(sock, message, strlen(message), 0,
           (struct sockaddr *)&addr, sizeof(addr));
    
    nbytes = recvfrom(sock, buffer, sizeof(buffer), 0, NULL, NULL);
    printf("Received: %.*s\n", nbytes, buffer);
    
    close(sock);
    return 0;
}
```

---

## 6. Non-blocking 소켓

### Non-blocking 설정

```c
#include <fcntl.h>

int sock = socket(AF_INET, SOCK_STREAM, 0);

// Non-blocking 모드 설정
int flags = fcntl(sock, F_GETFL);
fcntl(sock, F_SETFL, flags | O_NONBLOCK);

// 또는
int opt = 1;
ioctl(sock, FIONBIO, &opt);
```

### Select를 통한 멀티플렉싱

```c
#include <sys/select.h>

int main() {
    int sock1, sock2;
    fd_set rfds;
    int max_fd;
    
    sock1 = socket(AF_INET, SOCK_STREAM, 0);
    sock2 = socket(AF_INET, SOCK_STREAM, 0);
    
    // ... bind, connect, listen ...
    
    while (1) {
        FD_ZERO(&rfds);
        FD_SET(sock1, &rfds);
        FD_SET(sock2, &rfds);
        max_fd = (sock1 > sock2) ? sock1 : sock2;
        
        int activity = select(max_fd + 1, &rfds, NULL, NULL, NULL);
        
        if (FD_ISSET(sock1, &rfds)) {
            printf("Activity on sock1\n");
        }
        if (FD_ISSET(sock2, &rfds)) {
            printf("Activity on sock2\n");
        }
    }
    
    return 0;
}
```

---

## 7. Socket 옵션 설정

### 중요한 Socket 옵션

```c
#include <sys/socket.h>

int sock = socket(AF_INET, SOCK_STREAM, 0);

// SO_REUSEADDR: 빠른 재바인딩
int reuse = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, 
           &reuse, sizeof(reuse));

// SO_KEEPALIVE: Keep-alive 활성화
int keepalive = 1;
setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, 
           &keepalive, sizeof(keepalive));

// TCP_NODELAY: Nagle 알고리즘 비활성화 (낮은 레이턴시)
int nodelay = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, 
           &nodelay, sizeof(nodelay));

// SO_SNDBUF: 송신 버퍼 크기
int send_buf = 65536;
setsockopt(sock, SOL_SOCKET, SO_SNDBUF, 
           &send_buf, sizeof(send_buf));

// SO_RCVBUF: 수신 버퍼 크기
int recv_buf = 65536;
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, 
           &recv_buf, sizeof(recv_buf));
```

---

## 8. 에러 처리

### 일반적인 에러 코드

```c
#include <errno.h>
#include <string.h>

if (connect(sock, ...) < 0) {
    switch (errno) {
    case ECONNREFUSED:
        fprintf(stderr, "Connection refused\n");
        break;
    case ETIMEDOUT:
        fprintf(stderr, "Connection timeout\n");
        break;
    case EHOSTUNREACH:
        fprintf(stderr, "Host unreachable\n");
        break;
    default:
        perror("connect");
    }
}
```

---

## 다음 단계

실시간 섹션의 `06_Realtime/_01_scheduler.md`에서 QNX 스케줄링을 학습합니다.
