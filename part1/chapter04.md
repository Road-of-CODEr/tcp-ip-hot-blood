# Chapter 4 : TCP 기반 서버/클라이언트 1

## TCP/IP 프로토콜 스택

```
============================
|    Application 계층       |
============================
    ↕️                ↕️
===========      ===========
| TCP 계층 |      | UDP 계층 |
===========      ===========
        ↕️        ↕️
         ==========
         | IP 계층 |
         ==========
             ↕️
        ============
        | LINK 계층 |
        ============
                                      > TCP/IP 프로토콜 스택
```

데이터 송수신의 과정은 위 그림처럼 크게 네개의 영역으로 계층화되어 있다.

> OSI 7 계층
>
> 프로토콜 스택은 7계층으로 세분화되기도 하지만, 프로그래머의 관점에서는 4계층으로 이해하고 있어도 충분하다고 한다.

그럼 각 계층에 대해 하나씩 살펴보자.

### LINK 계층

LINK 계층은 물리적인 영역의 표준화이다. 즉, 통신이 가능하려면 물리적으로 호스트간의 연결이 되어 있어야 하는데 이 부분을 LINK 계층에서 담당하고 있다.

### IP 계층

물리적인 연결이 형성되어 있는 것에 기반하여 여러 경로중에 어떠한 경로로 가야할지 정하는 부분이 **IP(Internet Protocol) 계층**의 역할이다.

IP 프로토콜은 비 연결지향형, 신뢰할수 없는 프로토콜이다. 즉, 데이터가 오가는 와중에 중간에 손실되거나 왜곡되더라도 이에 대한 해결을 하지 않는다.

### TCP/UDP 계층 (Transport 계층)

IP 계층에서 정한 경로를 바탕으로 실제 데이터를 송수신을 담당하는 계층이다.

TCP의 경우에는 신뢰성 있는 데이터 전송을 담당한다. 즉, IP는 하나의 데이터 패킷이 전송되는 과정에만 중심을 두고 설계되었고, 각각 패킷의 순서나 오류에 대해서는 TCP가 담당하게 된다. - [TCP 데이터 송수신 과정](https://github.com/Road-of-CODEr/one-percent-network/blob/master/20201014/Chapter2-1.md#ack-%EB%B2%88%ED%98%B8%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%ED%8C%A8%ED%82%B7%EC%9D%B4-%EB%8F%84%EC%B0%A9%ED%96%88%EB%8A%94%EC%A7%80-%ED%99%95%EC%9D%B8%ED%95%9C%EB%8B%A4)

### Application 계층

위 3개의 계층은 `socket` 을 생성하면 데이터 송수신과정에서 자동으로 처리된다.

즉, 네트워크 프로그래밍에서는 이러한 과정을 신경쓰지 않아도 된다. socket을 이용해서 프로그램을 만들게 되면 클라이언트와 서버간의 데이터 송수신에 대한 규칙들이 생기는 데 이를 **APPLICATION 프로토콜**이라 부른다.

## TCP 기반 서버, 클라이언트 구현

### 서버

```
// TCP 서버의 함수 호출순서
socket() : 소켓 생성
  ⬇️
bind() : 소켓 주소할당
  ⬇️
listen() : 연결요청 대기상태
  ⬇️
accept() : 연결허용
  ⬇️
read()/write() : 데이터 송수신
  ⬇️
close() : 연결종료
```

#### listen()

```c
#include <sys/socket.h>

int listen(int sock, int backlog);  // 성공시 0, 실패시 -1
```

> sock - 연결 요청 대기 상태로 두고자 하는 소켓의 파일 스크립터
> backlog - 연결 요청 대기 큐의 크기

bind 함수를 통해 소켓에 주소할당한 후에 listen을 통해 서버는 클라이언트의 _연결 요청 대기 상태_ 로 들어가게 된다. 즉, listen 함수가 호출되어야 클라이언트는 connect 함수를 통해 서버에 연결 요청을 할 수 있다.

#### accept()

```c
#include <sys/socket.h>

int accept(int sock, struct sockaddr * addr, socklen_t * addrlen);
// 성공 시 생성된 소켓의 파일 디스크립터, 실패시 -1
```

> sock - 서버 소켓의 디스크립터
> addr - 연결요청 한 클라이언트의 주소 정보
> addrlen - 두 번째 매개변수 addr에 전달된 주소의 변수 크기

연결요청 대기중인 클라이언트의 연결 요청을 수락하는 함수이다.

- 데이터 입출력에 사용할 소켓 생성
- 연결요청한 클라이언트 소켓과의 연결

[hello_server.c 소스코드](https://github.com/Road-of-CODEr/tcp-ip-hot-blood/blob/master/codes/Chapter01%20%EC%86%8C%EC%8A%A4%EC%BD%94%EB%93%9C/hello_server.c)

### 클라이언트

```
// TCP 클라이언트 함수호출 순서
socket() : 소켓생성
  ⬇️
connect() : 연결요청
  ⬇️
read()/write() : 데이터 송수신
  ⬇️
close() : 연결종료
```

#### connect()

```c
#include <sys/socket.h>

int connect(int sock, struct sockaddr * servaddr, socklen_t addrlen); // 성공시 0, 실패시 -1
```

> sock - 클라이언트 소켓의 파일 디스크립터
> servaddr 서버의 주소정보
> addrlen - 서버 주소정보 변수의 크기를 바이트 단위로 전달

- 서버에 연결요청을 보낸다.

  - 서버에 의해 연결 요청 접수
    - 서버의 연결요청 대기 큐에 등록된 상황 (`accept()` 함수호출을 의미하는 것이 아니다)
    - 즉, connect 함수가 반환되더라도 당장에 서비스가 이뤄지지 않을 수 있다

- 클라이언트 소켓에 IP와 PORT 할당

[hello_client.c](https://github.com/Road-of-CODEr/tcp-ip-hot-blood/blob/master/codes/Chapter01%20%EC%86%8C%EC%8A%A4%EC%BD%94%EB%93%9C/hello_client.c)

### 에코 서버/클라

기본 동작방식

- 서버는 한번에 하나의 클라이언트와 연결되어 에코 서비스 제공
- 서버는 총 다섯개의 클라이언트에 순차적으로 서비스 제공
- 클라이언트는 문자열을 입력받아 서버에 전송
- 서버는 재전송
- 클라가 Q를 전송하면 연결 끊기

#### 서버

```
socket()
  ⬇️
bind()
  ⬇️
listen()
  ⬇️
accept()
  ⬇️
read()/write()
  ⬇️
close(client)
  ⬇️
close(server)
```

[소스코드](https://github.com/Road-of-CODEr/tcp-ip-hot-blood/tree/master/codes/Chapter04%20%EC%86%8C%EC%8A%A4%EC%BD%94%EB%93%9C)
