# 7장 - 소켓의 우아한 연결종료

## TCP 기반의 Half-close

### 일반적인 연결종료의 문제점

<p align="center">
<img width="500" src="https://user-images.githubusercontent.com/47745785/115983523-86ed0300-a5dc-11eb-8d36-e9f53d5e20e4.png" alt="일방적 연결종료">
</p>

* <b>지금까지 TCP 예제들은 close 함수들을 한쪽에서 일방적으로 호출함. (ex: 클라이언트에서 Q를 입력할 경우 무한 루프문을 빠져나오고 바로 close 함수 호출. )close 함수에서 Four-way handshaking을 하지만 사실은 일방적으로 close를 호출하기 때문에 우아하지 않은 종료임. </b>
* 호스트 A가 마지막 데이터를 전송하고 나서 close 함수의 호출을 통해서 연결을 종료.
* 그 이후부터 호스트 A는 호스트 B가 전송하는 데이터를 수신하지 못함. 

### 소켓과 스트림

<p align="center">
<img width="400" src="https://user-images.githubusercontent.com/47745785/115983625-45a92300-a5dd-11eb-9f31-26b3de03b426.png" alt="일방적 연결종료">
</p>

* 소켓을 통해서 두 호스트가 연결되서 상호간에 데이터의 송수신이 가능한 상태가 되면 '스트림이 형성된 상태'라 함. 
* 소켓의 스트림은 한쪽 방향으로만 데이터의 이동이 가능하기 때문에 양방향 통신을 위해서는 2개의 스트림이 필요. (물의 흐름과 비슷)
* 우아한 종료라는 것은 한번에 이 두 스트림을 모두 끊지 않고, 이 중 하나의 스트림만 끊는 것. 

### 우아한 종료를 위한 shutdown 함수

```c++
# include <sys/socket.h>

int shutdown(int sock, int howto);
/*
성공 시 0, 실패 시 -1
첫 번째 인자 sock에 종료할 소켓의 파일 디스크립터 전달.
두 번째 인자 howto에 종료방법에 대한 정보 전달. 
*/
```

* shutdown 함수의 두 번째 매개변수에 전달될 수 있는 인자의 종류
  * `SHUT_RD` : 입력 스트림 종료
  * `SHUT_WR` : 출력 스트림 종료
  * `SHUT_RDWR` : 입출력 스트림 종료
* SHUT_WR를 써서 출력 스트림이 종료되었지만, 출력 버퍼에 아직 전송되지 못한 상태로 남아 있는 데이터가 존재하면 해당 데이터는 목적지로 전송. 

### Half-close가 필요한 이유

> "클라이언트가 서버에 접속하면 서버는 약속된 파일을 클라이언트에게 전송하고, 클라이언트는 파일을 잘 수신했다는 의미로 문자열 "Thank you"를 서버에 전송"

* 충분히 여유를 두고 close 함수를 호출해줘서 송수신을 완료할 수도 있지만, 만약에 연결 종료 직전에 클라이언트가 서버에 전송해야 할 데이터가 존재하는 상황이 있다면 close 종료 함수 호출을 하기가 애매해짐. 

<br> 

* 하지만, 이 상황에 대한 프로그램의 구현도 힘든 부분이 있음. 파일을 전송하는 서버는 단순히 파일 데이터를 연속해서 전송하면 되지만, 클라이언트가 데이터의 수신의 끝을 모름. 
* 이러한 문제의 해결을 위해서 서버는 파일의 전송이 끝났음을 알리는 목적으로 EOF(End of file)를 마지막에 전송해야 함. 
* 클라이언트는 EOF의 수신을 데이터안이 아닌 함수의 반환 값을 통해서 확인이 가능하기 떄문에 저장된 데이터와 중복될 일도 없음.
* <b>`shutdown` 함수에 `SHUT_WR`인자를 줘서 호출해 출력 스트림을 종료하면 상대 호스트로 EOF가 전송된다. </b>
* close 함수호출을 통해서 입출력 스트림을 모두 종료해줘도 EOF는 전송되지만, 이럴 경우 상대방이 전송하는 데이터를 더 이상 수신 못한다는 문제가 있음. 

### Half-close 기반의 파일전송 프로그램

<p align="center">
<img width="500" src="https://user-images.githubusercontent.com/47745785/115983752-1b0b9a00-a5de-11eb-8bfd-3d6660305e80.png" alt="일방적 연결종료">
</p>

<b>서버</b>

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int serv_sd, clnt_sd;
	FILE * fp;
	char buf[BUF_SIZE];
	int read_cnt;
	
	struct sockaddr_in serv_adr, clnt_adr;
	socklen_t clnt_adr_sz;
	
	if(argc!=2) {
		printf("Usage: %s <port>\n", argv[0]);
		exit(1);
	}
	
	fp=fopen("file_server.c", "rb"); // ✅ 서버의 소스파일을 클라이언트한테 전송.
	if(fp == NULL)
	    error_handling("fopen() error");
	
	serv_sd=socket(PF_INET, SOCK_STREAM, 0);   
	if(serv_sd == -1)
	    error_handling("socket() error");
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(serv_sd, (struct sockaddr*)&serv_adr, sizeof(serv_adr)) == -1)
	    error_handling("bind() error");

	if(listen(serv_sd, 5) == -1)
	    error_handling("listen() error");
	
	clnt_adr_sz=sizeof(clnt_adr);    
	clnt_sd=accept(serv_sd, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
	if(clnt_sd == -1)
	    error_handling("accept() error");
		
	while(1)
	{
    // ✅ BUF_SIZE만큼 계속 읽어들이다가 마지막에 읽어들인 read_cnt가 BUF_SIZE보다 작으면 나머지를 보내고 무한루프를 빠져나옴.
		read_cnt=fread((void*)buf, 1, BUF_SIZE, fp);
		if(read_cnt<BUF_SIZE)
		{
			write(clnt_sd, buf, read_cnt);
			break;
		}
		write(clnt_sd, buf, BUF_SIZE);
	}
	
  /*✅ 출력 스트림에 대해 Half-close를 진행. 
  이 때 클라이언트로 파일의 끝을 알려주는 EOF가 전송되고, 클라이언트가 전송이 완료됨을 알고 'Thank you'를 보냄. 
  출력 스트림만 닫았으므로 입력 스트림으로 read 함수를 통해 'Thank you' 수신 가능. */
	shutdown(clnt_sd, SHUT_WR);	
	read(clnt_sd, buf, BUF_SIZE);
	printf("Message from client: %s \n", buf);
	
	fclose(fp);
	close(clnt_sd); close(serv_sd);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

<b>클라이언트</b>

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sd;
	FILE *fp;
	
	char buf[BUF_SIZE];
	int read_cnt;
	struct sockaddr_in serv_adr;
	if(argc!=3) {

		printf("Usage: %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	fp=fopen("receive.dat", "wb"); // ✅ 서버에서 전송 받을 데이터를 담을 파일을 생성
	if(fp == NULL)
	    error_handling("fopen() error");

	sd=socket(PF_INET, SOCK_STREAM, 0);   
	if(sd == -1)
	    error_handling("socket() error");

	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=inet_addr(argv[1]);
	serv_adr.sin_port=htons(atoi(argv[2]));

	if(connect(sd, (struct sockaddr*)&serv_adr, sizeof(serv_adr)) == -1)
	    error_handling("connect() error");
	
  // ✅ EOF가 전송될 때까지 데이터를 수신한 다음, 파일에 데이터를 씀. 
	while((read_cnt=read(sd, buf, BUF_SIZE ))!=0)
		fwrite((void*)buf, 1, read_cnt, fp);
	2
	puts("Received file data");
	write(sd, "Thank you", 10);
	fclose(fp);
	close(sd);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```