created: 2026-01-22 15:58
category: #web 
link: https://dreamhack.io/wargame/challenges/128

# Write-up
처음에는 MongoDB가 있으니 NoSQL injection 문제인줄 알고 해당 접근법으로 풀려고 생각했다. ex) `{ $ne: null }`

```javascript
app.get('/api/board/:board_id', function (req, res) {
    MongoBoard.findOne({ _id: req.params.board_id }, function (err, board) {
      if (err) return res.status(500).json({ error: err });
      if (!board) return res.status(404).json({ error: 'board not found' });
      res.json(board);
    })
  });
```
- 따라서 위 `/api/board/:board_id` API를 계속해서 공략하려고 했었다. 
- 하지만, `board_id`로 NoSQL injection 구문을 계속 집어넣고 전체적으로 문자열로 취급되어서 injection 공격이 통하지 않았다. 

그래서 계속 막혀서 댓글 힌트를 보기로 했다. 의외로 이 문제는 NoSQL injection 공격이 아니라 MongoDB의 `_id` 생성 규칙을 이용한 문제였다. 
`_id`는 `ObjectId` 타입으로, 3가지 부분으로 나뉜다.
1. 4바이트 타임 스탬프
2. 5바이트의 임의 값 (프로세스마다 고유)
3. 3바이트의 카운터

이를 알고 다시 게시판을 살펴보니 정말로 규칙성이 보였다. 게시판의 내용은 다음 이미지와 같다.
![[Pasted image 20260122170719.png|700]]
- `6971ca5915fbaf7eb33f3914`
- `6971ca5c15fbaf7eb33f3915`
- `6971ca6415fbaf7eb33f3917`

현재 보여지는 글의 `_id`를 3부분으로 나누면 다음과 같다.
- `6971ca59 15fbaf7eb3 3f3914`
- `6971ca5c 15fbaf7eb3 3f3915`
- FLAG!!
- `6971ca64 15fbaf7eb3 3f3917`

두 번째 부분은 다 같아보인다. 그리고 세 번째 부분은 순서대로 증가되므로 FLAG에서는 `3f3916` 일 것이다. 첫 번째 부분은 타임스탬프 따라서 대충 4씩 증가하는 것 같았다. 따라서 FLAG에서는 `6971ca60` 일 것이다. 
-> `6971ca6015fbaf7eb33f3916`

추정한 값을 입력하여 다음 이미지와 같이 플래그를 획득할 수 있었다.
![[Pasted image 20260122171230.png|700]]

# Reference
- https://www.mongodb.com/ko-kr/docs/manual/reference/bson-types/#objectid