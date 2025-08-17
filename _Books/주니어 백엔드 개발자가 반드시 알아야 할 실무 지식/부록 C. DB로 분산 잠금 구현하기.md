# DB로 분산 잠금 구현하기

동시에 두 개 이상의 프로세스가 실행되더라도 그중 하나의 프로세스, 하나의 스레드만 작업을 실행해야 한다. 이러한 요구 사항을 만족하려면 분산 잠금이 필요하다.

레디스나 주키퍼 같은 기술을 사용할 수도 있지만 구조를 단순하게 유지하기 위해 DB를 사용한 예시

<br/>

## 1. 잠금 정보 저장 테이블

 - `분산 잠금을 구현하기 위한 DB 테이블 구조`
    - name: 개별 잠금을 구분하기 위한 값으로, 주요 키이다.
    - owner: 잠금 소유자를 구분하기 위한 값으로, 여러 스레드가 같은 이름의 잠금을 시도할 떄 충돌을 처리한다.
    - expiry: 잠금 소유 만료 시간으로, 한 소유자가 오랜 시간 잠금을 소유하지 못하도록 한다.

<br/>

## 2. 분산 잠금 동작


```
1. 트랜잭션 시작
2. 선점 잠금 쿼리(for update)를 이용해 해당 행을 점유
3. 행이 없으면 잠금 테이블에 새로운 데이터 추가
4. owner가 다른데 아직 expiry가 지나지 않았다면, 잠금 획득 실패
5. owner가 다른데 expiry가 지났다면, owner와 expiry 값을 변경한 후 잠금 획득
6. owner가 같다면 expiry만 갱신한 후 잠금 획득
7. 트랜잭션을 커밋하고 소유 결과 리턴
8. 트랜잭션 커밋에 실패하면 잠금 획득도 실패
```
<br/>

## 3. DB 잠금 구현

```java
// 잠금 소유자를 표현하는 LockOwner
public record LockOwner(String owner, LocalDateTime expiry) {
    // 현재 소유자가 누구인지 비교
    public boolean isOwnedBy(String owner) {
        return this.owner.equals(owner);
    }
    // 잠금이 만료됐는지 여부 확인
    public boolean isExpired() {
        return expiry.isBefore(LocalDateTime.now());
    }
}

// 실제 잠금 로직을 구현하는 DistLock
@AllArgsConstructor
public class DistLock {
    private final DataSource dataSource;

    public boolean tryLock(String name, String owner, Duration duration) {
        Connection conn = null;
        boolean owned;

        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);

            // 동시 실행을 막기 위해 "select for update" 쿼리 실행
            LockOwner lockOwner = getLockOwner(conn, name);
            if (lockOwner == null || lockOwner.owner() == null) {
                // 아직 소유자가 없음 - 잠금 소유 시도
                insertLockOwner(conn, name, owner, duration);
                owned = true;
            } else if (lockOwner.isOwnedBy(owner)) {
                // 소유자 같음 - 만료 시간 연장
                updateLockOwner(conn, name, owner, duration);
                owned = true;
            } else if (lockOwner.isExpired()) {
                // 소유자 다름 && 만료 시간 지남 - 잠금 소유 시도
                updateLockOwner(conn, name, owner, duration);
                owned = true;
            } else {
                // 소유자 다름 && 만료 시간 안 지남 - 잠금 소유 실패
                owned = false;
            }
            conn.commit();
        } catch (Exception e) {
            owned = false;
            rollback(conn);
        } finally {
            close(conn);
        }
        return owned;
    }

    private LockOwner getLockOwner(Connection conn, String name) throws SQLException {
        String query = "select * from dist_lock where name = ? for update";

        try (PreparedStatement pstmt = conn.prepareStatement(query)) {
            pstmt.setString(1, name);
            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return new LockOwner(
                        rs.getString("owner");
                        rs.getTimestamp("expiry").toLocalDateTime();
                    )
                }
            }
        }

        return null;
    }

    private void insertLockOwner(
        Connection conn,
        String name,
        String ownerId,
        Duration duration
    ) throws SQLException {
        String query = "insert into dist_lock values (?, ?, ?)";
        try (PreparedStatement pstmt = conn.prepareStatement(query)) {
            pstmt.setString(1, name);
            pstmt.setString(2, ownerId);
            pstmt.setTimestamp(3, getExpiry(duration));
            pstmt.executeUpdate();
        }
    }

    private static Timestamp getExpiry(Duration duration) {
        return Timestamp.valueOf(
            LocalDateTime.now().plusSeconds(duration.getSeconds())
        );
    }

    private void updateLockOwner(
        Connection conn,
        String name,
        String ownerId,
        Duration duration
    ) throws SQLException {
        String query = "update dist_lock set owner = ?, expiry = ? where name = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(query)) {
            pstmt.setString(1, owner);
            pstmt.setTimestamp(2, getExpiry(duration));
            pstmt.setString(3, name);
            pstmt.executeUpdate();
        }
    }

    public void unlock(String name, String owner) {
        Connection conn = null;
        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);
            LockOwner lockOwner = getLockOwner(conn, name);
            if (lockOwner == null || !lockOwner.isOwnedBy(owner)) {
                // 잠금 소유자가 아닌 경우
                throw new IllegalStateException("no lock owner");
            }
            if (lockOwner.isExpired()) {
                // 잠금 소유자 같음 && 만료시간 지남
                throw new IllegalStateException("lock is expired");
            }
            clearOwner(conn, name);
            conn.commit();
        } catch (SQLException e) {
            rollback(conn);
            throw new RuntimeException("fail to unlock: " + e.getMessage());
        } finally {
            close(conn);
        }
    }

    private void clearOwner(
        Connection conn,
        String name
    ) throws SQLException {
        String query = "update dist_lock set owner = null, expiry = null where name = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(query)) {
            pstmt.setString(1, name);
            pstmt.executeUpdate();
        }
    }

    private void rollback(Connection conn) {
        if (conn != null) {
            try {
                conn.rollback();
            } catch (SQLException ex) {}
        }
    }

    private void close(Connection conn) {
        if (conn != null) {
            try {
                conn.setAutoCommit(false);
            } catch (SQLException ex) {}

            try {
                conn.close();
            } catch (SQLException e) {}
        }
    }
}
```
