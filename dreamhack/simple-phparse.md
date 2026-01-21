created: 2026-01-21 17:29
category: #web 
link: https://dreamhack.io/wargame/challenges/1367

# Write-up
```php
<?php
     $url = $_SERVER['REQUEST_URI'];
     $host = parse_url($url,PHP_URL_HOST);
     $path = parse_url($url,PHP_URL_PATH);
     $query = parse_url($url,PHP_URL_QUERY);
     echo "<div><h1> host: $host <br> path: $path <br> query: $query<br></h1></div>";

     if(preg_match("/flag.php/i", $path)){
        echo "<div><h1>NO....</h1></div>";
     }
     else echo "<div><h1>Cannot access flag.php: $path </h1></div> ";
    ?>
```
- 먼저 이번 문제의 메인 로직이다. 눈에 띄는 부분은 `$_SERVER['REQUEST_URI']`, `parse_url`, `preg_match` 였다.
- 일단 문제 설명에 의하면 `/flag.php` 경로에 플래그가 있다고 했다. 그러나 위 코드 때문에 해당 경로로 이동하면 `preg_match` 함수에 의해 다른 내용이 출력된다. 

`$_SERVER['REQUEST_URI']`에 대해서 공식 문서를 살펴보니 request uri의 뒤에 붙는 상대 경로 부분만 반환하는 듯 하다.

인코딩을 통해서 푼 사람도 있는 것 같지만 이번 문제의 제목에 "parse"가 들어간 만큼, `parse_url` 함수에 무언가가 있을 것이라고 생각했다.
그래서 제미나이를 통해 `parse_url`에 대한 취약점을 나열해보았고 유용했던 내용은 다음과 같다. 
**경로(Path) 시작 부분의 `/` 처리 (CVE-2016-10397 등)**
일부 PHP 버전에서는 URL의 스킴(scheme)이 없을 때 경로의 시작 부분을 호스트로 오인하는 버그가 있었습니다.
- **패턴:** `//google.com/test`
- **문제:** 프로토콜 상대 경로(`//`)를 사용할 때, `parse_url`은 `google.com`을 호스트로 인식합니다. 하지만 이를 검증 없이 다른 문자열과 결합하면 의도치 않은 로컬 파일 경로 접근이나 프로토콜 변조가 가능해집니다.

따라서 `//flag.php`를 사용하면 `$url` 변수에 `//flag.php`가 저장되고 `parse_url`은 호스트 부분을 `flag.php`라고 인식할 것 같았다. 그러면 조건문 검증을 우회할 수 있을 것 같았다. 
우선 온라인 php 샌드박스에서 위 내용을 실험 해보았다. 코드는 다음과 같다.
```php
<?php
	$url = '//flag.php';
	var_dump(parse_url($url));
	
	$url = '/flag.php';
	var_dump(parse_url($url));
?>
```

![[Pasted image 20260121181127.png]]
결과는 위 이미지와 같다. 정말로 "//"와 "/"의 차이로 url을 호스트로 해석하느냐 path로 해석하느냐의 차이가 있었다.

![[Pasted image 20260121180505.png|700]]
최종적으로 `//flag.php` 경로로 접속하여 위 이미지와 같이 플래그를 획득할 수 있었다.

# Reference
- https://hahahoho5915.tistory.com/44
- https://www.php.net/manual/en/reserved.variables.server.php
- https://www.php.net/manual/en/function.parse-url.php