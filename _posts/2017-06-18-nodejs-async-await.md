---
layout: post
title: "nodejs async-await 및 변천사"
description: "nodejs 가 콜백 헬을 탈출하기 위해 어떻게 변화해 왔는지 알아본다"
tags: [nodejs, node.js, async, await, co, promise]
---
### 콜백 지옥을 벗어나기 위해서
nodejs 를 2012년 경 처음 사용한 이후로 0.1x대 버전에서 현재 nodejs 8 버전에 이르기 까지 항상 고민이 되던 것은 어떻게 콜벡 지옥을 벗어날 것이냐 였다. ECMA Script 6 이상이 나오면서 가히 혁신이라고 말할 것들이 나오기 시작했고, 이제 7 이상에서 지원 되는 async-await 을 사용해보고 여전히 다소 불편함은 있지만 콜벡 지옥에서 완전히 벗어났다고 감히 단언하고 싶다. 지금 부터 이 변천사에 대해 간단한 예제와 함께 다뤄보고자 한다.

예제는 [https://github.com/doohwan-yoo/async-await-sample](https://github.com/doohwan-yoo/async-await-sample) 이 곳에 올려두었다.

### 처음에는...
간단한 파일을 쓰고 쓴 내용을 읽는 예제를 작성해서 변천사를 보여주고자 한다.
처음에 보양은 이러했다

```javascript
let fs = require('fs');
let path = require('path');

function testA() {

    let buf = '테스트';

    fs.writeFile(path.join(__dirname, '/sample.txt'), buf, function(err) {
        if(err) throw err;
        console.log('쓰기 완료');

        // 콜벡 안에 또 콜벡
        fs.readFile(path.join(__dirname, '/sample.txt'), 'utf8', function(err, result){
            if(err) throw err;
            console.log("읽기 : " + result);
        });
    });
}

testA();
```
결과
```
쓰기 완료
읽기 : 테스트
```

자바 스크립트 언어에 특성 상 순차적으로 비동기 메소드를 실행하려면 위와 같이 함수에서 콜벡 메소드를 선언하고 그 결과의 처리를 하였다. 저런 모습이 많아지게 되는 것이 바로 콜벡 지옥이였다. 유지보수도 힘들게 했고, 가독성 또한 매우 떨어졌다.

### async waterfall

그래서 나온게 조금 더 순차적인 모습으로 처리해 보고자 나온 것이 async 모듈에 있는 waterfall 함수였다.

```javascript
let asy = require('async');

function testB() {
    let buf = '테스트';

    asy.waterfall([
        function(cb) {
            fs.writeFile(path.join(__dirname, '/sample.txt'), buf, function(err) {
                if (err) throw err;
                else {
                    console.log('쓰기 완료');
                    cb(null);
                }
            });
        },
        function(cb) {
            fs.readFile(path.join(__dirname, '/sample.txt'), 'utf8', function(err, result){
                if(err) throw err;
                else {
                    console.log("읽기 : " + result);
                    cb(null);
                }
            });
        }
    ],
    function(err, result){
        console.log("완료");
    });
}

testB();
```
결과
```
쓰기 완료
읽기 : 테스트
완료
```

보는 바와 같이 순차적으로 실행은 하게 되었지만 코드 분량이 더 늘어났고, 완전히 콜벡지옥도 탈출하지 못했다. (여전히 일정한 Depth 에 콜벡은 마주하게 되었다)

### Generator 함수의 등장! Promise!!! 감사합니다!!

ECMA6 스펙으로 Generator 가 등장했고 그것을 랩핑하는 여러가지 방법들이 등장했다.
Generator 의 설명과 Promise 에 대한 자세한 설명은 하단의 링크를 참조하자. 아주 잘 설명해 주셨다
[http://blog.hellworld.me/6](http://blog.hellworld.me/6)

위와 같은 등장과 람다식의 등장 그리고 co npm 모듈의 등장으로 다음과 같이 함수형으로 변경 및 간결하게 콜백을 탈출하게 되었다.

```javascript

let read = () => {
    return new Promise((resolve, reject) => {
        fs.readFile(path.join(__dirname, '/sample.txt'), 'utf8', function(err, result){
            if(err) reject(err);
            else {
                console.log("읽기 : " + result);
                resolve();
            }
        });
    });
};

let write = (buf) => {
    return new Promise((resolve, rejct) => {
        fs.writeFile(path.join(__dirname, '/sample.txt'), buf, function(err) {
            if (err) reject(err);
            else {
                console.log('쓰기 완료');
                resolve();
            }
        });
    });
};

let co = require('co');

function testC() {
    let buf = '테스트';

    co(function *(){
        yield write(buf);
        yield read();
    }).then((result) => {
        console.log('완료');
    });
}

testC();
```
결과
```
쓰기 완료
읽기 : 테스트
완료
```
코드는 다소 길어 보일 수 있지만 read 와 write 함수가 재사용 된다고 했을 때 함수 testC() 를 봤을 때는 엄청나게 간소화 되었다.


### nodejs7 그리고 async-await 등장
드디어 최종 끝판왕이 등장했다. 위의 단점은 여전히 함수 내부에서 co 라는 모듈에 의존적이라는데 있다. 이러한 단점을 해결해주러 nodejs7 부터는 async-await 이 등장했다. 어떻게 바뀌었는지 한번 보자

```javascript
let read = () => {
    return new Promise((resolve, reject) => {
        fs.readFile(path.join(__dirname, '/sample.txt'), 'utf8', function(err, result){
            if(err) reject(err);
            else {
                console.log("읽기 : " + result);
                resolve();
            }
        });
    });
};

let write = (buf) => {
    return new Promise((resolve, rejct) => {
        fs.writeFile(path.join(__dirname, '/sample.txt'), buf, function(err) {
            if (err) reject(err);
            else {
                console.log('쓰기 완료');
                resolve();
            }
        });
    });
};

async function testD() {

    let buf = '테스트';
    await write(buf);
    await read();
    console.log('완료');
}

testD();
```
결과
```
쓰기 완료
읽기 : 테스트
완료
```

testD() 함수를 보자. 기존에 co를 쓰던 모습에서 함수 자체가 async 를 지원해준다. 이렇게 선언을 해 준 함수 내부에서 await 으로 promisify 한 함수를 호출하면 흡사 동기함수를 호출하는 모양새로 간결한 코드를 만들어 콜백지옥을 탈출할 수 있게 되었다. 이제 Promise 나 실제 제네레이터를 사용해서 만든 함수를 잘 사용해서 기능들을 잘 분리 한다면 nodejs 로도 즐거운 코딩을 할 수 있게 되었다.

현재 nodejs stable 버전은 6.x 이지만 7.x 로 버전업 할 것을 강추 한다.
