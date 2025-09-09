## 1️⃣ 카카오 동의 화면의 **CSS 조절 가능 여부**

* ❌ **불가능**

    * 동의 화면은 카카오 도메인(`kauth.kakao.com`)에서 직접 렌더링되는 페이지예요.
    * 우리가 CSS를 바꾸거나 UI를 수정할 권한은 없어요.
* 카카오에서 허용하는 건:

    * **앱 아이콘/이름/회사명** 정도 (→ [카카오 개발자 콘솔](https://developers.kakao.com)에서 수정 가능)
    * 색상, 버튼, 레이아웃은 **전부 고정**

👉 즉, 로그인 버튼이나 “동의 화면 전” 단계까지는 우리 UI로 꾸밀 수 있지만,
👉 “동의 화면” 자체는 카카오가 통제해요. (안전한 사용자 경험을 위한 정책)

---

## 2️⃣ 새 창(Popup)으로 띄우기

* 네, 가능합니다 ✅
* 카카오 로그인 URL을 그냥 `<a href>`로 연결하면 **현재 탭**에서 열려요.
* 대신 `window.open()`을 쓰면 새 창(팝업)으로 띄울 수 있습니다.

```javascript
const KAKAO_AUTH_URL = `https://kauth.kakao.com/oauth/authorize?response_type=code&client_id=${process.env.REACT_APP_KAKAO_REST_API_KEY}&redirect_uri=${process.env.REACT_APP_KAKAO_REDIRECT_URI}`;

function handleKakaoLogin() {
  window.open(KAKAO_AUTH_URL, "_blank", "width=500,height=600");
}
```

---

## 3️⃣ 새 창 → 동의 완료 후 Redirect 동작

여기서 중요한 부분이에요 👀

* 사용자가 **동의 버튼**을 누르면, 카카오 서버는 우리가 등록해둔 `redirect_uri`로 리다이렉트 시켜요.
* 그런데 이 리다이렉트는 **팝업 창 안에서만 일어납니다.**

    * 즉, 원래 있던 메인 페이지가 바뀌는 게 아니에요.
    * 팝업 창의 주소가 `redirect_uri?code=xxxx` 로 바뀌어요.

👉 그래서 보통은 이런 흐름을 씁니다:

1. 팝업 창 열기 (`window.open`)
2. 팝업 창에서 `redirect_uri`로 이동됨 → code 확보
3. 팝업 창에서 `window.opener.postMessage()` 같은 방식으로 부모창(원래 페이지)에 전달
4. 부모창에서 code를 받아 백엔드 호출 → JWT 발급
5. 팝업 창 닫기

---

## 📌 요약

* **CSS 수정:** 불가능 (카카오가 통제)
* **새 창으로 열기:** 가능 (`window.open`)
* **Redirect 동작:** 새 창 내부에서만 발생, 메인 페이지는 그대로

    * 메인 페이지를 바꾸려면 → 팝업이 부모창에 메시지를 전달해야 함



