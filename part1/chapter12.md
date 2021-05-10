# 12장 - IO Multiplexing(IO 멀티플렉싱)

## 왜 우리가 배웠던 멀티프로세스 서버로 구축하면 안되는가?

- 요청마다 프로세스를 만드는 만큼 많은 양의 연산이 요구된다
- 필요한 메모리 공간도 크다(요청마다 계속 늘어난다)
- 프로세스마다 데이터를 주고받기 힘들다(IPC)

## 멀티플렉싱이란?

- 한글로는 다중화
- 여러 개의 신호를 하나의 신호로 통합하는 과정
- 왜 멀티플렉싱을 하나?
    - 연결의 수를 줄일 수 있어서 훨씬 효율적이니까

## IO 멀티플렉싱이란?

- 하나의 프로세스 혹은 스레드에서 여러 입출력을 다룰 수 있는 기술

> IO 멀티플렉싱 개념을 서버에 적용하게 되면 클라이언트 요청이 늘어나도 서비스를 제공하는 프로세스의 수는 하나가 된다

### 서버 프로세스의 수가 하나인데 어떻게 여러 개의 클라이언트를 처리할 수 있을까?

- 서버 프로세스와 여러 개의 클라이언트를 연결해놓고 감시중.
- 그러다가 요청이 들어오면 그제서야 서버 프로세스가 알고 처리함
- 여기서 select 함수의 개념이 나온다
    - 감시하는 역할, 즉, 어느 소켓으로 요청이 들어오는지 알려주는 역할이 select 함수의 역할

## 비유로 멀티 프로세스 vs 멀티 플렉싱 이해하기

어떤 학교의 반 = 전체 아키텍쳐, 교사 = 서버(프로세스), 학생 = 클라이언트, 질문 = 요청(송수신)

### 멀티 프로세스

- 반에 학생이 새로 전학올 때마다 교사를 1명씩 추가해서 학생의 질문을 받음

### 멀티 플렉싱

- 반에 학생이 새로 전학와도 교사는 항상 1명
- 학생은 손을 들어서 질문을 하고 손든 학생만 교사는 (select 함수로 감시해서) 질문을 받음

## select 함수의 이해와 서버의 구현

- 책에서는 관찰이라는 단어를 많이 사용했지만 감시라는 단어가 더 맞다고 생각함
- select 함수를 사용하면 한 곳에 여러 개의 파일 디스크립터를 비트 배열로 모아놓고 동시에 이들에게 어떤 이벤트가 일어나는지 감시할 수 있다
    - 해당 파일 디스크립터의 비트가 1이면 감시 대상이라는 걸 나타낸다.
- 이벤트가 뭔가?
    - 해당 소켓에 대한 입력, 출력, 에러
- 이벤트가 발생하면, 이벤트가 발생한 파일 디스크립터의 수를 반환
- 여기서 파일 디스크립터는 소켓으로 해석할 수 있다
- 매크로 함수를 통해 전체를 초기화를 해주거나, 파일 디스크립터 정보를 등록하거나 해줄 수 있다

### select 함수

```c
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval * timeout);
```

**매개변수의 뜻 알아보기**

1. maxfd
    1. 감시 대상이 되는 파일 디스크립터 수
        - 파일 디스크립터 값은 생성될 때마다 1씩 증가하기 때문에 가장 큰 파일 디스크립터 값에 1을 더해서 인자로 전달하면 된다.
            - 1을 더하는 이유는 파일 디스크립터 값이 0에서부터 시작하기때문에.
2. *readset
    1. 데이터 수신을 감시할 소켓의 목록
    2. 수신할 데이터를 지니고 있는 소켓이 존재하는지
    3. 존재한다면 해당 소켓(파일 디스크립터) 정보를 등록
3. *writeset
    1. 데이터 송신을 감시할 소켓의 목록
    2. 데이터 전송(write) 시 블로킹되지 않고 바로 전송할 수 있는 소켓이 존재하는지
    3. 출력 스트림이 꽉 차지 않아서 바로 전송가능한 소켓 정보를 등록
4. *exceptset
    1. 예외 상황이 발생한 소켓 정보를 등록
5. timeout
    1. 타임아웃 설정
    2. select 함수는 감시중인 파일 디스크립터 값에 변화가 생겨야 반환을 한다. 변화가 생기지 않으면 무한정 블로킹 상태에 빠진다. 블로킹 상태에 빠지는 것을 방지하기 위해 타임아웃을 지정한다.
    3. 즉, timeout 시간 동안 select가 실행돼 블록 상태로 있으면서 소켓의 변화를 감시하고 있다.
    4. 타임아웃을 설정하고 싶지 않다면 NULL을 전달하면 된다.

### select 함수의 반환

- 오류 발생 시 `-1`을 반환한다
- 타임 아웃 시 `0`을 반환한다
- `0이 아닌 양수`가 반환되면 **그 수만큼** 파일 디스크립터에 변화가 발생했음을 의미한다
    - 예를 들어, `*readset`이라면 데이터 수신 여부의 감시 대상이므로 해당 소켓으로 수신된 데이터가 존재하는 경우 → 이게 변화가 발생한 경우라고 본다.
- select 함수호출이 완료되고 나면 1로 설정된 **모든 비트 중 변화가 발생하지 않은 비트는 다 0으로 변경된다.** 변화가 발생한 파일 디스크립터에 해당하는 비트만 1 그대로 남게 된다.

## 왜 select 기반 IO 멀티플렉싱은 느릴까?

- select 함수의 반환 값은 파일 디스크립터의 수라서 어떤 파일 디스크립터인지 파악하려면 매번 fd_set의 반복문을 돌아야 함

    ```c
    // tcp/ip 책 p.284

    int main(int argc, char * argv[]) {
        while (1) {
            // ...

            // select함수가 1이상이면

            for (int i = 0 ; i < fd_max + 1 ; i++) {
                // ...
            }
            
        }
    }
    ```

- select 함수를 호출할 때마다 fd_set을 복사해야 함

    ```c
    // tcp/ip 책 p.281

    int main(int argc, char * argv[]) {
        fd_set reads, temps;
        
        // ...
        
        while (1) {
            temps = reads;
            // ...
            // select 함수를 호출하면 변화가 없는 파일디스크립터는 0으로 초기화
        }
    }

    ```

- select 함수를 호출할 때마다 감시 대상의 정보 전체를 운영체제에게 전달해야 함

    ```c
    // tcp/ip 책 p.281

    int main(int argc, char * argv[]) {

        // ...
        
        while (1) {
            result = select(1, &temps, 0, 0, &timeout);
            // ...
        }
    }

    ```

- 검사할 수 있는 fd 개수에 제한이 있음
    - 최대 1024개

## Selector 기반 NIO 멀티플렉싱 in Java

- 자바에서도 입출력 채널(위에서 말한 파일 디스크립터라고 보면 될 것 같습니다)을 감시할 수 있는 컴포넌트를 가지고 있다
- 바로 Selector. 여러 개의 NIO 채널을 감시하고 언제 사용할 지 알려줄 수 있는 컴포넌트입니다. Selector를 사용하면 단일 스레드를 사용해 여러 채널을 관리할 수 있습니다
    - 채널이란 송신기와 수신기 사이에서 입출력을 위해 존재하는 가상의 장치입니다.
- 해당 코드는 [여기](https://github.com/aegis1920/my-lab/commit/408c98d764d9b3dbecc861b1ad4868f5013574a1)에 있습니다.

## 출처

- 열혈 TCP/IP 소켓 프로그래밍
- [https://m.blog.naver.com/PostView.nhn?blogId=manhdh&logNo=120164273361&proxyReferer=https:%2F%2Fwww.google.com%2F](https://m.blog.naver.com/PostView.nhn?blogId=manhdh&logNo=120164273361&proxyReferer=https:%2F%2Fwww.google.com%2F)
- [https://en.wikipedia.org/wiki/Multiplexing#cite_note-2](https://en.wikipedia.org/wiki/Multiplexing#cite_note-2)
- [https://www.baeldung.com/java-nio-selector](https://www.baeldung.com/java-nio-selector)
- Java NIO, 블로킹, 논블로킹 관련 좋은 글들
    - [https://blog.naver.com/n_cloudplatform/222189669084](https://blog.naver.com/n_cloudplatform/222189669084)
    - [https://blog.daum.net/haha25/5387473](https://blog.daum.net/haha25/5387473)
    - [https://jongmin92.github.io/2019/02/28/Java/java-with-non-blocking-io/](https://jongmin92.github.io/2019/02/28/Java/java-with-non-blocking-io/)
    - [https://jongmin92.github.io/2019/03/03/Java/java-nio/](https://jongmin92.github.io/2019/03/03/Java/java-nio/)
    - [https://m.blog.naver.com/PostView.nhn?blogId=parkjy76&logNo=30165812423&proxyReferer=https:%2F%2Fwww.google.com%2F](https://m.blog.naver.com/PostView.nhn?blogId=parkjy76&logNo=30165812423&proxyReferer=https:%2F%2Fwww.google.com%2F)
    - [https://notes.shichao.io/unp/ch6/](https://notes.shichao.io/unp/ch6/)
    - [http://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/](http://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/)
    - [https://limdongjin.github.io/concepts/blocking-non-blocking-io.html#특정-블로그-자료에-따른-분류](https://limdongjin.github.io/concepts/blocking-non-blocking-io.html#%E1%84%90%E1%85%B3%E1%86%A8%E1%84%8C%E1%85%A5%E1%86%BC-%E1%84%87%E1%85%B3%E1%86%AF%E1%84%85%E1%85%A9%E1%84%80%E1%85%B3-%E1%84%8C%E1%85%A1%E1%84%85%E1%85%AD%E1%84%8B%E1%85%A6-%E1%84%84%E1%85%A1%E1%84%85%E1%85%B3%E1%86%AB-%E1%84%87%E1%85%AE%E1%86%AB%E1%84%85%E1%85%B2)
