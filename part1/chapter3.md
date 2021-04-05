## Chapter 3: 주소체계와 데이터 정렬



### Internet Address

- IPv4: 4바이트 주소체계

  - 네트워크 주소 + 호스트 주소
  - 클래스 
    - 첫 바이트 범위 A: 0~127, B:128~191, C: 192~223

  ![image-20210404232053251](C:\Users\Minjae\AppData\Roaming\Typora\typora-user-images\image-20210404232053251.png)

- IPv6: 16바이트 주소체계



### PORT 번호

- 최종 목적지(응용 프로그램)
- NIC(네트워크 인터페이스 카드) 수신 -> 운영체제 PORT 분배



### IPv4 기반의 주소표현을 위한 구조체

```c
struct sockaddr_in
{
    sa_family_t		sin_family;  	// 주소체계
    uint16_t		sin_port;		// 16비트 TCP/UDP PORT 번호
    struct in_addr 	sin_addr;		// 32비트 IP 주소
    char 			sin_zero[8];	// 사용되지 않음
}
```



### 네트워크 바이트 순서

- Big Endian 

  - 0x12345678 -> 0x12, 0x34, 0x56, 0x78

- Little Endian 

  - 0x12345678 -> 0x78, 0x56, 0x34, 0x12

- 네트워크 바이트 순서는 **Big Endian** 으로! 

- 바이트 변환

  ```c
  unsigned short htons(unsigned short);
  unsigned short ntohs(unsigned short);
  unsigned long htonl(unsigned long);
  unsigned long ntohl(unsigned long);
  ```



### 문자열을 네트워크 바이트 순서 정수로 변환

```c
#include <arpa/inet.h>
in_addr_t inet_addr(const char * string); // Big Endian 32비트 정수 변환
int inet_aton(const char * string, struct in_addr * addr);  // in_addr 구조체 사용
```



### 인터넷 주소의 초기화

```c
struct sockaddr_in addr;
char *serv_ip="211.217.168.13";				// IP 주소 
char *serv_port="9190";						// PORT 번호
memset(&addr, 0, sizeof(addr));				// 초기화
addr.sin_family=AF_INET;					// 주소체계
addr.sin_addr.s_addr=inet_addr(serv_ip);	// 문자열 IP주소 초기화: serv_ip -> INADDR_ANY
addr.sin_port=htons(atoi(serv_port));		// 문자열 PORT 번호 초기화
```

- 클라이언트 주소의 초기화 

  bind 대신 connect 함수 사용

### 소켓에 인터넷 주소 할당

```c
#include <sys.socket.h>

int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);
```



### 윈도우 소켓 인터넷 주소 할당

```c
#include <winsock2.h>

INT WSAStringToAddress( 	// 주소정보 문자열 -> 주소정보 구조체 변수
    LPTSTR AddressString, INT AddressFamily, LPWSAPROTOCOL_INFO lpProtocolInfo,
    LPSOCKADDR lpAddress, LPINT lpAddressLength
);

INT WSAAddressToString( 	// 주소정보 구조체 변수 -> 주소정보 문자열
    LPSOCKADDR lpsaAddress, DWORD dwAddressLength,
    LPWSAPROTOCOL_INFO lpProtocolInfo, LPTSTR lpszAddressString,
    LPDWORD lpdwAddressStringLength
);
```









