created: 2026-01-27 23:57
category: #web 
link: https://dreamhack.io/wargame/challenges/116

# Write-up
```js
function main(){
        var _0x1046=['2XStRDS','1388249ruyIdZ','length','23461saqTxt','9966Ahatiq','1824773xMtSgK','1918853csBQfH','175TzWLTY','flag','getElementById','94hQzdTH','NOP\x20!','11sVVyAj','37594TRDRWW','charCodeAt','296569AQCpHt','fromCharCode','1aqTvAU'];
        var _0x376c = function(_0xed94a5, _0xba8f0f) {
            _0xed94a5 = _0xed94a5 - 0x175;
            var _0x1046bc = _0x1046[_0xed94a5];
            return _0x1046bc;
        };
        var _0x374fd6 = _0x376c;
        (function(_0x24638d, _0x413a92) {
            var _0x138062 = _0x376c;
            while (!![]) {
                try {
                    var _0x41a76b = -parseInt(_0x138062(0x17f)) + parseInt(_0x138062(0x180)) * -parseInt(_0x138062(0x179)) + -parseInt(_0x138062(0x181)) * -parseInt(_0x138062(0x17e)) + -parseInt(_0x138062(0x17b)) + -parseInt(_0x138062(0x177)) * -parseInt(_0x138062(0x17a)) + -parseInt(_0x138062(0x17d)) * -parseInt(_0x138062(0x186)) + -parseInt(_0x138062(0x175)) * -parseInt(_0x138062(0x184));
                    if (_0x41a76b === _0x413a92) break;
                    else _0x24638d['push'](_0x24638d['shift']());
                } catch (_0x114389) {
                    _0x24638d['push'](_0x24638d['shift']());
                }
            }
        }(_0x1046, 0xf3764));
        var flag = document[_0x374fd6(0x183)](_0x374fd6(0x182))['value'],
            _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
            _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
            operator = [(_0x3a6862, _0x4b2b8f) => {
                return _0x3a6862 + _0x4b2b8f;
            }, (_0xa50264, _0x1fa25c) => {
                return _0xa50264 - _0x1fa25c;
            }, (_0x3d7732, _0x48e1e0) => {
                return _0x3d7732 * _0x48e1e0;
            }, (_0x32aa3b, _0x53e3ec) => {
                return _0x32aa3b ^ _0x53e3ec;
            }],
            getchar = String[_0x374fd6(0x178)];
        if (flag[_0x374fd6(0x17c)] != 0x24) {
            text2img(_0x374fd6(0x185));
            return;
        }
        for (var i = 0x0; i < flag[_0x374fd6(0x17c)]; i++) {
            if (flag[_0x374fd6(0x176)](i) == operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])) {} else {
                text2img(_0x374fd6(0x185));
                return;
            }
        }
        text2img(flag);
    }
```
- 처음 봤을 땐 알 수 없는 16진수 값들이 많아서 당황스러웠다.
- 그래서 하나 하나씩 `console.log` 함수를 찍어가면서 동작 원리를 파악했다.

우선 해당 코드의 `flag` 변수는 당연히 HTML 폼에서 입력한 문자열이다.
`flag[_0x374fd6(0x17c)] != 0x24` 해당 조건문의 의미는 다음과 같다: `flag.length != 36`, 따라서 먼저 HTML 폼에서 길이 36의 문자열을 입력해야한다.

`flag[_0x374fd6(0x176)](i) == operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])`
- 우선 `flag[_0x374fd6(0x176)](i)`는 `flag.charCodeAt(i)`와 의미가 같다. 따라서 입력한 문자열을 순회하면서 UTF-16의 코드 값 획득한다.
- `operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])` 그러므로 `flag`의 각 문자들이 해당 `operator` 함수 결과 값과 같아야한다. 
	- 그래서 `operator` 함수의 동작 원리를 몰라도 된다고 생각했다. 어차피 코드는 나한테 있고 그냥 `console.log`로 각 `operator` 함수 결과를 출력하기만 하면 된다. 

최종적으로 다음과 같은 절차를 마련하였다. 
1. 반복문 내에서 `operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])`의 결과 값을 다시 원래의 문자로 되돌린다.
2. 그러기 위해서 `String.fromCharCode` 함수를 사용할 수 있다.

```js
let result = "";
        for (var i = 0x0; i < flag[findKeywordPtr(0x17c)]; i++) {
            result += String.fromCharCode(operator[i % operator[findKeywordPtr(0x17c)]](_0x4949[i], _0x42931[i]));
        }
        console.log(result);
```
해당 반복문을 위 코드와 같이 수정하였고, 길이 36의 문자열을 입력하니 브라우져 개발자 도구 콘솔 창에서 플래그 값을 획득할 수 있었다.

# Reference
- https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt
- https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/String/fromCharCode