created: 2026-01-30 00:16
category: #web 
link: https://dreamhack.io/wargame/challenges/435

# Write-up
CSP 헤더가 추가된 XSS 문제이다. 
`/flag` 페이지에서 CSP를 우회할 수 있는 어떠한 데이터를 입력하여 `/memo` 페이지에 플래그를 써서 획득할 수 있을 것으로 보인다. 
`/flag` 요청 시 서버 내부적으로 쿠키에 플래그를 삽입한다... 하지만 CSP 정책 때문에 해당 쿠키를 `/memo`에 유출시키는 게 쉽지 않다. 

우선 이 문제의 CSP 정책은 다음과 같다.
```python
response.headers[
        "Content-Security-Policy"
    ] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}'"

```
- `default-src 'self'`: 기본적으로 페이지의 현재 출처에서 로드되는 리소스만 허용
- `img-src https://dreamhack.io`
- `style-src 'self' 'unsafe-inline'`
- `script-src 'self' 'nonce-{nonce}`: 페이지의 현재 출처에서 로드되거나 nonce가 맞으면 허용

이번에는 제미나이를 활용하여 취약점을 찾아보았다. `script-src 'self' 'nonce-{nonce}'`가 취약점이다.
- 'nonce-{nonce}'는 사실 inline 스크립트를 실행할 때 필요한 것이다.
- 'self' 현재 출처에서 로드되는 `.js` 파일은 허용된다. (nonce가 필요없다!)
따라서 `/flag` 페이지 내부적으로 `/vuln` 페이지를 거치는 데 `/vuln` 페이지에서 쿼리로 전달한 것이 그대로 반사되는 것을 활용한다.

따라서 다음과 같은 쿼리를 작성할 수 있었다. `<script src="/vuln?param=alert(1)"></script>`
- `/vuln` 페이지에서 테스트해보니 드디어 alert이 뜨게 되었다!
- "/vuln?param=alert(1)"은 결국 내부 도메인이기 때문에 `script-src 'self'`에 의해 로드되어 실행되는 것이 허용되는 듯 하다.
이렇게 스크립트를 실행시킬 수 있는 방법을 찾았으므로 플래그를 획득하는건 쉽다. 

`<script src="/vuln?param=location.href=`/memo?memo=${document.cookie}`"></script>`
`/flag` 페이지에서 해당 스크립트를 전송 후 `/memo` 페이지에서 정상적으로 플래그를 획득할 수 있었다.

ps. 결국은 신뢰 도메인에 소스코드를 업로드 후 로드하는 방법인 것 같다. `<script>` 태그를 inline 방식으로만 쓰는 것을 생각해서 `src` 속성으로 로드할 생각을 못했었다. 까먹지 말자...
# Reference
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy
- https://www.hahwul.com/blog/2019/csp-bypass-technique-xss/
- https://hg2lee.tistory.com/entry/Content-Security-PolicyCSP-Bypass