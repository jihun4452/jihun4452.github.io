---
layout: post
title: "DataBase001"
date: 2024-10-27
categories: [데이터 베이스 공부]
---

# H2 DataBase를 기반으로 실습합니다.
참고: 김영한의 DB 1편

영상을 보면서 실습을 했을때 이해가 가지않는 부분이 많았지만 일단 끄적여보겠다.
처음에 H2 데이터베이스를 하는데 연결이 안돼서 애좀 먹었다. 이 부분은 오류 카테고리에 적어놓도록 하겠다.

일단 실습 첫번째 소스 코드만 적어두고 나머지는 코드 안에 주석으로 설명하겠다.

# ConnectionConst 코드
```java
package solo.database.connection;

public abstract class ConnectionConst {
    public static final String URL = "jdbc:h2:tcp://localhost/~/test";
    public static final String USERNAME = "sa";
    public static final String PASSWORD = "";
}
```
# DBConnectionUtil
```java
package solo.database.connection;

import lombok.extern.slf4j.Slf4j;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

import static solo.database.connection.ConnectionConst.*;

@Slf4j
public class DBConnectionUtil {
    public static Connection getConnection() {
        try {
            Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
            log.info("get connection={}, class={}", connection,
                    connection.getClass());
            return connection;
        } catch (SQLException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

# TestCode // DBConnectionUtilTest
```java
package solo.database.jdbc;

import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import solo.database.connection.DBConnectionUtil;

import java.sql.Connection;
import static org.assertj.core.api.Assertions.assertThat;

@Slf4j
class DBConnectionUtilTest {
    @Test
    void connection() {
        Connection connection = DBConnectionUtil.getConnection();
        assertThat(connection).isNotNull();
    }
}
```

이렇게 했을 때 H2데이터베이스와 연결이 되어야한다. 실행을 시켜보면 테스트결과 확인 완료시 성공이다.

다음 도메인과 , 레포를 만들어보도록 하자

멤버는 일전에 사용했던 변수들을 사용해주겠다.

# Member
```java 
package solo.database.jdbc.domain;

import lombok.Data;

@Data
public class Member {
    private String memberId;
    private int money;

    public Member(){

    }
    public Member(String memberId, int money) {
        this.memberId = memberId;
        this.money = money;
    }
}

```

레포도 생성 해주겠다.

# MemberRepositoryV0
```java
package solo.database.jdbc.repository;

import lombok.extern.slf4j.Slf4j;
import solo.database.connection.DBConnectionUtil;
import solo.database.jdbc.domain.Member;

import java.sql.*;

@Slf4j
public class MemberRepositoryV0 {
    public Member save(Member member) throws SQLException {
        String sql="insert into member(member_id,money) values(?,?)"; //파라미터 바인딩 방식으로 하면 단순하게 데이터 취급만되고
        //만약 그 방법이 아니라면 세세하게 호출을 할 수 있어 해킹 당할 위험이 있다.
        //sql을 전달해주어야 해서 인서트문으로 준비

        Connection con=null;
        PreparedStatement pstmt=null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            // (?,?) 처음은 1번이라 1번 할당, 두번째는 String 타입이라 setString 으로 지정해주었다.
            pstmt.setInt(2, member.getMoney());
            // (?,?) 은 두번째라 2번, 두번째는 Int타입이라 IntString 으로 지정.
            pstmt.executeUpdate();
            // SQL 커넥션을 통해 디비에 전달, 이것은 Int를 반환한다. 영향반은 DB row수를 반환, 하나의 row를 등록해서 1을 반환해준다.
            // 역량받은 수만큼 돌려보내준다.
            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }
    private void close(Connection con, Statement stmt, ResultSet rs) {
        if (rs != null) {
            try {
                rs.close();// 여기서 sql익셉션이 터져도 캐치로 잡아버린다.
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
        if (stmt != null) {
            try {
                stmt.close();
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
        if (con != null) {
            try {
                con.close();
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
    }

    //사용한 리소스들은 모두~~ 다  닫아주어야함
    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
}


``` 
설명들은 다 적어놨다 

주의
리소스 정리는 꼭! 해주어야 한다. 따라서 예외가 발생하든, 하지 않든 항상 수행되어야 하므로 finally 구문에
주의해서 작성해야한다. 만약 이 부분을 놓치게 되면 커넥션이 끊어지지 않고 계속 유지되는 문제가 발생할 수 있
다. 이런 것을 리소스 누수라고 하는데, 결과적으로 커넥션 부족으로 장애가 발생할 수 있다.
참고
PreparedStatement 는 Statement 의 자식 타입인데, ? 를 통한 파라미터 바인딩을 가능하게 해준다.
참고로 SQL Injection 공격을 예방하려면 PreparedStatement 를 통한 파라미터 바인딩 방식을 사용해야
한다.

이것을 꼭 기억하고 실천하도록 하자.
