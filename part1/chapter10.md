# 10장 - 멀티프로세스 기반의 서버구현

## 프로세스의 이해와 활용

* 지금까지 저희가 공부한 소켓통신은 서버의 listen() 함수 인자 값을 통해서 여러개의 클라이언트의 연결 요청을 받을 수 있지만, 클라이언트가 동시에 연결되는것이 아닌 순차적으로 한개씩만 연결이 가능했다. 
* 이번 단원에서는 동시에 여러개의 클라이언트가 서버에 접속할 수 있는 방법에 대해 논의하고 있다. 

### 다중접속 서버의 구현방법들

다중접속 서버를 왜 구현해야 할까?

* 네트워크 프로그램은 데이터의 송수신 시간이 큰 비중을 차지하고, 만약 클라이언트가 1개라면 데이터 송수신 시간에는 cpu는 놀게 된다. 따라서 cpu가 노는 시간에 다른 클라이언트가 cpu를 쓸 수 있게 되면 즉, 둘 이상의 클라이언트에게 동시에 서비스를 제공하는 것이 효율적이다. 

다중 접속 서버 구현 방법

* 멀티프로세스 기반 서버 : 다수의 프로세스를 생성하는 방식으로 서비스 제공(이번 단원, Window에서는 이 방식 지원 안함)
* 멀티플렉싱 기반 서버 : 입출력 대상을 묶어서 관리하는 방식으로 서비스 제공(12단원)
* 멀티쓰레딩 기반 서버 : 클라이언트의 수만큼 쓰레드를 생성하는 방식으로 서비스 제공(18단원)

### 프로세스의 이해 

> 메모리 공간을 차지한 상태에서 실행중인 프로그램

ex) 벽돌 깨기 게임은 프로그램, 게임을 실행하면 프로세스라고 부를 수 있다. 

* 프로세스는 운영체제의 관점에서 프로그램 흐름의 기본 단위가 되며, 여러 개의 프로세스가 생성되면 이들은 동시에 실행이 된다. <b>그러나 하나의 프로그램이 실행되는 과정에서 여러 개의 프로세스가 생성되기도 한다. 지금부터 우리가 구현할 멀티 프로세스 기반의 서버가 대표적인 예이다.</b>
* 참고 : 두 개의 연산장치(사람 2명)가 존재하는 CPU를 가리켜 듀얼 코어 CPU, 4개의 연산장치(사람 4명)가 존재하는 CPU를 가리켜 쿼드 코어 CPU라 한다. 코어의 수만큼 프로세스가 동시 실행이 가능하다. 반면 코어의 수를 넘어서는 개수의 프로세스가 생성되면, 프로세스 별로 코어에 할당되는 시간이 나뉘게 된다. 그러나 고속으로 프로세스를 실행하기 떄문에 우리는 모든 프로세스가 동시에 실행되는 것처럼 느끼게 된다. 

### 프로세스 ID 

* 모든 프로세스는 운영체제로부터 ID를 부여 받는데 이를 가리켜 '프로세스 ID'라 한다. 이 프로세스 ID는 2 이상의 정수 형태를 띠는데 숫자 1은 운영체제가 시작되자마자 실행되는 프로세스에게 할당되기 때문에 우리가 만들어 내는 프로세스는 1이라는 값의 ID를 받을 수 없다. 

### fork 함수호출을 통한 프로세스의 생성(멀티 프로세스 기반)

```c++
#include <unistd.h>

pid_t fork(void);

// 성공 시 자식 프로세스 ID, 실패 시 -1 반환
```

* fork 함수는 호출한 프로세스의 복사본을 생성한다. 
* 이렇게 생성된 2개의 프로세스는 fork 함수가 끝나고 반환 이후 문장을 실행하게 된다. 
* 이 두 프로세스는 완전히 동일한 프로세스로, 메모리 영역까지 동일하게 복사하기 때문에 이후의 프로그램 흐름은 fork 함수의 반환 값을 기준으로 나뉘도록 프로그램을 짜야한다. 
  * 부모 프로세스 : fork 함수의 반환 값은 자식 프로세스의 ID
  * 자식 프로세스 : fork 함수의 반환 값은 0
* 부모 프로세스는 원본 프로세스, 즉 fork 함수를 호출한 주체이고, 자식 프로세스는 부모 프로세스의 fork 함수 호출을 통해서 복사된 프로세스를 의미한다. 

```c++
// fork.c

#include <stdio.h>
#include <unistd.h>
int gval=10;

int main(int argc, char *argv[])
{
	pid_t pid;
	int lval=20;
	gval++, lval+=5; // 11, 25
	
	pid=fork();		
	if(pid==0)	// if Child Process
		gval+=2, lval+=2;
	else			// if Parent Process
		gval-=2, lval-=2;
	
	if(pid==0)
		printf("Child Proc: [%d, %d] \n", gval, lval);
	else
		printf("Parent Proc: [%d, %d] \n", gval, lval);
	return 0;
}
/*
13, 27

9 23
*/
```

## 프로세스 & 좀비(Zombie) 프로세스

* 프로세스가 생성되고 나서 할 일을 다 하면 사라져야 하는데 사라지지 않고 좀비가 되어 시스템의 중요한 리소스를 차지하기도 하는데 이 상태에 있는 프로세스를 가리켜 <b>'좀비 프로세스'</b>라 한다. 

### 좀비 프로세스의 생성이유

<b>fork 함수의 호출로 생성된 자식 프로세스가 종료되는 상황 2가지 </b>

* 자식 프로세스 코드 구간 안에서 인자를 전달하면서 exit를 호출하는 경우
* main 함수에서 return문을 실행하면서 값을 반환하는 경우(자식 프로세스 코드 구간 안에서 return 호출)

<br>

* <b>자식 프로세스 내에서 exit이나 return이 없으면 좀비 프로세스 생성된다. </b>

```c++
// zombie.c

#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
	pid_t pid=fork();
	
	if(pid==0)     // if Child Process
	{ 	
		puts("Hi I'am a child process");
	}
	else
	{
		printf("Child Process ID: %d \n", pid);
		sleep(30);     // Sleep 30 sec.
	}

	if(pid==0)
		puts("End child process");
	else
		puts("End parent process");
	return 0;
}

/*
Hi, I am a child process 
End child process
Child Process ID: 10977
*/
```

### 좀비 프로세스의 소멸1: wait 함수의 사용

* <b>부모 프로세스가 자식 프로세스가 종료될 때 까지 대기를 한다. </b>
* 자식 프로세스가 정상 종료 여부, 반환 값을 알 수 있음

```c++
#include <sys.wait.h> 

pid_t wait(int * statloc); 
// 성공시 종료된 자식 프로세스의 ID, 실패 시 -1 반환
```

* 위 함수가 호출되었을 때, 이미 종료된 자식 프로세스가 있다면, 자식 프로세스가 종료되면서 전달한 값(exit 함수의 인자 값, return에 의한 반환 값)이 매개변수로 전달된 주소의 변수에 저장된다. 
* <b>그런데 이 변수에 저장되는 값에는 자식 프로세스가 종료되면서 전달한 값 이외에도 다른 정보가 함께 포함되어 있어서, 다음의 매크로 함수를 통해서 값의 분리 과정을 거쳐야 한다. </b>
  * **WIFEXITED** : 자식 프로세스가 정상 종료한 경우 '참(true)'을 반환한다. 
  * **WEXITSTATUS** : 자식 프로세스의 전달 값을 반환한다. 

```c++
if(WIFEXITED(status)) // 정상 종료가 되었다면
{
	puts("Normal termination!");
	printf("Child pass num: %d", WEXITSTATUS(status)); // 반환 값 확인
}
```

```c++
// wait.c

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
	int status;
	pid_t pid=fork();
	
	if(pid==0)
	{
		return 3;   	
	}
	else
	{
		printf("Child PID: %d \n", pid);
		pid=fork();
		if(pid==0)
		{
			exit(7);
		}
		else
		{
			printf("Child PID: %d \n", pid);
			wait(&status);
			if(WIFEXITED(status))
				printf("Child send one: %d \n", WEXITSTATUS(status));

			wait(&status);
			if(WIFEXITED(status))
				printf("Child send two: %d \n", WEXITSTATUS(status));
			sleep(30);     // Sleep 30 sec.
		}
	}
	return 0;
}

/*
Child PID: 12337
Child PID: 12338
Child send one: 3	
Child send two: 7
*/
```

### 좀비 프로세스의 소멸2: waitpid 함수의 사용

* <b>wait 함수는 자식 프로세스가 종료될 때 까지 대기를 하기 때문에 블로킹이 문제가 된다. 따라서 블로킹 문제를 해결하려면 waitpid 함수를 사용하면 된다. </b>

```c++
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int * statloc, int options);
/*
성공시 종료된 자식 프로세스 ID, 실패 시 -1 반환. 
pid에 종료를 확인하고자 하는 자식 프로세스 ID 전달. -1을 전달하면 wait 함수와 마찬가지로 임의의 자식 프로세스가 종료되기를 기다린다. 

statloc는 상태 값을 저장하는 포인터 변수. 매크로 함수로 분리해야함. 

options은 WNOHANG을 인자로 전달하면, 종료된 자식 프로세스가 존재하지 않아도 블로킹 상태에 있지 않고, 0을 반환하면서 함수를 빠져 나온다. 
*/
```

```c++
// waitpid.c

#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
	int status;
	pid_t pid=fork();
	
	if(pid==0)
	{
		sleep(15);
		return 24;   	
	}
	else
	{
		while(!waitpid(-1, &status, WNOHANG))
		{
			sleep(3);
			puts("sleep 3sec.");
		}

		if(WIFEXITED(status))
			printf("Child send %d \n", WEXITSTATUS(status));
	}
	return 0;
}

/*
sleep 3sec.
sleep 3sec.
sleep 3sec.
sleep 3sec.
sleep 3sec.
Child send 24 
*/
```

## 시그널 핸들링

* 자식 프로세스가 언제 종료될지 모르는 상태에서 waitpid 함수만 호출하고 있기에는 너무 비효율적이다. 
* 자식 프로세스의 종료를 기다리면서 waitpid 함수만 호출할 수는 없다. 

### 운영체제야! 네가 좀 알려줘

* 자식 프로세스 종료의 인식주체는 운영체제이다. 따라서 운영체제가 열심히 일하고 있는 부모 프로세스에게 자식 프로세스가 종료 되었다고 이야기해줄 수 있다면 효율적인 구현이 가능하다. 
* 시그널 핸들링 : 시그널은 특정상황이 발생했음을 알리기 위해 운영체제가 프로세스에게 전달하는 메시지를 의미하고, 메시지에 반응해서 미리 정의된 작업이 진행되는 것을 가리켜 핸들링이라 한다. 

### 시그널 등록 함수

* 프로세스가 자식 프로세스의 종료 상황 발생시, 특정 함수의 호출을 운영체제에게 요구하는 것. 

```c++
#include <signal.h>

void (*signal(int signo, void (*func)(int)))(int); // 반환형이 함수 포인터

/*
함수 이름  :signal
매개변수 선언 : int signo, void(*func)(int)
반환형 : 매개변수형이 int이고 반환형이 void인 함수 포인터
*/
```

* 첫 번째 인자로 특정 상황에 대한 정보를, 두 번째 인자로 특정 상황에서 호출될 함수의 주소 값(포인터)을 전달한다. 
* 그러면 첫 번째 인자를 통해 명시된 상황 발생시, 두 번째 인자로 전달된 주소 값의 함수가 호출된다. 

<br> 

* signal 함수를 통해서 등록 가능한 특정 상황과 그 상황에 할당된 상수들
  * SIGALRM : alarm 함수호출을 통해서 등록된 시간이 된 상황
  * SIGINT : CTRL + C가 입력된 상황
  * SIGCHLD : 자식 프로세스가 종료된 상황

```c++
// "자식 프로세스가 종료되면 mychild 함수를 호출해 달라."
signal(SIGCHLD, mychild);

// "alarm 함수호출을 통해서 등록된 시간이 지나면 timeout 함수를 호출해 달라."
signal(SIGALRM, timeout);

// "CTRL + C가 입력되면 keycontrol 함수를 호출해 달라."
signal(SIGINT, keycontrol);
```

#### alarm 함수

```c++
#include <unistd.h>

unsigned int alarm(unsigned int seconds);
// 0 또는 SIGALRM 시그널이 발생하기까지 남아있는 시간을 초 단위로 반환
```

* 양의 정수를 인자로 전달하면, 전달된 수에 해당하는 시간(초 단위)이 지나서 SIGALRM 시그널이 발생한다. 
* 0을 인자로 전달하면 이전에 설정된 SIGALRM 시그널 발생의 예약이 취소된다. 
* <b>위의 함수호출을 통해서 시그널의 발생을 예약만 해놓고, 이 시그널이 발생했을 때 호출되어야 할 함수를 지정하지 않으면(signal 함수호출을 통해서) 프로세스가 그냥 종료되어 버린다.</b>

```c++
// signal.c

#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void timeout(int sig)
{
	if(sig==SIGALRM)
		puts("Time out!");

	alarm(2);	
}
void keycontrol(int sig)
{
	if(sig==SIGINT)
		puts("CTRL+C pressed");
}

int main(int argc, char *argv[])
{
	int i;
	signal(SIGALRM, timeout);
	signal(SIGINT, keycontrol);
	alarm(2);

	for(i=0; i<3; i++)
	{
		puts("wait...");
		sleep(100);
	}
	return 0;
}

/*
(아무런 입력이 없을 때)
wait...
Time out!
wait...
Time out!
wait...
Time out!
*/
```

* <b>시그널이 발생하면 sleep 함수의 호출로 블로킹 상태에 있던 프로세스가 깨어난다. </b>
* 함수의 호출을 유도하는 것은 운영체제이지만, 그래도 프로세스가 잠들어 있는 상태에서 함수가 호출될 수는 없다. 따라서 시그널이 발생하면, 시그널에 해당하는 시그널 핸들러의 호출을 위해서 sleep 함수의 호출로 블로킹 상태에 있던 프로세스는 깨어나게 된다. 

### sigaction 함수를 이용한 시그널 핸들링

* signal 함수랑 역할은 같다. 
* signal 함수보다 안정적이다. 왜냐하면 signal 함수는 유닉스 계열의 운영체제 별로 동작방식에 있어서 약간의 차이를 보일 수 있지만, sigaction 함수는 차이를 보이지 않는다. 

```c++
#inlcude <signal.h>

int sigaction(int signo, const struct sigaction * act, struct sigaction * oldact);
/*
성공 시 0, 실패 시 -1 반환. 
signo : 시그널 정보
act : 첫 번째 인자로 전달된 상수에 해당하는 시그널 발생시 호출될 함수(시그널 핸들러)의 정보 전달. 
oldact : 이전에 등록되었던 시그널 핸들러의 함수 포인터를 얻는데 사용되는 인자, 필요 없다면 0 전달. 
*/
```

* 위 함수의 호출을 위해서는 sigaction이라는 이름의 구조체 변수를 선언 및 초기화해야 한다. 

```c++
struct sigaction
{
	void (*sa_handler)(int);
	sigset_t sa_mask;
	int sa_flags;
}
```

* 좀비 프로세스의 생성을 막기 위해서는 아래 2개의 변수는 0으로 초기화 하고, 맨 위의 구조체 멤버 sa_handler에 시그널 핸들러의 함수 포인터 값(주소 값)을 저장하면 된다. 

```c++
// sigaction.c

#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void timeout(int sig)
{
	if(sig==SIGALRM)
		puts("Time out!");
	alarm(2);	
}

int main(int argc, char *argv[])
{
	int i;
	struct sigaction act;
	act.sa_handler=timeout;
	sigemptyset(&act.sa_mask);
	act.sa_flags=0;
	sigaction(SIGALRM, &act, 0);

	alarm(2);

	for(i=0; i<3; i++)
	{
		puts("wait...");
		sleep(100);
	}
	return 0;
}
/*
wait...
Time out!
wait...
Time out!
wait...
Time out!
*/
```

### 시그널 핸들링을 통한 좀비 프로세스의 소멸

```c++
// remove_zombie.c

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>

void read_childproc(int sig)
{
	int status;
	pid_t id=waitpid(-1, &status, WNOHANG);
	if(WIFEXITED(status))
	{
		printf("Removed proc id: %d \n", id);
		printf("Child send: %d \n", WEXITSTATUS(status));
	}
}

int main(int argc, char *argv[])
{
	pid_t pid;
	struct sigaction act;
	act.sa_handler=read_childproc;
	sigemptyset(&act.sa_mask);
	act.sa_flags=0;
	sigaction(SIGCHLD, &act, 0);

	pid=fork();
	if(pid==0)
	{
		puts("Hi! I'm child process");
		sleep(10);
		return 12;
	}
	else
	{
		printf("Child proc id: %d \n", pid);
		pid=fork();
		if(pid==0)
		{
			puts("Hi! I'm child process");
			sleep(10);
			exit(24);
		}
		else
		{
			int i;
			printf("Child proc id: %d \n", pid);
			for(i=0; i<5; i++)
			{
				puts("wait...");
				sleep(5);
			}
		}
	}
	return 0;
}

/*
Hi! I'm child process
Child proc id: 9529 
Hi! I'm child process
Child proc id: 9530 
wait...
wait...
Removed proc id: 9530 
Child send: 24 
wait...
Removed proc id: 9529 
Child send: 12 
wait...
wait...
*/
```

## 멀티태스킹 기반의 다중접속 서버

### 프로세스 기반의 다중접속 서버의 구현 모델

* 동시에 둘 이상의 클라이언트에게 서비스를 제공하는 형태로 에코 서버를 확장한다. 
* 다음 그림은 구현할 멀티프로세스 기반의 다중접속 에코 서버의 구현모델.

<p align="center">
<img width="600" src="https://user-images.githubusercontent.com/47745785/116848376-774a6b80-ac27-11eb-950e-a2baa7328f4b.png" alt="다중접속서버">
</p>

* 클라이언트의 서비스 연결요청이 있을 때마다 에코 서버는 자식 프로세스를 생성해서 서비스를 제공한다. 
* 에코 서버는 다음의 과정을 거쳐야 한다. 
  1. 에코 서버(부모 프로세스)는 accept 함수호출을 통해서 연결요청을 수락한다.
  2. 이때 얻게 되는 소켓의 파일 디스크립터를 자식 프로세스를 생성해서 넘겨준다. 
  3. 자식 프로세스는 전달받은 파일 디스크립터를 바탕으로 서비스를 제공한다. 

```c++
// echo_mpserv.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);
void read_childproc(int sig);

int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	
	pid_t pid;
	struct sigaction act;
	socklen_t adr_sz;
	int str_len, state;
	char buf[BUF_SIZE];
	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}

	act.sa_handler=read_childproc;
	sigemptyset(&act.sa_mask);
	act.sa_flags=0;
	state=sigaction(SIGCHLD, &act, 0);
	serv_sock=socket(PF_INET, SOCK_STREAM, 0);
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)
		error_handling("bind() error");
	if(listen(serv_sock, 5)==-1)
		error_handling("listen() error");
	
	while(1)
	{
		adr_sz=sizeof(clnt_adr);
		clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
		if(clnt_sock==-1)
			continue;
		else
			puts("new client connected...");
		pid=fork();
		if(pid==-1)
		{
			close(clnt_sock);
			continue;
		}
		if(pid==0)
		{
			close(serv_sock);
			while((str_len=read(clnt_sock, buf, BUF_SIZE))!=0)
				write(clnt_sock, buf, str_len);
			
			close(clnt_sock);
			puts("client disconnected...");
			return 0;
		}
		else
			close(clnt_sock);
	}
	close(serv_sock);
	return 0;
}

void read_childproc(int sig)
{
	pid_t pid;
	int status;
	pid=waitpid(-1, &status, WNOHANG);
	printf("removed proc id: %d \n", pid);
}
void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

### fork 함수호출을 통한 파일 디스크립터의 복사

* 위의 echo_mpserv.c에서 fork 함수호출을 통한 파일 디스크립터의 복사(자식 프로세스에게)를 보여준다. 

> <b>파일 디스크립터만 복사가 되나?? 아니면 소켓도 같이 복사가 되나??</b>

* <b>소켓은 프로세스의 소유가 아니고 운영체제의 소유이고, 해당 소켓을 의미하는 파일 디스크립터만 프로세스의 소유이다. 
* 소켓이 복사되면 동일한 PORT에 할당된 소켓이 두 개 이상이 되기 때문에 안된다. </b>

<p align="center">
<img width="450" src="https://user-images.githubusercontent.com/47745785/116852579-89c8a300-ac2f-11eb-989c-9fd8d0da7f37.jpg" alt="파일디스크립터복사">
</p>

* 위 그림과 같이 하나의 소켓에 두 개의 파일 디스크립터가 존재하는 경우, 두 개의 파일 디스크립터가 모두 종료(소멸)되어야 소켓은 소멸된다. 
* 때문에 위의 그림과 같은 형태를 유지하면 이후에 자식 프로세스가 클라이언트와 연결되어 있는 소켓을 소멸하려 해도 소멸되지 않고 계속 남아있게 된다. 
* **그래서 fork 함수호출 후에는 다음 그림에서 보이듯이 서로에게 상관이 없는 소켓의 파일 디스크립터를 닫아줘야 한다.**

<p align="center">
<img width="450" src="https://user-images.githubusercontent.com/47745785/116853739-8209fe00-ac31-11eb-9c01-3a5561f2cbcf.png" alt="파일디스크립터지우기">
</p>

* 그래서 echo_mpserv.c 60, 69행 close 함수 호출함. 

## TCP의 입출력 루틴(Routime) 분할

* 지금까지 구현한 에코 클라이언트의 데이터 에코방식은 한번 데이터를 전송하면 에코되어 돌아오는 데이터를 수신할 때까지 기다려야 했다. 
* 프로그램 코드의 흐름이 read write를 반복하는 구조였기 때문이다. 
* 하나의 프로세스 기반으로 프로그램이 동작해서 그렇게 할 수 밖에 없었다. 

<br>

* 이제는 둘 이상의 프로세스를 생성할 수 있으니, 이를 바탕으로 데이터의 송신과 수신을 분리할 수 있다. 

<p align="center">
<img width="450" src="https://user-images.githubusercontent.com/47745785/116856536-43c30d80-ac36-11eb-945a-8f2ffc1ef793.jpg" alt="파일디스크립터지우기">
</p>

* 프로그램이 복잡할수록 구현이 한결 수월해진다. 
* 데이터 송수신이 잦은 프로그램의 성능향상.

```c++
// echo_mpclient.c 

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);
void read_routine(int sock, char *buf);
void write_routine(int sock, char *buf);

int main(int argc, char *argv[])
{
	int sock;
	pid_t pid;
	char buf[BUF_SIZE];
	struct sockaddr_in serv_adr;
	if(argc!=3) {
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_STREAM, 0);  
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=inet_addr(argv[1]);
	serv_adr.sin_port=htons(atoi(argv[2]));
	
	if(connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr))==-1)
		error_handling("connect() error!");

	pid=fork();
	if(pid==0)
		write_routine(sock, buf);
	else 
		read_routine(sock, buf);

	close(sock);
	return 0;
}

void read_routine(int sock, char *buf)
{
	while(1)
	{
		int str_len=read(sock, buf, BUF_SIZE);
		if(str_len==0)
			return;

		buf[str_len]=0;
		printf("Message from server: %s", buf);
	}
}
void write_routine(int sock, char *buf)
{
	while(1)
	{
		fgets(buf, BUF_SIZE, stdin);
		if(!strcmp(buf,"q\n") || !strcmp(buf,"Q\n"))
		{	
			shutdown(sock, SHUT_WR);
			return;
		}
		write(sock, buf, strlen(buf));
	}
}
void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}	
```