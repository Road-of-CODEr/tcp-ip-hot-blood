### Chapter 11. 프로세스간 통신

#### 프로세스간 통신의 기본 이해

- 프로세스간 통신이 가능하다는 것은 두 프로세스가 동시에 접근 가능한 메모리 공간이 있다는 것
- 프로세스간에는 메모리 공간을 공유하지 않음 -> 별도의 방법 사용



#### 파이프 기반의 프로세스 통신

- 파이프는 운영체제에 속하는 자원(fork 함수 호출에 대한 복사 대상이 아님)

  ```c
  #include <unistd.h>
  
  int pipe(int filedes[2]);  //성공 시 0, 실패 시 -1 반환
  ```

- 부모 프로세스가 pipe 함수를 호출하면 파이프가 생성되고 출입구에 해당하는 두 파일 디스크립터를 얻게됨 



#### 파이프 기반의 프로세스간 양방향 통신

- 파이프에 데이터가 전달되면, 먼저 가져가는 프로세스에게 이 데이터가 전달됨
- 파이프를 두개 생성하여 각각 다른 데이터의 흐름을 담당하도록 하면 됨
