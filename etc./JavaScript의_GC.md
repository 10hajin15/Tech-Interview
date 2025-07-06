## JavaScript의 GC
### 가비지 컬렉션(GC)
프로그래밍 언어에서 동적으로 할당된 메모리 중 더 이상 사용되지 않는 영역, 즉 '쓰레기'를 자동으로 찾아 제거하여 메모리 공간을 확보하는 자동 메모리 관리 기법

**⦿ 메모리 생존 주기**

**1. 메모리 할당**

원시 타입의 값들은 스택 영역에 저장, 참조 타입의 값은 힙 영역에 저장되며 그 주소값은 스택 영역에 저장된다.

변수 식별자는 스택의 실행 컨텍스트의 렉시컬 환경에 저장!
![](https://velog.velcdn.com/images/oazin15/post/3f56a54a-b682-498b-aab3-744619e467e5/image.png)

**2. 할당된 메모리 사용(참조)**

v8 엔진에 의해 전역 실행 컨텍스트의 렉시컬 환경에 있는 식별자를 참조

**3. 메모리 해제**

C나 C++은 `free()`, `malloc()`으로 수동으로 해제해야 하지만 JavaScript는 자동으로 관리를 해준다. 어떻게??

### 가비지 컬렉션 알고리즘

**① Reference-Counting**

자신을 참조하는 객체의 개수를 카운팅하고, 참조 되지 않는 객체가 있다면 수거
![스크린샷 2025-07-06 오전 11 50 56](https://github.com/user-attachments/assets/f6a8a59e-0e3d-413c-973d-a63ec35c6494)

- 순환 참조 문제


![](https://velog.velcdn.com/images/oazin15/post/6ec199a9-d5f7-46a8-b402-2618537dcc69/image.png)
```javascript
let obj1 = {};
let obj2 = {};

obj1.a = obj2;
obj2.b = obj1;

obj1 = null;
obj2 = null;
```

`obj1`과 `obj2`는 `null`이 되었지만 서로 참조하기 때문에 할당된 메모리를 해제하지 않는다. → 메모리 누수


**② Mark-and-Sweep**

- 루트에서 도달 가능한 객체를 표시한다
- 루트에서 접근이 불가능한 객체를 수거한다


### V8 메모리 구조
- 자바스크립트는 단일 스레드 언어이기 때문에, V8 엔진은 자바스크립트 컨텍스트마다 하나의 독립된 실행 환경(V8 인스턴스)을 사용한다.
- Resident Set은 실행 중인 프로그램이 실제 물리 메모리에서 차지하고 있는 메모리 영역을 뜻한다.

![](https://velog.velcdn.com/images/oazin15/post/3bde72b1-5bfe-4553-92f4-240f81c0393d/image.png)

**⦿ 힙 메모리**

V8 엔진은 힙 메모리에 객체나 동적 데이터를 저장한다. 또한, 가비지 컬렉션(GC)이 발생하는 곳이다
- New space(Young generation): 새로 만들어진 Object가 저장
- Old space(Old generation): New space의 마이너 가비지 컬렉션이 2번 발생한 후 살아남은 객체들이 저장
    - Pointer space: 살아남은 객체들을 가지며, 이 객체들은 다른 객체를 참조
    - Data space: 문자열, 실수 등의 데이터만 가진 객체들(다른 객체를 참조하지 않는다)을 가짐

### V8 메모리 관리: 가비지 컬렉션
- 객체는 New space의 Semi space에 할당되며 메모리 영역이 모두 할당되면 Minor GC가 발생한다
- GC로부터 살아남으면 다른 Semi space로 이동한다
- 또 한 번의 GC로부터 살아남으면 Old space로 이동한다

> **The Generaional Hypothesis**
> 대부분의 경우 새로운 객체가 오래된 객체보다 쓸모없어질 가능성이 높다는 가설


![](https://velog.velcdn.com/images/oazin15/post/c240154f-506a-44f4-b2fd-215829c00016/image.png)

**⦿ 마이너 GC**
- New space의 쓸모없어진 객체를 수집해주는 마이너 GC
- 마이너 GC에서 살아남은 객체는 비어있는 영역으로 대피(evacuation)를 한다
	→ 2개의 Semi space에서 1개는 언제나 비어있다
- 비어있는 영역을 To space, 머무르는 영역을 From space라고 한다
- From space에서 To space로 이동할 때 객체들은 연속적인 메모리로 이동하여 메모리 단편화를 주기적으로 방지한다

![](https://velog.velcdn.com/images/oazin15/post/ce775832-a869-4198-8b3b-aa1f7c5b88be/image.png)


- 대피가 완료되면 From space에 있는 쓸모없는 객체들은 버리고, From space와 To space의 역할을 바꾼다

![](https://velog.velcdn.com/images/oazin15/post/068a2ca3-ac2a-4014-9171-71cafe69af22/image.png)

- From space에서 2번 생존한 객체는 To space가 아닌 Old space로 이동한다



**⦿ 메이저 GC**
- Old 영역의 메모리가 충분하지 않다고 판단될 때 발생한다
- Mark-Sweep-Compact 알고리즘과 Tri-color 알고리즘을 사용한다


① 마킹(Marking)

어떤 객체가 GC 대상인지 알아내기 위한 단계로 힙 메모리를 방향 그래프로 간주해 깊이 우선 탐색을 수행하며 Tri-color(white, gray, black)로 마킹을 한다.
- white: GC가 아직 탐색을 하지 못한 상태
- gray: 탐색은 했으나 해당 객체가 참조하고 있는 객체가 있는지 확인을 안한 상태
- black: 해당 객체가 참조하고 있는 객체까지 확인을 한 상태


```
1. 모든 객체를 흰색으로 마킹
2. root 객체를 모두 회색으로 마킹한 후 덱에 push_front
3. 덱에서 pop_front로 객체를 꺼내 검은색으로 마킹
4. 검은색으로 마킹된 객체가 참조하는 객체들을 회색으로 마킹하고 덱에 push_front
	- 흰색 → 회색인 경우에만 push_front
    - 여러 객체가 참조하는 경우 이미 회색 or 검은색일 수 있음
5. 3~4번 과정을 덱이 빌 때까지 반복
```

```javascript
let obj1 = {
  x: {
    y: 1
  }
};

let obj2 = obj1;
obj1 = 10;

let obj3 = obj2.x.y;
obj2 = 'happy';

obj3 = null;
```

![](https://velog.velcdn.com/images/oazin15/post/894f9d13-8413-429a-90a2-4446ae82b0a0/image.png)

② 스위핑(Sweeping)

GC가 힙 메모리 순회하면서 흰색으로 마킹된 객체들의 메모리 주소를 기록한다. 이 주소는 free-list에서 사용 가능하다고 표시되며 다른 객체들을 저장하는 데 사용할 수 있다.

③ 압축(Compacting)

스위핑이 일어난 다음 메모리 단편화가 심한 페이지들을 재배치하여 메모리를 확보한다.

### Orinoco: 최적화
마이너 GC와 메이저 GC가 수행될 때 stop-the-world(프로그램 일시 중단)이 발생한다. 이 시간이 길어질수록 프로그램의 응답성과 성능이 저하된다. 이를 개선하기 위해 V8은 [Orinoco 프로젝트](https://v8.dev/blog/trash-talk)를 통해 V8의 성능을 향상시켜왔다.

**⦿ Parallel**

메인 스레드와 헬퍼 스레드들이 동시에 하나의 GC 작업을 협력하여 처리한다.
스레드 간 동기화로 인해 일부 오버헤드는 발생하지만, 그만큼 stop-the-world 시간이 크게 줄어든다.

![](https://velog.velcdn.com/images/oazin15/post/ffbe267d-499e-462b-8a76-ce3847f9a24a/image.png)


**⦿ Incremental**

GC 작업이 작은 단위로 나뉘어, 메인 스레드 실행 도중에 짧게 끼어들며 수행된다. 이 방식은 GC에 소요되는 시간을 여러 시점에 분산시켜, 메인 스레드의 응답성을 높이고 더 나은 사용자 경험(UX)을 제공한다.

![](https://velog.velcdn.com/images/oazin15/post/4bed92a2-4636-4247-90f6-87b95cb8c510/image.png)


**⦿ Concurrent**

메인 스레드는 GC 작업에 관여하지 않고, 헬퍼 스레드들이 해당 작업을 백그라운드에서 수행한다. GC 작업이 완전히 백그라운드에서 처리되기 때문에, 메인 스레드는 자바스크립트 코드를 자유롭게 실행할 수 있다.

![](https://velog.velcdn.com/images/oazin15/post/5b37161c-f931-47b0-9271-522f934bae5f/image.png)


**⦿ Idle-time GC**

Idle GC는 메인 스레드의 유휴 시간을 활용해 GC 작업을 미리 수행하는 기법이다.
예를 들어, 1초에 60프레임을 렌더링해야 하는 크롬 브라우저는 약 16ms 간격으로 프레임을 처리한다. 만약 한 프레임을 그리는 데 16ms보다 적은 시간이 걸리면, 남은 시간 동안 크롬은 유휴 상태가 된 메인 스레드를 활용해 가비지 컬렉션(GC) 작업을 수행한다.

![](https://velog.velcdn.com/images/oazin15/post/0ff606eb-6cb2-44ad-a169-7d6de4ac22cb/image.png)

### Javascript 메모리 누수
① 클로저(Closure)로 인한 메모리 누수
함수 내부에서 외부 변수를 참조하는 클로저가 필요 이상으로 오래 유지될 때 불필요한 변수, 객체가 해제되지 않고 계속 메모리에 남는다.

② 이벤트 리스너 미제거
DOM 요소에 등록한 이벤트 리스너를 제거하지 않아 해당 요소가 삭제되어도 이벤트 리스너 때문에 메모리가 해제되지 않는다.

③ 전역 변수 남용
전역 객체(window 등)에 불필요한 변수, 객체를 할당하면 가비지 컬렉터가 메모리를 회수하지 못한다.

④ DOM 참조 유지
삭제된 DOM 요소에 대한 참조가 코드 내에 남아있으면 브라우저가 해당 노드를 메모리에서 해제하지 못한다.

⑤ 타이머(setInterval, setTimeout) 미정리
setInterval 또는 반복 setTimeout을 중지하지 않으면 관련 콜백과 클로저가 계속 메모리에 남는다.


#### 참조
[이미지 출처] https://v8.dev/blog/trash-talk

https://tech.kakaoent.com/front-end/2022/220519-garbage-collection/

https://ui.toast.com/weekly-pick/ko_20200228

https://www.youtube.com/watch?v=1BoJZqxFYfQ

https://gdsc-cau.github.io/articles/javascript-deep_study-gabage_collection/
