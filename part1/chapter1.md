## chapter 1: 네트워크 프로그래밍과 소켓의 이해

![network programming](/assets/network_programming.jpg)

### Network Programming
- 소켓 기반의 프로그래밍
- 네트워크로 연결된 컴퓨터 사이의 데이터 송수신 프로그램 작성


### Socket
- 네트워크의 연결도구
- OS 가 제공하는 Software
- 데이터 송수신에 대한 물리적, 소프트웨어적 세세한 내용을 신경 쓰지말자


![network programming](/assets/network_programming2.jpg)

### server 소켓(listening socket)
1. phone 구입
2. 전화번호를 할당
3. 전화기가 수신가능 한 상태
4. 통화버튼을 누른다


#### phone 을 산다: 소켓을 생성한다 
- socket 함수 호출
- 수신과 송신할 때 소켓 생성 방법 차이가 있음
- 파일 디스크립터 리턴

```cpp
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

#### 개통: 주소 할당
- 고유한 번호(IP:Port)가 있어야 구분이 되겠구나
- bind 함수를 호출해서 번호를 할당받자(주소 할당)

```cpp
#include <sys/socket.h>
int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);
```

#### 전화기 수신가능한 상태
- 전화기를 전화망에 연결하여 전화를 받을 수 있는 상태로하자

```cpp
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

#### 통화버튼을 누른다
- 누군가 전화를 걸었을 때 통화버튼을 눌러야 한다.
- 통화연결이 됐으면 송수신은 양방향으로 가능함

```cpp
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, soclen_t *addrlen);
```

### 결론
socket -> bind -> listen -> accept

### client 소켓
- 소켓 생성, 주소 할당 과정 자동화

```cpp
#include <sys/socket.h>
int connect(int sockfd, strucy sockaddr *serv_addr, socklen_t addlen);
```


### Linux 기반 File I/O 


![network programming](/assets/network_programming3.jpg)
1. 소켓은 file 로 간주된다. 
2. C 언어상의 표준 -> OS 라이브러리(운영체제가 할수있는 모든 일들; system 함수들, winAPI)
- 리눅스를 만들 때 c 표준 함수를 만들지않아..
- 컴파일러가 라이브러리를 이용해서 윈도우 or 리눅스에서 동작하는 바이트코드를 만들어내는 것


