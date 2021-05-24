# Chapter 13. 다양한 입출력 함수들

## 리눅스에서의 send & recv

```c
#include <sys/socket.h>
ssize_t send(int sockfd, const void * buf, size_t nbytes, int flags); 
// 성공 시 전송된 바이트 수, 실패시 -1
```

> - sockfd - 데이터 전송 대상과의 연결을 의미하는 소켓읠 파일 디스크립터 전달
> - buf - 전송할 데이터를 저장하고 있는 버퍼의 주소 값 전달
> - nbytes - 전송할 바이트 수 전달
> - flags - 데이터 전송시 적용할 다양한 옵션 정보 전달

```c
# include <sys/socket.h>
ssize_t recv(int sockfd, void * buf, size_t nbytes, int flags);
// 성공 시 수신한 바이트 수, 실패시 -1
```

> - sockfd - 데이터 수신 대상과의 연결을 의미하는 소켓의 파일 디스크립터 전달
> - buf - 수신된 데이터를 저장할 버퍼의 주소 값 전달
> - nbytes - 수신할 수 있는 최대 바이트 수 전달
> - flags - 데이터 수신 시 적용할 다양한 옵션 정보 전달



### send & recv의 옵션

| Option        | 의미                                                         | send | recv |
| ------------- | ------------------------------------------------------------ | ---- | ---- |
| MSG_OOB       | 긴급 데이터(Out-of-band data)의 전송을 위한 옵션             | ✅    | ✅    |
| MSG_PEEK      | 입력버퍼에 수신된 데이터의 존재유무 확인을 위한 옵션         |      | ✅    |
| MSG_DONTROUTE | 데이터 전송과정에서 라우팅 테이블을 참조하지 않을 것을 요구하는 옵션 - 로컬 네트워크 | ✅    |      |
| MSG_DONTWAIT  | 입출력 함수 호출과정에서 블로킹 되지 않을 것을 요구하기 위한 옵션 | ✅    | ✅    |
| MSG_WAITALL   | 요청한 바이트 수에 해당하는 데이터가 전부 수신될 때까지, 호출된 함수가 반환되는 것을 막기 위한 옵션 |      | ✅    |

- 위 옵션의 지원여부는 운영체제마다 조금씩 차이가 날수 있다 - 운영체제의 이해 필요
- 운영체제에 따라서 차이를 보이지 않는 옵션들 위주로 살펴보자.

#### MSG_OOB: 긴급 메시지 전송

*비유를 통해 쉽게 알아보자*

병원에 응급환자가 들어오면, 다른 환자들 보다 우선 순위로 처리를 해주어야 한다. 그리고 응급환자들을 처리해줄수 있는 창구인 응급실이 필요하다.

즉, MSG_OOB 옵션은 긴급으로 전송해야 할 메시지가 있어 메시지의 전송방법 및 경로를 달리하고자 할 때 사용한다.

```c
// oob_send.c

// 생략...

int main(int argc, char *argv[])
{
  int sock;
  // 중략 ...

  write(sock, "123", strlen("123"));
  send(sock ,"4", strlen("4"), MSG_OOB);
  write(sock, "567", strlen("567"));
  send(sock, "890", strlen("890"), MSG_OOB);
  close(sock);
  return 0;
}
	
// 생략...
```

```c
// oob_recv.c

// 생략...

int main(int argc, char *argv[])
{
	struct sockaddr_in recv_adr, serv_adr;
	int str_len, state;
	socklen_t serv_adr_sz;
	struct sigaction act;
	char buf[BUF_SIZE];
  
	// 중략...
	
	act.sa_handler=urg_handler;
	sigemptyset(&act.sa_mask);
	act.sa_flags=0; 
	
	// 중략 ...
	
	fcntl(recv_sock, F_SETOWN, getpid()); 
	state=sigaction(SIGURG, &act, 0);
	
	while((str_len=recv(recv_sock, buf, sizeof(buf), 0))!= 0) 
	{
		if(str_len==-1)
			continue;
		buf[str_len]=0;
		puts(buf);
	}
	close(recv_sock);
	close(acpt_sock);
	return 0; 
}

void urg_handler(int signo)
{
	int str_len;
	char buf[BUF_SIZE];
	str_len=recv(recv_sock, buf, sizeof(buf)-1, MSG_OOB);
	buf[str_len]=0;
	printf("Urgent message: %s \n", buf);
}

// 생략...
```

> 소스코드의 전문을 원한다면 [여기](../codes/Chapter13%20소스코드/)에서 확인할 수 있다.

- `fcntl(rev_sock, F_SETOWN, getpid()); `  - *fcntl 함수에 대한 자세한 설명은 17장에서...*
  - 위 함수는 파일 디스크립터의 컨트롤에 사용
  - rev_sock 이 가리키는 소켓의 소유자(F_SETOWN)를  getpid 함수가 반환하는 ID의 프로세스로 변경하겠다
  - `F_SETOWN` : SIGIO와 SIGURG신호를 받도록 시정된 프로세스 ID나 프로세스 그룹 ID를 설정
  - **즉, `recv_sock` 이 가리키는 소켓에서 발생하는 긴급처리 시그널을 처리하는 프로세스를 현재 실행중인 프로세스로 변경하겠다.** 
  - why?
    - 하나의 소켓에 대한 파일 디스크립터를 여러 프로세스가 함께 소유할 수 있다
    - SIGURG 시그널 발생시 어느 프로세스의 핸들러 함수를 호출해야할지 반드시 지정해주어야 한다.



##### Urgent mode

>  데이터의 전송순서는 유지되지만, 데이터의 처리를 독촉한다. 

병원에 비유를 하자면 응급환자가 발생했을 때 두가지 조건을 만족시켜야한다.

- 병원으로의 빠른 이동
- 병원에서의 빠른 응급조치

즉, TCP 의 긴급 메시지는 기본적으로 TCP의 전송특성이 유지되기 때문에 전송 순서는 유지되지만, **빠른 처리를 하도록 독촉**하는 역할을 합니다.





![TCP-headers](http://www.ktword.co.kr/img_data/1889_1.JPG)

```
|| URG=1, URG Pointer=3 || 8 | 9 | 0 ||
<---------TCP 헤더-------><----데이터---->
```

> "Urgent Pointer가 가리키는 오프셋 3의 바로 앞에 존재하는 것이 긴급 메시지다!"

긴급 메시지는 메시지 처리를 재촉하는데 의미가 있는 것이지 제한된 형태의 메시지를 긴급으로 전송하는 데 의미가 있는 것은 아니다



#### MSG_PEEK, MSG_DONTWAIT

```c
// peek_recv.c

// 생략 ...

int main(int argc, char *argv[])
{
	// 생략 ...
  
	while(1)
	{
		str_len=recv(recv_sock, buf, sizeof(buf)-1, MSG_PEEK|MSG_DONTWAIT);
		if(str_len>0)
			break;
	}

	buf[str_len]=0;
	printf("Buffering %d bytes: %s \n", str_len, buf);
 	
	str_len=recv(recv_sock, buf, sizeof(buf)-1, 0);
	buf[str_len]=0;
	printf("Read again: %s \n", buf);
	close(acpt_sock);
	close(recv_sock);
	return 0; 
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

- 데이터가 존재하지 않아도 블로킹 상태로 두지 않기 위해 `MSG_DONTWAIT` 옵션을 같이 준다
- `MSG_PEEK` : 입력 버퍼에 존재하는 데이터가 읽혀지더라도 입력버퍼에서 데이터가 지워지지 않는다. 즉, 데이터의 존재유무를 확인하기 위한 용도이다.
- 따라서, 입력 버퍼에서 데이터를 읽어들이기 위해 recv 를 한번 더 호출해준다.



## readv & writev

> 데이터를 모아서 전송, 수신



### writev

```c
#include <sys/uio.h>
ssize_t writev(int filedes, const struct iovec * iov, int iovcnt);
// 리턴값 : 전송된 바이트 수 or -1
```

> - filedes: 데이터 전송의 목적지를 나타내는 파일 디스크립터 전달
> - iov : 구조체 iovec 배열의 주소 값 전달
> - iovcnt : iov 배열의 길이 정보 전달

### readv

```c
#include <sys/uid.h>
ssize_t readv(int filedes, const struct iovec * iov, int iovcnt);
// 리턴값 : 수신된 바이트 수 or -1
```

> - filedes : 데이터를 수신할 파일 디스크립터
> - iov : 데이터를 저장할 위치와 크기 정보를 담고 있는 iovec 구조체 배열의 주소 값 전달
> - iovcnt : iov 배열의 크기 정보

#### iovec 구조체

```c
struct iovec
{
  void * iov_base; // 버퍼의 주소 정보
  size_t iov_len; // 버퍼의 크기 정보
}
```

```
// example

writev(1, ptr, 2);

| ptr | -> | iov_base  | -> | A | B | C |
           | iov_len=3 |
           
           | iov_base  | -> | 1 | 2 | 3 | 4 |
           | iov_len=4 |
           
```

### 

```c
// writev.c
#include <stdio.h>
#include <sys/uio.h>

int main(int argc, char *argv[])
{
	struct iovec vec[2];
	char buf1[]="ABCDEFG";
	char buf2[]="1234567";
	int str_len;

	vec[0].iov_base=buf1;
	vec[0].iov_len=3;
	vec[1].iov_base=buf2;
	vec[1].iov_len=4;
	
	str_len=writev(1, vec, 2);
	puts("");
	printf("Write bytes: %d \n", str_len);
	return 0;
}

// readv.c
#include <stdio.h>
#include <sys/uio.h>
#define BUF_SIZE 100

int main(int argc, char *argv[])
{
	struct iovec vec[2];
	char buf1[BUF_SIZE]={0,};
	char buf2[BUF_SIZE]={0,};
	int str_len;

	vec[0].iov_base=buf1;
	vec[0].iov_len=5;
	vec[1].iov_base=buf2;
	vec[1].iov_len=BUF_SIZE;

	str_len=readv(0, vec, 2);
	printf("Read bytes: %d \n", str_len);
	printf("First message: %s \n", buf1);
	printf("Second message: %s \n", buf2);
	return 0;
}
```







