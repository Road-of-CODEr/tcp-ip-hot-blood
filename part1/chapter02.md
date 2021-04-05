# 소켓의 타입과 프로토콜의 설정

## 프로토콜이란 무엇인가?

컴퓨터 상호간의 대화에 필요한 통신규약

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol); // 성공 시 파일 디스크립터, 실패 시 -1 반환
// E.X
serv_sock = socket(PF_INET, SOCK_STREAM, 0);
// IPv4, 열결지향형, TCP 소켓.
```

domain: 소켓이 사용할 프로토콜 체계 정보 전달

type: 소켓의 데이터 전송방식에 대한 정보 전달

protocol: 프로토콜 정보 전달

### Domain - 프로토콜 체계

PF_INET: IPv4

PF_INET6: IPv6

PF_LOCAL: 로컬 통신을 위한 UNIX 프로토콜

PF_PACKET: Low level 소켓을 위한 프로토콜

PF_IPX: IPX 노벨 프로토콜

실제 소켓이 사용할 최종 프로토콜 정보는 세 번째 인자인 `protocol` 에 존재한다. 단, 첫 번째 인자를 통해 지정한 프로토콜 체계 범위 내의 프로토콜만 지정할 수 있다.

> RFC 460에서 정의된 IPv6(IP 버전 6)는 IETF(Internet Engineering Task Force)로 정의되는 최신 세대의 인터넷 프로토콜(IP)입니다. 최초의 안정적인 인터넷 프로토콜(IP) 버전은 IPv4(IP 버전 4)였습니다. IPv6는 최종적으로 IPv4를 대체하기 위한 것이지만, 현재는 둘 모두 결합되어 사용되고 있습니다. 대부분의 엔지니어들은 두 가지를 함께 실행하고 있습니다.

> ref: https://www.juniper.net/kr/kr/products-services/what-is/ipv4-vs-ipv6/

### 소켓의 타입

소켓의 타입은 소켓의 데이터 전송방식을 의미한다.

1. 연결 지향형 소켓(SOCK_STREAM)
   - 연결 지향형 소켓은 자신과 연결된 상대 소켓의 상태를 파악해 가면서 데이터를 전송한다.
   - 그러므로 버퍼를 가득 채워도 안전하게 처리하며 제대로 전송되지 않을 경우 재전송도 한다.
   - 따라서 전송되는 데이터의 경계(Boundary)가 존재하지 않는다.(`read`, `write` 함수의 호출 횟수가 서로 다르다)
   - 연결 지향형 소켓은 반드시 1:1 연결이다.
2. 비 연결 지향형 소켓(SOCK_DGRAM)
   - 전송된 순서에 상관없이 가장 빠른 전송을 지향한다.
   - 데이터 손실 및 파손 우려가 있다.
   - 전송되는 데이터의 경계(Boundary)가 존재한다.
   - 한번에 전송할 수 있는 데이터의 크기가 제한된다.

### 프로토콜

`socket` 함수의 세 번째 인자인 프로토콜은 최종적으로 소켓이 사용하게 될 프로토콜 정보를 전달하는 목적으로 존재한다.

첫 번째와 두 번째 인자만으로도 원하는 유형의 소켓을 생성할 수 있다. 하지만 **하나의 프로토콜 체계 안에 데이터의 전송방식이 동일한 프로토콜이 둘 이상 존재하는 경우** 세 번째 인자가 필요하다.

즉, 소켓의 데이터 전송방식은 같지만, 그 안에서 프로토콜이 다시 나뉘는 상황이다.

TCP: `IPPROTO_TCP`

UDP: `IPPROTO_UDP`

### 예제 코드

```c
#include <stdio.h>
#include <stdlib.h>
#include <winsock2.h>
void ErrorHandling(char* message);

int main(int argc, char* argv[])
{
	WSADATA wsaData;
	SOCKET hSocket;
	SOCKADDR_IN servAddr;

	char message[30];
	int strLen=0;
	int idx=0, readLen=0;

	if(argc!=3)
	{
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}

	if(WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
		ErrorHandling("WSAStartup() error!");

	hSocket=socket(PF_INET, SOCK_STREAM, 0);
	if(hSocket==INVALID_SOCKET)
		ErrorHandling("hSocket() error");

	memset(&servAddr, 0, sizeof(servAddr));
	servAddr.sin_family=AF_INET;
	servAddr.sin_addr.s_addr=inet_addr(argv[1]);
	servAddr.sin_port=htons(atoi(argv[2]));

	if(connect(hSocket, (SOCKADDR*)&servAddr, sizeof(servAddr))==SOCKET_ERROR)
		ErrorHandling("connect() error!");

	while(readLen=recv(hSocket, &message[idx++], 1, 0))
	{
		if(readLen==-1)
			ErrorHandling("read() error!");

		strLen+=readLen;
	}

	printf("Message from server: %s \n", message);
	printf("Function read call count: %d \n", strLen);

	closesocket(hSocket);
	WSACleanup();
	return 0;
}

void ErrorHandling(char* message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```
