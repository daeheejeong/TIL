## SQL Bulk-Insert 성능에 대하여

### 내용 정리에 앞서

최근 MyBatis에서 배치타입(`ExcutorType.BATCH`) 세션을 통해 대량의 row를 일괄 insert할 때의 성능개선에 대한 글을 작성하였습니다. ([batch-type-session](https://github.com/Daehee-Jeong/TIL/blob/master/MyBatis/batch-type-session.md "대용량 엑셀파일(약1만 건 이상) 업로드 기능 개선을 위해 확인한 내용"))

해당 과정을 통해 저는 약1만 8천건의 row를 갖는 엑셀파일을 테이블에 insert할 때
소요시간을 약5분에서 20초대 까지 감소시킬 수 있었습니다.

하지만 사용자 입장에서 어떠한 기능이 처리되기까지에 UI단이 멈춘채로 20초 가량을 대기해야 한다는 것이 타당성이 많이 부족하다고 생각하여 조금더 성능 개선을 해보고싶었습니다.
(물론 UI단이 멈추지 않도록 프로세스를 바꿀 수는 있겠지만요)

### 성능개선 retry !

배치타입 세션을 이용해서 1차적 성능 개선을 한 뒤 생각해보니 약1만 건의 insert에 대해 당연히 매번 SqlSession을 create하는 소요는 불필요한 것이 맞다고 생각했습니다.

하지만 이후에는 어떤 부분을 개선할 수 있을지 많이 고민했는데요.

무언가 일괄로 처리할때 MyBatis를 사용하시는 분들은 많이 알고있는 `<foreach>`문법이 있습니다.

Query XML내에 foreach 구문을 사용해 일정 구문을 반복할 수 있는데 이는 결국 Java로 Loop을 수행하며 PreparedStatment를 빌딩하며 매 파라미터를 맵핑하는 소요를 발생시킵니다.

만일 이기능 만을 이용하여 약만건의 데이터를 모두 처리한다면 성능 문제는 물론 애초에 **DB Spec에 따라서** raw하게 들어가는 **파라미터들에 대한 갯수제한**이 걸리게 되구요
(저는 현재 회사에서 Oracle 10g DB 기준으로 IN절 파라미터 1000개 제한 등의 문제를 경험해보았습니다.)

그런데 또 생각을 해보면

```sql
INSERT INTO [테이블]
    VALUES ([컬럼1 값], [컬럼2 값])
```
위의 단일 건 삽입구문을 전체 row 만큼 반복하는 것 보다는

```sql
INSERT ALL
    INTO [테이블] ([컬럼1 값], [컬럼2 값])
    INTO [테이블] ([컬럼1 값], [컬럼2 값])
    INTO [테이블] ([컬럼1 값], [컬럼2 값])
    ...
SELECT *
FROM DUAL;
```
위 처럼 n개의 row를 bulk로 insert했을시에는 수행 구문의 개수도 (전체 row 수)/n 으로 줄어들며 성능 개선이 있을 거라고 생각했습니다.

### 어떻게 해야하지 ..?

배치타입 session과 bulk-insert 구문을 혼용해서 조금 더 성능개선을 하고싶은데 감을 못잡고 있다가 구글에서 다음과 같은 사진을 찾게 되었습니다 !

![2, 7, 10, 23개의 컬럼을 가진 테이블에 n개씩 한 묶음으로 bulk-insert수행 시 속도 비교](/images/bulk-insert-performance-01.png)
>출처: <https://www.red-gate.com/simple-talk/sql/performance/comparing-multiple-rows-insert-vs-single-row-insert-with-three-data-load-methods/>

내용은 즉, 컬럼의 갯수별로 n개씩 bulk insert를 함에 있어서 가장 효율적인 n값의 구간대가 존재 한다는 것입니다.

저의 경우에는 컬럼 3개를 보유한 테이블이었고 이에 따라 18000번의 insert가 수행되던 것을

**18000 / 50(10~100개 사이 최적의 숫자로 선정) = 360번의 bulk insert수행, 매 insert마다 50개의 row를 한번에 삽입**

위와 같이 바꾸고 다시 실행을 한 결과 소요시간이

최초 5분 -> 약20초(배치타입 Session 적용) -> **6초(배치타입 Session + bulk insert 적용)**

으로 훨씬 더 개선된 성능을 얻을 수 있었습니다.

DB Spec에 따라 위의 표가 신뢰도가 달라질 수도 있겠지만, 저도 구간별로 테스트를 해본결과 어느 정도 신뢰를 할 수 있는 자료라고 판단을 하였습니다.

결과 적으로 현재는 Admin에서 현업자들이 엑셀로 일괄 업로드를 하는 기능들에 대해 제가 정리한 내용을 일괄적으로 적용을 하였고, 현재는 기능을 원활히 사용중입니다.

### 끝으로
있어도 느려서 쓰지 않는다는 기능에 대해 무조건적인 성능개선을 목표로 이것저것 찾아보고 적용하게 된 내용이었습니다. 혹시나 제 글에 대해 잘못된 부분이 있다면 언제든 PR주시면 정말 감사하겠습니다. 😀

___

>**수정이력**  
2019.08.11 최초작성