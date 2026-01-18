created: 2026-01-18 16:49
category: #web 
link: https://dreamhack.io/wargame/challenges/442

# Write-up
우선 문제 서버의 구조는 다음과 같다. 
1. `/flag` 경로에 요청
2. `check_csrf` 함수 호출
3. `/vuln` 경로를 포함하여 `read_url` 함수 호출
4. admin 계정으로 로그인 후 (세션 쿠키 발급받음), 지정한 url에 접속
- 그리고 `/change_password` 경로에서 세션 쿠키로 인증 후 비밀번호를 변경할 수 있다.
- admin 계정으로 로그인한 상태로 메인 화면에 접속하면 플래그를 조회할 수 있을 것으로 보인다.

살펴보니 `/vuln`에서 XSS 취약점 필터링이 되어 있다. 따라서 `<img src=~ />` 태그를 활용하여 `/change_password` 경로에 요청을 날리고 admin의 비밀번호를 수정하여 admin으로 로그인하는 것이 최선이라고 생각하였다. 

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        try:
            pw = users[username]
        except:
            return '<script>alert("user not found");history.go(-1);</script>'
        if pw == password:
            resp = make_response(redirect(url_for('index')) )
            session_id = os.urandom(8).hex()
            session_storage[session_id] = username
            token_storage[session_id] = md5((username + request.remote_addr).encode()).hexdigest()
            resp.set_cookie('sessionid', session_id)
            return resp 
        return '<script>alert("wrong password");history.go(-1);</script>'
```
- 로그인 API는 위 코드와 같다. 
- 간단하게 비밀번호 인증 후, 랜덤으로 `session_id`를 생성한다. 
- 해당 `session_id`를 키로 설정하여 `session_storage`와 `token_storage`에 값을 저장한다.
- `token_storage`의 값은 `username`과 `request.remote_addr`을 활용해 해시값을 사용한다. 이는 로컬 환경에서도 충분히 똑같은 값을 재현할 수 있을 것으로 보인다. 

```python
@app.route("/change_password")
def change_password():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
        csrf_token = token_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')
    pw = request.args.get("pw", None)
    if pw == None:
        return render_template('change_password.html', csrf_token=csrf_token)
    else:
        if csrf_token != request.args.get("csrftoken", ""):
            return '<script>alert("wrong csrf token");history.go(-1);</script>'
        users[username] = pw
        return '<script>alert("Done");history.go(-1);</script>'
```
- 다음으로 비밀번호 변경 API이다. 
- 세션 쿠키를 통해 인증을 진행한다. 특이한 것은 파라미터로 `csrftoken`을 받아서 메모리 상에 저장된 실제 값과 비교한다. 따라서 로그인 API에서 분석한 것과 같이 정말로 로컬에서 똑같은 값을 구현하여 전달 해야겠다고 생각했다. 

![[Pasted image 20260118182105.png]]
- 먼저 `/vuln` 페이지에 접속하여 개발자 도구의 네트워크 항목을 조회한 결과이다. 이미지 태그의 `src` 속성을 통해 HTTP GET 요청을 전송하라는 힌트인 것 같다. 

이미지 태그를 활용하는 건 맞는 듯 하니, 해시 값을 재현하기 위한 코드를 다음과 같이 작성하였다.
```python
from hashlib import md5

username = "admin"
remote_addr = "127.0.0.1"
result = md5((username + remote_addr).encode()).hexdigest()

print(result)
```
- 실제로 `/flag`를 통해서 로그인을 한다면, `request.remote_addr`는 "127.0.0.1"이 될 것이다. 따라서 `remote_addr` 또한 "127.0.0.1"로 초기화하였다. 
- `username`은 당연히 "admin"으로 초기화했다.
- 실행 이후 해시 값은 `7505b9c72ab4aa94b1a4ed7b207b67fb`으로 나왔다.

이후 `/flag` 페이지에서 파라미터로 다음과 같이 전송하였다.
```txt
<img src="/change_password?pw=test&csrftoken=7505b9c72ab4aa94b1a4ed7b207b67fb" />
```
- 여기서는 임의로 `pw=test`로 전달하였다. 만약 성공한다면 admin 계정의 비밀번호가 test로 바뀔 것이다. 
- 재밌는 점은 `/vuln` 페이지에서 비슷한 걸 하려고 하면 `&` 기호 때문에 잘 안된다. 여기서는 따로 `&`가 아니라 인코딩된 값인 `%26`으로 바꿔서 보내야 잘 되었다.

![[Pasted image 20260118183236.png]]
바뀐 비밀번호로 로그인 이후 위 이미지와 같이 플래그를 획득할 수 있었다. 

# Reference
- https://www.geeksforgeeks.org/python/md5-hash-python/