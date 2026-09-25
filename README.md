# spring-cgv-24th
CEOS 24기 백엔드 스터디 - CGV 클론 코딩 프로젝트

## ERD
![erd.png](erd.png)
[ERD Cloud 가서 보기](https://www.erdcloud.com/d/aBzYHMExGpfHZXSEE)

- 테이블들의 PK값은 변하지 않도록 별개의 단일 id를 만드는 방식으로 통일했습니다.
  - `@GeneratedValue(strategy = GenerationType.IDENTITY)`
  - 복합키도 사용할 수 있지만, 나중에 로직 단의 코드가 복잡해질 것이라 생각했는데, 리뷰어 분들은 어떻게 생각하시는지 궁금합니다!
- 테이블을 짤 때 가장 고민이 많았던 부분은 예매 관련 부분이었습니다.
  - 현재의 구조에서 예매와 관련된 테이블들은 다음과 같습니다.
    - `Booking`: 예매
    - `BookingSeat`: 예매 시 선택한 좌석(들)
    - `Showtime`: 상영일정, 즉 어떤 영화가 어떤 상영관에서 언제 상영되는지
  - 여기서 `Booking`과 `BookingSeat`가 연결되어 있고, `Booking`과 `Showtime`이 연결되어 있습니다.
  - 하지만 여기서 문제가 발생합니다!
    - 현재의 `BookingSeat`는 행과 열을 저장하지만, 상영일자(`Showtime`)를 저장하지는 않습니다. 그렇기 때문에 DB 단에서 (showtime_id, row_no, column_no)를 한번에 UNIQUE로 처리하지 못합니다.
    - 물론 로직 상으로 처리할 수는 있지만 DB 단에서는 중복예매가 가능하다는거죠.
    - 여기서 제가 생각해본 해결방법은 2가지 였습니다.
      1. `BookingSeat`와 `Showtime` 연결
         → 하지만 이 방식을 사용하면 연결이 돌고 돌아 Booking의 showtime_id와 BookingSeat의 showtime_id이 불일치 할 수 도 있는 상황이 발생합니다.
      2. `BookingSeat`와 `Showtime` 사이에 `ShowtimeSeat` 생성
         → 이 방법은 DB 상으로는 나름..? 깔끔해보이지만 ShowtimeSeat라는게 상영일자가 새로 생길때마다 모든 좌석에 대해 데이터가 생겨야 한다는 번거로움 + 불필요한 데이터 추가 라는 문제가 있습니다.
    - 코드리뷰를 통해, 1번 방식을 보충하는 방향으로 코드를 수정해보았습니다. 설계의 흐름을 정리하면 다음과 같습니다.
      1. **애플리케이션 레벨에서 불일치 방지**
         <br>: BookingSeat의 showtime을 외부에서 직접 세팅하지 않고, `Booking.addSeat()`를 통해서만 생성하도록 했습니다. 이때 `BookingSeat.showtime`은 항상 `Booking.showtime`에서 가져옵니다.
      2. **DB 레벨에서도 불일치 방지**
         <br>: 애플리케이션 코드만으로는 완전한 보장이 어렵기 때문에 `(booking_id, showtime_id)`를 복합 FK로 묶어 두 테이블의 상영 정보가 반드시 일치하도록 하였습니다.
      3. **좌석 중복 예매는 DB UNIQUE로**
         <br>: 같은 showtime + row + column에 활성 예약이 두 개 이상 생기지 못하도록 DB 제약을 두었습니다. 동시 요청이 들어와도 DB가 최종적으로 한 건만 허용하도록 합니다. 
      4. **취소 이력을 남기기 위해 상태값 사용**
      5. **MySQL에서는 active_seat generated column 사용**
         <br>: BOOKED이면 active_seat = 1, 이 외의 상태면 NULL이 되도록 DB가 자동 계산하게 하였습니다. 이후 `UNIQUE(showtime_id, row_no, column_no, active_seat)`를 걸어 현재 점유 중인 좌석만 중복을 막고, 취소/만료 이력은 여러 건 남길 수 있도록 하였습니다. 
      6. **현재 개발 단계에서는 schema.sql 사용**
         <br>: 아직 ddl-auto: create를 유지하고 있으므로 Hibernate가 테이블을 생성한 뒤 schema.sql에서 복합 FK, generated column, UNIQUE INDEX와 같이 코드 상으로 직접 추가하기 번거로운 사항들에 대해 추가 제약을 적용하고, data.sql에서는 더미 데이터만 넣도록 역할을 분리했습니다.

## 구현 코드
- 도메인 별로 개발을 해서 도메인형 구조로 코드를 짜보았습니다. 도메인 별로 개발을 하는데, `controller/`, `service/` 이런 식으로 계층형 구조를 쓰게 되면, 한 도메인과 관련된 코드를 쓰기 위해 모든 계층의 폴더를 열고 닫아야 하는것이 싫어서... 선택해보았습니다. 이 방법보다 계층형이 더 좋다! 하시는 분들의 의견도 궁금합니다.
- `createdAt` 필드가 중복되는 곳이 몇군데 있어서 `BaseEntity`로 JpaAuditing 기능을 분리하였습니다.
- 급하게 짠 코드들이 있어서... 아직 코드가 지저분합니다. 많은 잔소리와 지적... 부탁드립니다