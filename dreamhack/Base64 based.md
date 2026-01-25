created: 2026-01-21 22:11
category: #web 
link: https://dreamhack.io/wargame/challenges/1785

# Write-up
```php
<?php    
	define('ALLOW_INCLUDE', true);

	if (isset($_GET['file'])) {
		$encodedFileName = $_GET['file'];
		if (stripos($encodedFileName, "Li4v") !== false){
			echo "<p class='error'>Error: Not allowed ../.</p>";
			exit(0);
		}
		if ((stripos($encodedFileName, "ZmxhZ") !== false) || (stripos($encodedFileName, "aHA=") !== false)){
			echo "<p class='error'>Error: Not allowed flag.</p>";
			exit(0);
		}
		$decodedFileName = base64_decode($encodedFileName);

		$filePath = __DIR__ . DIRECTORY_SEPARATOR . $decodedFileName;

		if ($decodedFileName && file_exists($filePath) && strpos(realpath($filePath),__DIR__) == 0) {
			echo "<p>Including file: <strong>$decodedFileName</strong></p>";
			echo "<div>";
			require_once($decodedFileName);
			echo "</div>";
		} else {
			echo "<p class='error'>Error: Invalid file or file does not exist.</p>";
		}
	} else {
		echo "<p class='error'>No file parameter provided.</p>";
	}
?>
```
- 코드를 살펴보면, `file` 파라미터를 받아서 필터링 후에 base64 디코딩을 해서 나온 파일명에 `require_once` 함수를 호출한다.
- 즉, 디코딩해서 나온 파일명이 대상 시스템에 존재하는 php 파일이라면 `require_once` 함수에 의해 실행될 것이다.
- 여기서는 플래그 획득을 위해서 디코딩 후 `flag.php`가 나와야한다. 그러므로 단순하게 생각하면 `flag.php`를 base64 인코딩한 값인 `ZmxhZy5waHA=`를 전송하면 될 것이다.

하지만 언급했다시피 `stripos` 함수를 이용한 검증 과정이 있기 때문에 그냥 `ZmxhZy5waHA=`값을 전달하면 조건문에서 걸린다. 그래서 `stripos` 함수를 우회해야겠다고 생각하여 조사를 했다.

조사하다보니 `base64_decode` 함수에서 strict 모드가 아닐 때, base64 알파벳이 아닌 문자는 그냥 버린다는 사실을 알게 되었다. 예를 들어서 띄어쓰기를 사이에 넣으면 버려질 것이다. 그러면 검증 로직은 우회하고 디코딩은 정상적으로 될 것이다. 

최종적으로 `/?file=Zmxh Zy5wa HA=`와 같이 중간에 띄어쓰기를 넣어주었고 이대로 접속하여 플래그를 획득할 수 있었다. 띄어쓰기의 위치는 `(stripos($encodedFileName, "ZmxhZ") !== false) || (stripos($encodedFileName, "aHA=") !== false)` 조건문에 걸리지만 않는다면 어디든 상관없을 것 같다. 

다른 사람의 Write-up을 보니 보통은 `./flag.php`를 인코딩하여 해결했다고 한다. 찾아보니 base64 인코딩은 3 바이트 단위로 인코딩하기 때문에 "./f", "lag", ".ph", "p"로 나뉘어서 인코딩이 될 것이다. 이 결과값들은 모두 필터링에 걸리지 않는다. 

# Reference
- https://www.php.net/manual/en/function.stripos.php
- https://www.php.net/manual/en/function.base64-decode.php
- https://developer.mozilla.org/en-US/docs/Glossary/Base64