# 🛠️ 트러블슈팅: OAuth2 Success Handler에서 JWT subject로 `providerId`를 사용한 문제

## 📌 상황

OAuth2 인증이 성공하면 `OAuth2SuccessHandler`에서 JWT 토큰을 발급하는 로직을 작성했다.
처음에는 토큰의 식별자(subject)에 \*\*소셜 로그인 providerId(예: 카카오 user id)\*\*를 사용했다.

```java
String token = jwtTokenProvider.createToken(oAuth2User.getProviderId(), oAuth2User.getRole());
```

---

## 📌 문제

1. **외부 서비스 종속성**

    * `providerId`는 카카오, 구글 등 외부 서비스에서 발급하는 값이다.
    * 동일 사용자가 카카오/구글을 동시에 로그인하면, 우리 DB에서는 한 명인데 providerId는 서로 다르다.
    * 결과적으로 동일 사용자가 **중복 계정처럼 보이는 문제** 발생.

2. **DB 조회 비효율**

    * 토큰에서 providerId만 가지고 있으면 우리 시스템의 User 엔티티를 직접 식별할 수 없다.
    * 매번 `findByProviderId(...)`로 조회해야 해서 코드가 복잡하고, 확장성도 떨어진다.

---

## 📌 해결

JWT 토큰의 subject로 \*\*우리 DB의 내부 User PK(userId)\*\*를 사용하도록 수정했다.
providerId는 단순히 "소셜 로그인 유저를 찾는 용도"로만 사용하고,
JWT 안에서는 우리 서비스에서 관리하는 userId를 넣는 방식으로 변경했다.

### ✅ 수정된 코드

#### OAuth2SuccessHandler

```java
User user = userRepository.findByProviderIdAndProvider(
        oAuth2User.getProviderId(), 
        oAuth2User.getProvider()
).orElseThrow(() -> new RuntimeException("회원가입이 필요합니다."));

String token = jwtTokenProvider.createToken(user.getId().toString(), user.getRole());
```

#### JwtTokenProvider

```java
public String createToken(String userId, String role) {
    Claims claims = Jwts.claims().setSubject(userId); // ✅ 내부 userId 사용
    claims.put("role", role);

    Date now = new Date();
    Date validity = new Date(now.getTime() + validityInMilliseconds);

    return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(SignatureAlgorithm.HS256, secretKey)
            .compact();
}
```

---

## 📌 결과

* JWT 토큰에서 바로 `userId`를 꺼내 DB 조회 가능.
* 로그인 방식(카카오, 구글, 자체 회원가입 등)에 상관없이 동일한 유저를 일관되게 식별.
* 서비스가 외부 providerId에 **종속되지 않고 독립성 확보**.

---

## 📌 배운 점

* **토큰의 subject는 우리 서비스 기준으로 일관된 식별자를 사용해야 한다.**
* 외부 로그인 식별자는 "로그인 수단"일 뿐, 서비스 내부의 유저 식별자로 사용하면 안 된다.
* 확장성, 유지보수성, 보안 측면에서 모두 `userId` 사용이 더 안전하다.

