
# 문제 해결 전락

## 빌드 환경 구성 
  각 리소스가 index.js 와 디펜던시를 갖도록 하여 hot-reload 적용
  
  ### 폴더 구조
```bash
└── TypingGame
    ├── src
    │   ├── component       # 각 화면을 구현한 component 폴더
    │   ├── css             # assets css
    │   ├── libs            # 라이브러리
    │   ├── route           # 라우터
    │   └── index.js        # root 파일
    ├── template            # index.html 파일
    └── public              # build output 폴더
```



## 게임 엔진 구현
  게임은 일반적으로 연속적으로 화면을 그리고 지우는것을 반복하여 원하는 에니메이션및 동작이 가능하도록 하는 로직을 기반으로 한다.
  
  이러한 연속성을 만들기 위해서 위해서 게임엔진은 최적화된 loop를 생성하고 관리해야 한다.
  
  javascript를 이용하여 loop를 생성하는 방식에는 여려가지가 있지만 그 중 게임에 사용하기 적합한 requestAnimationFrame룰 사용한다.
  
  requestAnimationFrame를 사용하면 퍼포먼스에 영향을 덜 받으며 구현하려는 게임에 필요한 시간 계산을 용이하게 한다.
  
[특징]
* 백그라운드 동작 및 비활성화 시 중지
* 최대 1ms로 제한되며 1초에 60번 동작
* 다수의 애니메이션에도 동일한 타이머를 참조

#### Game 객체 구현
게임 엔진을 구현한 이 객체는 loop를 생성하기 위한 play 메소드를 갖고 있으며 인자로 매 프레임마다 호출될 콜백을 받는다.

이 콜백 메소드에서 실제 게임을 위한 로직을 구현하는 방식으로 설계한다.

```c
play(draw) {
    if (isplaying || typeof draw !== "function") return;
    isplaying = true;
    callback = draw;
    requestAnimationFrame(loop);
  }
```

Game 객체는 게임에 필요한 리소스를 다운받거나 스토리지에 저장하고 관리하는 기능을 내포한다.

```c
save(key, data) {
    Storage.localStorage(key, data);
  }

  load(key) {
    return Storage.localStorage(key);
  }
```

게임 전반에 걸친 에러와 예외처리를 담당하는 기능을 내포한다.


## 게임 화면 표현
  웹 화면을 조작하는 방법중 가장 일반 적인 방법은 Dom을 이용한 방식이다.
  
  화면의 dom 엘리먼트를 획득하고 각 속성 및 값을 변경할 수 있다.
  
  다만 게임과 같이 1초에 수십번의 화면 조작을 해야 하는 경우에 dom을 이용하는것은 퍼포먼스에 치명적인 악영향을 줄수 있으며 화면 프리징이 발생될수 있다.
  
  따라서 dom 조작을 최소화 하고 canvas에 원하는 화면을 직접 그리는 방식으로 접근한다.
  
canvas에 그려줄 데이터
* 남은 시간, 점수, 문제 단어

dom 조작을 이요할 데이터 
* input 값, 버튼 속성
   
 #### Canvas 객체 구현
 화면을 표헌하기 위한 객체로 빌더 형태의 draw 메소드가 존재한다.
 
 canvas에 화면을 그리기 위해서는 먼저 기존의 화면을 지우고 난 후에 그려야 하기때문에 그리는 순서가 정해지게된다. 
 
 그리는 시점에 begin() 메소드를 호출하여 화면을 지우고 반환되는 canvas로 연속적인 draw가 가능하도록 설계하였다.
  
 예) 
  ```c
  canvas.begin()
    .drawTitle(txt)
    .drawText(`남은시간: ${remaintime}초`, canvas.node.width / 2 - 100, 20, 10)
  ```
  ##### draw text 줄바꿈 처리
  text를 canvas에 그려줄때 고려된 사항중에 중요한 것이 줄바꿈 처리이다.

  실제 제공되는 데이터가 어떤 형태인지알수 없기에 \n 개행은 줄바꿈으로 판단하여

  \n을 기준으로 문장을 나누고 각 나눠지 단어를 아래쪽에 그리는 형태로 구현하였다.


## 라우팅 구현
  히스토리 관리가 편리하고 비교적 구현이 쉬운 SPA - hash 방식을 이용한 라우팅 구현한다.
  
  게임의 특성상 다음 페이지로 이동할때는 history를 남기지 않기 위해서 location.replace를 이용할 것이다. 
  
  router에서는 브라우저의 url 값을 확인하고 path를 추출하고 이 path를 이용하여 미리 저정된 Component 함수를 얻어와서 호출하여 root 노드에 삽입하도록 구현했다. 
  
  ```c
  const path = location.hash.slice(1).toLowerCase() || "/";
  ```

  현 게임 특성상 path parameter를 받도록 구현하지는 않았으며 단순히 path 값만을 파싱하고 라우팅 하도록 하였다.

  게임의 결과나 진행 사항에 대한 데이터는 Game 객체에서 처리하고 있기에 필요하면 각 화면에서 Game 객체를 통해 결과를 얻도록 하였다.

  #### Component 객체 
  리턴 값으로 HTML 테그 스트링을 리턴하며 이 string을 root에 innerHTML로 삽입한다.
  
  이 객체에는 onLoad라는 메소드가 존재하며 innerHTML로 html이 삽입된 후에 호출된다.
  
  ```c
  document.getElementById("root").innerHTML = foundRouter
    ? foundRouter.component()
    : "Page Not Found!";
    
  foundRouter.component.onLoad();
  ```
  
  따라서 화면이 그려진 후에 작업할 로직을 추가할수 있다. 이는 dom이 생성되기 전에 dom에 접근하려는 시도를 제거하여

  안전하게 dom처리가 가능하도록 하기 위함이다.

  
## 게임 로직

  #### 게임 리소스 로드
  Game 객체의 init 메소드를 호출하여 서버로 부터 데이터를 받아오는 방식으로 설계하였다.

  받아온 데이터는 LocalStorage에 저장하고 필요한 시점에 획득할수 있도록 하였다.

  게임이 재시작 되면 기존 LocalStorage를 초기화 하고 새로운 데이터를 받는다.

  
  #### 남은 시간 계산
  play() 메소드에서 전달되는 timeStemp 값을 이용하여 서버에서 받은 데이터의 성공 시간을 비교한다.

  화면에는 초단위로 표시되도록 하였다.

  ```c
   Math.ceil(
    GameEngine.data[currentIndex].second - delayTime / 1000
  );
  ```
  남은시간 1.1초는 결국 2초와 같으므로 계산 결과를 올림하여 화면에 표시한다.

  남은시간이 성공 시간을 초과하면 점수를 -1하고 다음 단어 표시를 위해 currentIndex 값을 +1 시킨다.


  #### 남은시간 표시 추가 기능

  문제 단어 하단에 underline이 생성되는데 이 라인은 시간이 지남에 따라 서서히 줄어 들도록 구현하였다.
  
  성공 시간에서 현재 남은 시간을 나눠 그 값을 현재 단어의 width 값에 곱을 하는 식으로 underline의 width를 계산하였다.

  
  #### 문제 단어 노출

  단순히 currentIndex를 0부터 문제의 길이까지 +1 시키면서 그 인덱스에 맞는 text를 뿌려준다.


  #### 정답 로직

  <Input> 값이 입력되고 엔터(keyCode : 13)을 확인하면 그 값과 현재 표시되는 값을 비교하고 정답여부를 메모리에 저장한다.

  저장된 데이터는 play() 호출될 때, 정답을 확인하고 다음 단어를 표시할지 여부를 판단한다.


## 테스트

  Jest 라이브러리를 이용하여 단위 테스트를 진행한다.

  


  



  

