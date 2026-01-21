created: 2026-01-19 19:07
category: #web 
link: https://dreamhack.io/wargame/challenges/960

# Write-up
```php
<?php
    function getRandStr($length = 10) {
        $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $charactersLength = strlen($characters);
        $randomString = '';
    
        for ($i = 0; $i < $length; $i++) {
            $randomString .= $characters[mt_rand(0, $charactersLength - 1)];
        }
        return $randomString;

    }
    require_once('flag.php');
    error_reporting(0);
    $id = getRandStr();
    $pw = sha1("1");
    // POST request
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
      $input_id = $_POST["input1"] ? $_POST["input1"] : "";
      $input_pw = $_POST["input2"] ? $_POST["input2"] : "";
      sleep(1);

      if((int)$input_id == $id && strlen($input_id) === 10){
        echo '<h4>ID pass.</h4><br>';
        if((int)$input_pw == $pw && strlen($input_pw) === 8){
            echo "<pre>FLAG\n";
            echo $flag;
            echo "</pre>";
          }
        } else{
          echo '<h4>Try again.</h4><br>';
        }
      }else {
      echo '<h3>Fail...</h3>';
     }
    ?>
```
- 해당 코드가 로그인 API 로직이다. 만약 입력한 `id`와 `pw`가 맞다면(혹은 우회한다면), 플래그 값이 출력될 것이다.
- `id` 값은 알파벳 대소문자와 숫자로 이루어진 10자리 문자열이고 `pw` 값은 `sha1("1")`의 값이 저장되어 있다. 
- `id` 값은 경우의 수가 너무 다양해서 브루트 포스로 풀기는 어려워 보였다. 심지어 (아마도) 매번 로그인을 시도할 때 마다 `getRandStr` 함수가 실행되어서 매번 `id` 값이 바뀔 것이다. 그리고 `pw` 값은 구현하기는 쉽지만 (온라인 php 에디터로 구한 결과, 356a192b7913b04c54574d18c28d46e6395428ab) `input_pw` 문자열 길이는 8자리여야 한다.
- 따라서, 특정한 방법으로 풀어야하는 문제라고 생각했다. 

솔직히 이 문제를 풀 수 있는 방법을 떠오르기 힘들었어서 댓글로 힌트를 얻었다. `id`와 `pw`를 검증할 때 조건문은 각각 다음과 같았다. 
- `(int)$input_id == $id`
- `(int)$input_pw == $pw`
여기서 " === " 이 아니라 " == " 가 쓰였기 때문에 느슨한 비교로 진행된다. 

로그인을 시도했을 때, `id` 값이 알파벳으로 시작하게 된다면 느슨한 비교를 통해 `id` 값이 정수 0으로 된다. 그리고 `pw` 값은 356까지만 정수로 변환될 것이다. 

![[Pasted image 20260119192050.png]]
최종적으로 위와 같이 문자열 길이를 맞춰서 입력 값을 주었다. `id` 값이 알파벳으로 시작한다면 로그인이 성공할 것이다. 

![[Pasted image 20260119192153.png|700]]
몇 번 시도하다 보니 위 이미지와 같이 플래그를 획득할 수 있었다. 

# Reference
- https://www.moonding.co.kr/php-type-juggling-vulnerability/