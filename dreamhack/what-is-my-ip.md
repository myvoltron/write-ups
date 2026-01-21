created: 2026-01-19 07:19
category: #web 
link: https://dreamhack.io/wargame/challenges/1186

# Write-up
```python
@app.route('/')
def flag():
    user_ip = request.access_route[0] if request.access_route else request.remote_addr
    try:
        result = run(
            ["/bin/bash", "-c", f"echo {user_ip}"],
            capture_output=True,
            text=True,
            timeout=3,
        )
        return render_template("ip.html", result=result.stdout)

    except TimeoutExpired:
        return render_template("ip.html", result="Timeout!")
```
- 우선 처음엔 `["/bin/bash", "-c", f"echo {user_ip}"]` 코드를 보고 명령어 injection 문제라고 판단하였다.
- 따라서 `user_ip`를 조작하기 위해서 `request.access_route`의 정체를 파악 해야겠다고 생각했다. 

```Dockerfile
RUN chown root:$user /flag
RUN chmod 744 /flag
```
- Dockerfile의 일부분이다. `/flag` 파일이 있으므로 명령어 injection을 통해 해당 파일을 읽으면 되겠다고 생각했다. 

`request.access_route`의 정체는 X-Forwarded-For라고 불리는 HTTP header였다. 이는 브라우저 개발자 도구로도 충분히 조작하여 전송할 수 있었다. 

![[Pasted image 20260119072404.png]]
- 따라서 위 이미지 처럼, X-Forwarded-For에 `'1'; cat /flag` 값을 저장했다. 
- 예상대로라면, 1이 출력되고 `/flag`도 출력될 것이다. 

![[Pasted image 20260119072607.png]]
전송 후에 위 이미지처럼, 플래그 값을 획득할 수 있었다.

# Reference
- https://flask.palletsprojects.com/en/stable/api/#flask.Request.access_route
- https://stackoverflow.com/questions/22868900/how-do-i-safely-get-the-users-real-ip-address-in-flask-using-mod-wsgi
- https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/X-Forwarded-For