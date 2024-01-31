---
date: '2023-10-05'
title: 'navigator.clipobard Web API'
categories: ['트러블슈팅']
summary: '엘리스 1차 쇼핑몰 프로젝트 작업 중 발생한 복사하기 기능 오류'
thumbnail: '../development/mvc-design-pattern.png'
---

# 복사하기 기능 오류

엘리스 1차 프로젝트 팔이피플 쇼핑몰 사이트 작업 중 발생한 오류

```jsx
const orderId = new URL(urlStr).searchParams.get('orderId');
guestModeEl.innerHTML = `주문번호 <button type="button" id="order-id">${orderId}</button>와 비밀번호를 기억해주세요!`;
const orderIdCopy = document.querySelector('#order-id');

orderIdCopy.addEventListener('click', async () => {
  try {
    await navigator.clipboard.writeText(orderId);
    alert(`주문번호: ${orderId}`);
  } catch (err) {
    console.log('복사실패', err);
  }
});
```

---

## 1. 문제 정의

배포 후, 결제완료 페이지에서 구현한 주문번호 복사하기 기능이 되지않는 문제를 발견했다.

---

## 2. 사실 수집

콘솔에 `navigator.clipboard is undefined` 라는 에러를 확인했고, 로컬서버 환경에서는 정상작동이 되는 것을 확인했기 때문에 배포서버 환경에서 나타나는 문제로 인식하였다.

---

## 3. 원인 추론

사용한 코드는 Web API인 `Clipboard API`이고, mdn 공식문서에서 확인해본 결과 해당 API는 보안 컨텍스트(HTTPS)나 localhost 에서만 사용할 수 있는 것으로 확인되었다.

때문에 localhost 주소였던 로컬서버에서는 문제없이 작동되었지만 배포서버는 HTTP 환경이어서 작동이 되지 않았던 것이다.

[🔗 참고자료 | Clipboard - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard)

---

## 4. 조사방법 결정

배포서버는 HTTP 프로토콜이기 때문에 HTTPS 프로토콜 환경으로 다시 배포하여 기능이 정상 작동할 수 있도록 Clipboard API를 사용하거나, 다른방법을 찾아 복사하기 기능을 구현해야했다.

1. **사용자가 직접 브라우저 설정하기**  
   조사해보니 HTTP에서 해결할 수 있는 방법도 있었는데, 사용자가 직접 브라우저에서 방문할 사이트의 주소 안전한것으로 취급하는 설정 허용시켜줘야하는 것이었다.  
    하지만 보안이슈와 사용자가 직접 허용을 해줘야하는 불필요함이 발생해 안좋은 방법이라고 판단되었다.

   _크롬 브라우저 설정 → 개인 정보 보호 및 보안 → 사이트 설정 → 추가 콘텐츠 설정 → 안전하지 않은 콘텐츠_

2. **✅ Document: execCommand() 메서드 사용하기**  
   document가 디자인 모드, 즉 편집 가능한 상태가 되면 `exeCommand()`매서드를 사용할 수 있는데 execCommand를 호출 후 copy 명령어를 사용한다.  
   로직은 input같은 form 엘리먼트에서만 사용 가능한 `select()`메서드를 통해 선택 영역을 지정해준다.
   `select()`로 선택한 영역을 범위로 execCommand의 copy 메서드가 작동하며 클립보드에 정상적으로 저장이 된다.

   ```jsx
   function onCopy(e) {
     e.preventDefault();
     const content = uuid.current;
     // uuid => ref로 받아온 input의 current
     if (content) {
       content.select();
       document.execCommand('copy');
     }
   }
   ```

   [🔗 참고자료 | Document.execCommand() - Web API | MDN](https://developer.mozilla.org/ko/docs/Web/API/Document/execCommand)

---

## 5. 조사 방법 구현

```jsx
const orderId = new URL(urlStr).searchParams.get('orderId');
guestModeEl.innerHTML = `주문번호 <label for="order-id" id="order-id-label">${orderId}</label>
<input id="order-id" />와 비밀번호를 기억해주세요!`;
const orderIdInput = document.querySelector('#order-id');
const orderIdCopy = document.querySelector('#order-id-label');

orderIdCopy.addEventListener('click', () => {
  orderIdInput.value = orderId;
  orderIdInput.select();
  document.execCommand('copy');
  alert('주문번호가 복사되었습니다!');
});
```

`Document: execCommand()` 메서드를 사용하려면 input이나 textarea 같은 form 엘리먼트에서 사용 가능한 입력태그를 이용해야해서 기존의 button 태그에서 input 태그로 변경하였다.

---

## 6. 결과 관찰

기능은 잘 구현되지만 input 태그가 select 되었을 때 화면상 입력태그가 선택된 모양이 너무 이상했다. 사용자가 클릭을 하고싶은 모양인 버튼 디자인처럼 보이길 원했다.

그래서 클릭되는 요소를 label 태그로 바꾸고 input 태그와 연결시켜주었다. 그리고 input 태그를 보이지않게 하기위해 input 태그에 `disable` 태그 속성, `display: none`또는 `visibillity: hidden` 으로 스타일을 적용하였더니 이번에는 기능이 구현되지않았다.

input 태그를 숨기거나 비활성화되게 만들면 `Document: execCommand() 메서드`가 적용되지 않는 듯 했다. 고민 끝에 `position: absolute`로 요소를 겹치게해주고 `opacity: 0`으로 input 태그를 임의로 숨겨서 기능도 작동하고 UI 상에서도 label 태그를 button 태그처럼 보이도록 원하는대로 변경하였다.

---

### 마치며

`Document: execCommand()` 메서드는 현재 대부분 브라우저에서 지원하긴 하지만 웹 표준에서는 삭제된 기능이므로 clipboard.js 와 같은 라이브러리를 사용하는것도 좋은 방법일듯 하다.

아니면 `window.isSecureContext` 속성을 사용해 HTTPS 프로토콜 환경인지 조건문으로 확인하여 `Document: execCommand()`와 `Clipboard API`를 적절히 사용하는 방법도 좋을 것 같다.

```jsx
if (window.isSecureContext) {
  // Clipboard API 사용
} else {
  // Document: execCommand() 메서드 사용
}
```

[🔗 참고자료 | navigator.clipboard is undefined in JavaScript issue [Fixed] | bobbyhadz](https://bobbyhadz.com/blog/navigator-clipboard-is-undefined-in-javascript)

---

### 추가 내용

> **HTTP와 HTTPS**
>
> - HTTP는 웹에서 서버와 클라이언트가 어떻게 메세지를 서로 주고받을지 정해놓은 규칙이자 HTML문서같은 정보들을 빠르게 교환하기 위한 프로토콜이다. 브라우저가 웹 서버에 HTTP요청을 하고 웹 서버에서 HTTP응답을 하는것이고 우리가 사이트에 방문할 때 요청을 하고 응답으로 사이트를 보여주는 것이다.
> - HTTPS는 HTTP에 데이터 암호화가 추가된 프로토콜이다. 브라우저랑 서버가 데이터를 전송하기 전에 암호화된 연결을 설정한 것이며 암호화된 데이터를 전송한다.
