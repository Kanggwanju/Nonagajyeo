## 🛠️ 트러블슈팅: JPA 네이티브 쿼리에서 Enum 타입 바인딩 문제

-----

### 📌 상황

모임(Meeting)에 참여한 사용자의 목록을 \*\*참여 상태(ApplicationStatus)\*\*별로 조회하는 `getParticipatedMeetings` API를 구현하고 있었다. `status` 파라미터를 `ApplicationStatus` Enum 타입으로 받아 JPA의 **네이티브 쿼리**를 사용해 데이터를 조회하는 방식이었다.

### 📌 문제

**1. Enum 타입이 숫자로 변환되어 쿼리 실행**

`http://localhost:9000/api/meetings/participated?page=0&size=2&status=PENDING`처럼 `PENDING`이라는 문자열을 요청 파라미터로 보냈는데, JPA가 Enum의 이름(`'PENDING'`)이 아닌 Enum의 순서(`0`)로 인식하여 DB에 `application_status = 0`이라는 쿼리를 보냈다.
\!

**2. 타입 불일치로 인한 조회 실패**

`MeetingParticipant` 엔티티에서는 `@Enumerated(EnumType.STRING)` 어노테이션을 사용하여 DB에 Enum 값을 문자열로 저장하고 있었다. 따라서 DB의 `application_status` 컬럼에는 `'PENDING'`이나 `'APPROVED'`와 같은 문자열이 저장되어 있었기 때문에, `application_status = 0`이라는 쿼리 조건은 항상 `false`가 되어 아무런 데이터도 조회되지 않았다.

-----

### 📌 해결

JPA가 네이티브 쿼리 파라미터를 잘못 변환하는 문제를 해결하기 위해, Enum 값을 리포지토리로 전달하기 전에 **서비스 레이어에서 명시적으로 문자열로 변환**하도록 코드를 수정했다.

### ✅ 수정된 코드

#### `FindMeetingService.java`

서비스 레이어에서 `.name()` 메서드를 사용해 `ApplicationStatus` Enum을 문자열로 변환하고, 이를 리포지토리 메서드에 전달한다.

```java
// FindMeetingService.java

public MeetingProfilePageResponse getParticipatedMeetings(
    String providerId,
    MeetingParticipant.ApplicationStatus status,
    Pageable pageable) {

    log.info("참여 상태 모임 목록 조회 서비스 메서드 - 페이징: {}", pageable);

    User user = usersRepository.findByProviderId(providerId)
        .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

    // ✅ status.name()으로 Enum을 문자열로 변환
    Page<Meeting> meetingPage = meetingsRepository.findMeetingsWithMartByUserIdAndStatus(
        user.getUserId(),
        status.name(),
        pageable
    );
    ...
}
```

-----

#### `MeetingsRepository.java`

네이티브 쿼리 파라미터의 타입을 `String`으로 변경하여, 서비스 레이어에서 받은 문자열을 그대로 사용하도록 한다. 이렇게 하면 JPA가 자동으로 Enum을 숫자로 변환하는 것을 방지할 수 있다.

```java
// MeetingsRepository.java

@Query(value = """
    SELECT m.*
    FROM meetings m
    JOIN meeting_participants mp ON m.meeting_id = mp.meeting_id
    JOIN marts ma ON m.mart_id = ma.mart_id
    WHERE mp.user_id = :userId AND mp.application_status = :status_name
        AND m.deleted_at IS NULL
    ORDER BY m.created_at DESC
""",
countQuery = """
    SELECT count(m.meeting_id)
    FROM meetings m
    JOIN meeting_participants mp ON m.meeting_id = mp.meeting_id
    WHERE mp.user_id = :userId AND mp.application_status = :status_name
        AND m.deleted_at IS NULL
""",
nativeQuery = true)
Page<Meeting> findMeetingsWithMartByUserIdAndStatus(
    @Param("userId") Long userId,
    @Param("status_name") String statusName, // ✅ String 타입으로 변경
    Pageable pageable
);
```