## chapter 8: 도메인 이름과 인터넷 주소
- DNS 가 왜 필요할까?

```cpp
#include <netdb.h>

struct hostent * gethostbyname(const char * hostname);
```
<br/>

![결과](/assets/dns_hostent_result.png)


![hostent](/assets/dns_hostent.png)


### 어떻게?
- 그렇다면 DNS 서버의 IP 주소는 어떻게 알까?
- IP 주소는 알아왔다고 치자, 어떻게 보내지?
- 같은 subnet 인건 어떻게 알지?(gateway 주소)


### MORE
- [DNS query 과정](https://www.cloudflare.com/ko-kr/learning/dns/what-is-dns/)
- ARP
- Routing Protocol: RIP, OSPF, BGP
- Longest prefix match
