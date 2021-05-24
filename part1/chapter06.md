# 6장 - UDP 기반 서버/클라이언트

## UDP(User Datagram Protocol)에 대한 이해

### UDP 소켓의 특성은?

- 전송 계층에 있는 프로토콜
- UDP는 TCP의 ACK 처럼 메시지를 보내는 일도 없고 SEQ와 같이 패킷에 번호를 부여하는 일도 없습니다.
- 손실나면 손실나는대로, 잘 받았는지 확인도 하지 않습니다. (**흐름 제어를 하지 않음!**)
- 그래서 **TCP보다 간결한 구조를 가지고 있고 빠르고 구현하기도 용이합니다.**
    - 단, 한번에 송수신하는 데이터 양이 많다면 TCP도 UDP처럼 빠른 속도를 낼 수 있습니다
        - 여러번 송수신하고 데이터 양이 적다면 TCP는 그만큼 흐름 제어하는 횟수가 많고 연결 수립, 연결 해제를 많이 할테니까요

### UDP 소켓의 구조

- **UDP = UDP Header + UDP Data**
- Header(각각 2바이트 씩, 총 8바이트)
    - 소스 Port
        - 0~ 65535, 2바이트라 2^16이라서
    - 목적지 Port
        - 위와 마찬가지
    - 전체 길이
        - UDP 헤더와 UDP 데이터를 구성하는 전체 바이트 수를 지정합니다
    - 체크섬
        - 전송된 데이터가 변형됐는지 확인하는 값

**IP 프로토콜에서 전체 길이를 어차피 재는데 왜 전체 길이 필드가 필요할까?**

IP 프로토콜에서는 전체 길이를 재는 필드가 있어서 중복되나 IP 프로토콜이 아닌 다른 프로토콜과 함께 쓰일 경우,  UDP 길이 필드가 필요하다고 합니다.

### UDP의 역할

- UDP의 역할 중 가장 중요한 것은 호스트로 수신된 패킷을 PORT 정보를 참조하여 최종 목적지인 UDP 소켓에 전달하는 것입니다.
- 호스트까지의 데이터 전달은 IP의 역할입니다.
- 전체적인 Flow는 호스트까지 데이터 전달은 IP가 하고 그 후에 UDP가 호스트에 수신된 패킷을 PORT 정보를 참조해 목적지 UDP 소켓에 전달합니다.

### UDP는 어디에 사용되나?

- 압축 파일의 경우, 파일의 일부만 손실되어도 압축 해제가 어려워 TCP 기반으로 송수신한다
- 실시간 영상 및 음성을 전송하는 경우, 파일의 일부가 손실되어도 잠깐의 화면 떨림, 약간의 잡음 정도라 UDP 기반으로 송수신한다.
- 그래서 보통 UDP는 VoIP(음성통화), mVoIP(카카오 음성통화), 온라인 게임에서 많이 쓰인다.

### TCP가 UDP에 비해 느린 이유는?

- 연결수립(`3 way handshake`), 연결 해제(`4 way handshake`) 과정이 있습니다
- **신뢰성 보장을 위한 흐름 제어 기능이 있습니다**

> 데이터 양은 작으면서 잦은 연결이 필요한 경우, UDP가 TCP보다 훨씬 효율적이고 빠르게 동작합니다

## UDP의 데이터 송수신 특성과 UDP에서의 connect 함수 호출

### 데이터의 경계가 존재하는 UDP 소켓

- UDP는 데이터의 경계가 존재하기 때문에 **하나의 패킷이 하나의 데이터**로 간주된다. 그래서 데이터 그램 패킷이라고 표현한다.
- 여러 개의 데이터를 보냈을 때 TCP는 단 한번의 입력 함수 호출을 통해서 모든 데이터를 읽어들일 수 있으나 **UDP는 그만큼 입력 함수를 호출해야한다.**
- 즉, TCP는 TCP 클라이언트가 10바이트 데이터를 3번 연속으로 보냈다면 TCP 서버가 30바이트 데이터로 1번에 받을 수 있다. 그러나 UDP는 UDP 클라이언트가 10바이트를 3회에 걸쳐 데이터를 보냈다면 UDP 서버도 마찬가지로 3회에 걸쳐 데이터를 받아야 한다.

### connected UDP 소켓, unconnected UDP 소켓

- connected 소켓
    - 목적지 정보가 등록되어있는 소켓
- unconnected 소켓
    - 목적지 정보가 등록되어있지 않아 매번 등록해줘야하는 소켓
- 하나의 호스트와 오랜 시간 데이터를 송수신해야 한다면 connected로 만들어 성능을 향상시키자
    - UDP 소켓에 IP와 PORT를 등록하는 행위와 UDP 소켓에 등록된 목적지 정보를 삭제하는 행위는 UDP 전송과정 시간의 1/3에 해당한다.

## UDP 기반 서버/클라이언트의 구현

- UDP는 연결 수립, 연결 해제 과정이 필요없으므로 listen, accept 함수 호출이 필요없습니다
- 그냥 UDP 소켓을 생성하고 데이터 송수신만 하면 됩니다.
- **서버와 클라이언트 모두 소켓이 하나만 있으면 됩니다.** (TCP처럼 클라이언트 소켓이 많아진다고 해서 서버의 소켓이 그만큼 늘어나야되는 게 아닙니다)
- UDP는 연결 수립, 연결 해제가 없으므로 연결 상태가 없습니다. 그래서 데이터를 전송할 때마다 **반드시 목적지의 주소 정보를 추가해야합니다.**

### UDP 기반의 데이터 입출력 함수 (C언어 기준)

- sendto 함수
    - UDP 소켓의 파일 디스크립터, 전송할 데이터(버퍼 주소값, 크기), 목적지 주소 정보
    - 이 함수가 호출되는 시점에 해당 소켓에 IP와 PORT 번호가 자동으로 할당된다
        - 그래야 프로그램이 종료될 때까지 계속 해당 IP와 PORT로 유지된다
- recvfrom 함수
    - UDP 소켓의 파일 디스크립터, 수신할 데이터(버퍼 주소값, 최대 크기), 발신지 정보

## UDP 기반의 에코 서버, 에코 클라이언트, 테스트

코드는 [여기](https://github.com/aegis1920/my-lab/tree/master/tcp-ip/src/main/java/com/bingbong/tcpip/chapter06)에 있으며 Java 기반입니다. :)

### UDP 에코 서버

```java
import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.SocketException;

public class UdpEchoServer extends Thread {

    private final DatagramSocket socket;
    private final byte[] buffer = new byte[256];

    public UdpEchoServer(int port) throws SocketException {
        socket = new DatagramSocket(port);
        System.out.println(port + " 포트로 UDP 서버가 실행됐습니다!");
    }

    @Override
    public void run() {
        DatagramPacket packet = new DatagramPacket(buffer, buffer.length);

        while (true) {
            try {
                socket.receive(packet);
                String receivedData = new String(packet.getData(), 0, packet.getLength());
                System.out.println("클라이언트로부터 받은 메시지 : " + receivedData);

                if ("q".equals(receivedData)) {
                    socket.send(packet);
                    System.out.println("서버를 종료합니다.");
                    break;
                }
                System.out.println("클라이언트로부터 받은 메시지를 재전송합니다.");
                socket.send(packet);
            } catch (IOException e) {
                e.printStackTrace();
                break;
            }
        }

        socket.close();
    }
}
```

### UDP 에코 클라이언트

```java
import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.nio.charset.StandardCharsets;

public class UdpEchoClient {

    private final DatagramSocket socket = new DatagramSocket();
    private final int port;
    private final InetAddress address;

    public UdpEchoClient(InetAddress address, int port) throws SocketException {
        this.address = address;
        this.port = port;
    }

    public void makeConnectedUdp() {
        socket.connect(address, port);
    }

    public String send(String message) throws IOException {
        byte[] buffer = message.getBytes(StandardCharsets.UTF_8);

        DatagramPacket packet;

        if (socket.isConnected()) {
            packet = new DatagramPacket(buffer, buffer.length);
        } else {
            packet = new DatagramPacket(buffer, buffer.length, address, port);
        }

        socket.send(packet);
        socket.receive(packet);

        return new String(packet.getData());
    }

    public void close() {
        socket.close();
    }

    public boolean isConnected() {
        return socket.isConnected();
    }
}
```

### UDP 클라이언트 및 서버 테스트

```java
import static org.assertj.core.api.AssertionsForClassTypes.assertThat;

import java.io.IOException;
import java.net.InetAddress;
import java.net.SocketException;
import java.net.UnknownHostException;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

class UdpEchoTest {

    private UdpEchoClient client;

    @BeforeEach
    public void setup() throws UnknownHostException, SocketException {
        int port = 5001;
        new UdpEchoServer(port).start();
        client = new UdpEchoClient(InetAddress.getLocalHost(), port);
    }

    @AfterEach
    void tearDown() throws IOException {
        client.send("q");
        client.close();
    }

    @DisplayName("Echo Client, Echo Server 테스트, not connected UDP 소켓")
    @Test
    void sendMessage() throws IOException {
        String message1 = "Hello World!1";
        String receivedMessage1 = client.send(message1);

        assertThat(receivedMessage1).isEqualTo(message1);

        String message2 = "Hello World!2";
        String receivedMessage2 = client.send(message2);

        assertThat(receivedMessage2).isEqualTo(message2);

        String message3 = "Hello World!3";
        String receivedMessage3 = client.send(message3);

        assertThat(receivedMessage3).isEqualTo(message3);
        assertThat(client.isConnected()).isFalse();
    }

    @DisplayName("Echo Client, Echo Server 테스트, connected UDP 소켓")
    @Test
    void sendMessageWithConnectedUdp() throws IOException {
        // connect 함수
        client.makeConnectedUdp();

        String message1 = "Hello World!1";
        String receivedMessage1 = client.send(message1);

        assertThat(receivedMessage1).isEqualTo(message1);

        String message2 = "Hello World!2";
        String receivedMessage2 = client.send(message2);

        assertThat(receivedMessage2).isEqualTo(message2);

        String message3 = "Hello World!3";
        String receivedMessage3 = client.send(message3);

        assertThat(receivedMessage3).isEqualTo(message3);
        assertThat(client.isConnected()).isTrue();
    }
}
```

## 출처

- 윤성우의 열혈 TCP/IP 소켓 프로그래밍
- [https://stackoverflow.com/questions/41394281/why-udp-header-has-length-field](https://stackoverflow.com/questions/41394281/why-udp-header-has-length-field)
- [https://www.baeldung.com/udp-in-java](https://www.baeldung.com/udp-in-java)
