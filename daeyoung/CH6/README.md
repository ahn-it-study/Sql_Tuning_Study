# 기본 DML 튜닝
## DML 성능에 영향을 미치는 요소
- 인덱스
- 무결성 제약
- 조건절
- 서브쿼리
- Redo 로깅
- Undo 로깅
- Lock
- 커밋
## 인덱스와 DML 성능
테이블에 레코드를 입력하면 인덱스에도 입력해야 하는데 테이블은 Freelist(테이블마다 데이터 입력이 가능한 블록 목록을 관리하는데, 이를 Freelist라고 한다.)를 통해 입력할 블록을 할당 받지만  
인덱스는 기본적으로 정렬된 자료구조 이므로 수직적 탐색을 통해 입력할 블록을 찾아야 합니다.  
즉, 인덱스에 입력하는 과정이 더 복잡해지므로 DML 성능에 미치는 영향도 크다.  
DELETE, UPDATE도 마찬가지입니다.  
DELETE의 경우 테이블에서 레코드를 하나 삭제하면 인덱스에서도 찾아서 삭제해주어야 하고 UPDATE의 경우는 테이블의 레코드가 변경이 되면 인덱스도 변경을 해주어야 하는데,  
이 경우 인덱스는 정렬이돼있어야 하기 때문에 재정렬의 추가 연산이 들어갑니다.  
인덱스 개수가 DML 성능에 미치는 영향이 큰 만큼 인덱스 설계를 잘해야하고 핵심 트랜잭션 테이블에서 인덱스를 하나라도 줄이면 TPS는 그만큼 향상됩니다.  
### 테스트 해보기
```sql
CREATE TABLE SOURCE as 
SELECT b.no, a.*
FROM (SELECT * FROM emp WHERE rownum <= 10) a
    ,(SELECT rownum as no FROM dual connect by level <= 100000) b;

CREATE TABLE target
as
SELECT * FROM SOURCE where 1 = 2;

ALTER TABLE target add
constraint target_pk primary key(no, empno);
```
SOURCE 테이블에는 레코드 100만개가 입력돼있다고 가정.  
TARGET테이블은 현재 비어있다. TARGET 테이블에서 PK 인덱스 하나만 생성한 상태에서 SOURCE 테이블을 읽어 레코드 100만개를 입력하면  
```sql
set timing on;
insert into target
select * from source;
```
책에서 4.95초 가량 걸린다고 합니다. 인덱스를 두개 더 생성하고 100만건을 입력한다고 가정하면
```sql
> truncate table target;
> create index target_x1 on target(ename);
> create index target_x2 on target(deptno, mgr);
> insert into target
  select * from SOURCE;
```
38.98초 가량 걸렸다고 나와있고 이전 결과에 비해 여덟배나 느려졌다는것을 볼 수 있었습니다.  
### 결론
설계를 잘하자, 변경이 많은 테이블에는 인덱스를 어떻게 설계할지 깊게 고민해야한다는것을 알 수 있었음
## 무결성 제약과 DML 성능
데이터베이스에 논리적으로 의미 있는 자료만 저장되게 하는 데이터 무결성 규칙으로는 아래 네 가지가 있습니다.   
### 개체 무결성
- 기본키는 null 값이 될 수 없음  
### 참조 무결성
- 외래키는 참조할 수 없는 값을 가질 수 없음  
### 도메인 무결성
- 특정 속성값은 그 속성이 정의된 도메인에 속한 값이어야 함  
### 사용자 정의 무결성
- 속성 값들이 사용자가 정의한 제약조건에 만족되어야 한다는 규정  
이들 규칙을 애플리케이션으로 구현할 수도 있지만, DBMS에서 PK, FK, Check, Not Null같은 제약을 설정하면 더 완벽하게 데이터 무결성을 지켜낼수있습니다.  
PK, FK 제약은 Check, Not Null 제약보다 성능에 더 큰 영향을 미친다고 합니다.  
Check, Not Null은 정의한 제약 조건을 준수하는지만 확인하면 되지만, PK, FK 제약은 실제 데이터를 조회해 봐야 하기 때문이다.  
### 일반 인덱스와 PK 제약을 모두 제거한 상태에서 100만건을 입력하는데 걸리는 시간 확인
```sql
> drop index target_x1;
> insert into target
  select * from SOURCE;
  truncate table target;
  /* - - - - - - - - - - - - - - - - - - - - - - - -  */
> drop index target_x2;
> insert into target
  select * from SOURCE;
  truncate table target;
  /* - - - - - - - - - - - - - - - - - - - - - - - -  */
> alter table target drop primary key;
> insert into target;
  select * from SOURCE;
```
![무제](https://user-images.githubusercontent.com/23313008/174824902-36d3ba53-42a6-456d-8c46-9fe449e386ff.png)  
## 조건절과 DML 성능
```sql
UPDATE emp SET sal = sal * 1.1 where deptno = 40;
DELETE from emp where deptno = 40;
```
SELECT 문과 실행계획이 다르지 않으므로 이들 DML문에는 인덱스 튜닝원리를 그대로 적용할 수있다고 합니다.  
## 서브쿼리와 DML 성능
```sql
update emp e set sal = sal * 1.1
where exists
    (select 'x' from dept where deptno=e.deptno and loc = 'CHICAGO');

delete from emp e 
where exists
    (select 'x' from dept where deptno = e.deptno and loc = 'CHICAGO');

insert into emp
select e.*
from emp_t e
where exists
    (select 'x' from dept where deptno = e.deptno and loc = 'CHICAGO');
```
이 또한 실행계획을 확인해보면 SELECT문과 다르지 않으므로 조인 튜닝 원리를 그대로 적용할 수 있습니다.  
(4장 4절의 서브쿼리 조인과 밀접한 관련이 있다고 합니다.)  
## Redo 로깅과 DML 성능
오라클은 데이터파일과 컨트롤 파일에 가해지는 모든 변경사항을 Redo 로그에 기록합니다.  
Redo 로그는 트랜잭션 데이터가 어떤 이유에서건 유실됐을 때, 트랜잭션을 재현함으로써 유실 이전 상태로 복구하는데 사용합니다.  
하지만 이 Redo 로그는 DML성능에 영향을 미친다고 하는데 이유는 DML을 수행할 때마다 Redo 로그를 생성해야 하기 떄문입니다.  
## Redo 로그의 용도?
### Database Recovery 
물리적으로 디스크가 깨지는 등의 Media Fail 발생 시 데이터베이스를 복구하기 위해 사용합니다.  
이때는 온라인 Redo 로그를 백업해 둔 Archived Redo 로그를 이용하게 됩니다.  
### Cache Recovery
Redo 로그는 캐시 복구를 위해 사용하며 다른 말로 Instance Recovery 라고도 합니다.  
모든 DBMS가 버퍼캐시를 도입하는 이유가 I/O 성능을 높이기 위해서인데, 버퍼 캐시는 휘발성이고 캐시에 저장된 변경사항이 디스크 상의 데이터 블록에 아직 기록되지 않은  
상태에서 정전 등이 발생해 인스턴스가 비정상적으로 종료되면 그때까지의 작업내용을 모두 잃게 된다는 뜻입니다.  
이러한 트랜잭션 데이터 유실에 대비하기 위해 Redo 로그를 남깁니다.  
### Fast Commit
Redo 로그는 Fast Commit을 위해도 사용하는데, 변경된 메모리 버퍼 쁠록을 디스크 상으로 반영하는 작업은 랜덤 액세스 방식이므로 매우 느립니다.  
반면 로그는 Append 방식으로 기록하므로 상대적으로 빠른데요. 따라서 트랜잭션에 의한 변경사항을 우선 Append 방식으로 빠르게 로그 파일로 기록하고 변경된 메모리 블록과  
데이터파일 블록 간 동기화는 적절한 수단을 이용해 나중에 배치 방식으로 일괄 수행하게됩니다.  
## Undo 로깅과 DML 성능
Redo 로그는 트랜잭션을 재현함으로써 과거를 현재 상태로 되돌리는 데 사용하고, Undo는 트랜잭션을 롤백함으로써 현재를 과거 상태로 되돌리는데 사용합니다.  
![무제](https://user-images.githubusercontent.com/23313008/174831622-7fca0d30-f065-43a5-ac39-b382d81b03a4.png)  
Undo에는 변경된 블록을 이전 상태로 되돌리데 필요한 정보를 로깅합니다.  
마찬가지로 DML를 수행할 때마다 Undo를 생성해야 하므로 Undo 로깅은 DML 성능에 영향을 미칩니다.  
그렇다고 Undo를 안남길 순 없고 오라클은 Undo 기능을 끌 수 있는방법을 아예 제공하지 않는다고 합니다. (강제!)  
## 언두 로그 용도
### Transaction Rollback
트랜잭션에 의한 변경사항을 최종 커밋하지 않고 롤백하고자 할 때 Undo 데이터를 이용합니다.  
### Transaction Recovery (Instance Recovery 시 rollback 단계)
Instance Crash 발생 후 Redo를 이용해 roll foward 단계가 완료되면 최종 커밋되지 않은 변경사항까지 모두 복구됩니다.  
따라서 시스템이 셧다운된 시점에 아직 커밋되지 않았던 트랜잭션들을 모두 롤백해야 하는데, 이때 Undo 데이터를 사용합니다.  
## MVCC
[MVCC](https://github.com/devdynam0507/MySQL-Study/blob/master/InnoDB_%EC%8A%A4%ED%86%A0%EB%A6%AC%EC%A7%80%EC%97%94%EC%A7%84_%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98.md)  
+ SCN   
## Lock과 DML 성능
Lock은 DML 성능에 매우 크고 직접적인 영향을 미칩니다.  
Lock을 필요 이상으로 자주, 길게 사용하거나 레벨을 높일수록 DML 성능은 느려집니.  
그렇다고 Lock을 너무 적게, 짧게 사용하거나 필요한 레벨 이하로 낮추면 데이터 품질이 나빠집니다.  
성능과 데이터 품질이 모두 중요한데 이 둘은 Trade-off 관ㅇ계여서 어렵습니다.  
두 마리 토끼를 다 잡으려면 매우 세심한 동시성 제어가 필요합니다.  
동시성 제어란 동시에 실행되는 트랜잭션 수를 최대화 하면서도 입력 수정 삭제 검색 시 데이터 무결성을 유지하기 위해 노력하는것을 말합니다.  
## 커밋과 DML 성능
DML이 Lock에 의해 블로킹된 경우 커밋은 DML 성능과 직결됩니다.  
DML을 완료할 수 있게 Lock을 푸는 열쇠가 커밋이기 때문입니다.  
모든 DBMS가 Fast Commit을 구현하고 있습니다. 구현방식은 서로 다르지만 갱신한 데이터가 아무리 많아도 커밋만큼은 빠르게 처리한다는 점은 같습니다.  
Fast Commit의 도움으로 커밋을 순간적으로 처리하긴 하지만, 커밋은 결코 가벼운 작업이 아닙니다.  
커밋의 내부 매커니즘을 통해 그 이유를 살펴봅시다.  
## DB 버퍼캐시
DB에 접속한 사용자를 대신해 모든 일을 처리하는 서버 프로세스는 버퍼캐시를 통해 데이터를 읽고 씁니다.  
버퍼 캐시에 변경된 블록을 모아 주기적으로 데이터파일에 기록하는 작업은 DBWR(Database Writer) 프로세스가 맡습니다.  
## Redo 로그버퍼
Redo 로그는 디비가 깨졌을 경우 복구하는데에 이용되고 Fast Commit에도 이용됩니다.  
Redo 로그도 Append 방식이라고는 하지만 결국 Disk I/O 이기 떄문에 느립니다.  
이를 해결하기 위해 오라클은 로그버퍼를 가지고 있는데 Redo 로그 파일에 기록하기전에 로그 버퍼에 먼저 기록하고  
LGWR 프로세스가 Redo 로그 파일에 일괄 기록합니다.  
## 트랜잭션 데이터 과정 
![무제](https://user-images.githubusercontent.com/23313008/174846435-da2b440a-60d3-43f3-b91c-9daade29c064.png)  
1. DML 문을 실행하면 Redo 로그버퍼에 변경사항을 기록한다.  
2. 버퍼블록에서 데이터를 변경(레코드 추가/수정/삭제)한다. 물론, 버퍼캐시에서 블록을 찾지 못하면, 데이터파일에서 읽는 작업부터 한다.  
3. 커밋한다.  
4. LGWR 프로세스가 Redo 로그버퍼 내용을 로그파일에 일괄 저장한다.?  
5. DBWR 프로세스가 변경된 버퍼블록들은 데이터파일에 일괄 저장한다.  
- 오라클은 데이터를 변경하기 전에 항상 로그부터 기록합니다.  
- DBWR 프로세스가 Dirty 블록을 디스크에 기록하기전에 LGWR 프로세스가 Redo 로그파일에 로그를 먼저 기록하는 이유이기도 합니다.(이를 Write Ahead Logging 이라고 합니다.)  
한가지 의문이 생길 수 있습니다. 메모리 버퍼 캐시가 휘발성이어서 Redo로그를 남기는데 Redo 로그 마저 휘발성 로그 버퍼에 기록한다면  
트랜잭션 데이터를 안전하게 지킬 수 있을까요?  
이는 잠자던 DBWR과 LGWR 프로세스는 주기적으로 깨어나 각각 Dirty 블록 (데이터가 변경이 된 블록)과 Redo 로그 버퍼를 파일에 기록합니다.  
LGWR 프로세스는 서버 프로세스가 커밋을 발행했다고 신호를 보낼 때도 깨어나서 활동을 시작합니다.  
적어도 커밋 시점에는 리두 로그버퍼 내용을 로그파일에 기록한다는 뜻입니다. 이를 Log Force at Commit 이라고 부릅니다.  
서버 프로세스가 변경한 버퍼블록들을 디스크에 기록하지 않았더라도 커밋 시점에 Redo 로그를 디스크에 안전하게 기록했다면,  
그 순간부터 트랜잭션의 영속성은 보장됩니다.  
## 커밋 = 저장 버튼
결론부터 말하자면 커밋은 Sync(동기) 방식입니다.  
문서를 작성할 때 저장 버튼을 누르면 내용이 디스크에 저장이 되고 워드 프로세서가 저장을 완료할 떄 까지 사용자는 아무런 작업을 할 수 없습니다.  
커밋도 이와 같이 워드프로세서의 저장 버튼과 같은 기능을 하는데요.  
커밋을 하게되면 서버 프로세스가 그때까지 했던 작업을 디스크에 기록하라는 명령을 하고 완료할 떄 까지 다음 작업을 진행할 수 없게됩니다.  
Redo 로그 버퍼에 기록된 내용을 디스크에 기록하도록 LGWR 프로세스에 신호를 보낸 후 작업을 완료했다는 신호를 받아야 다음 작업을 진행할 수 있습니다.  
하지만 LGWR 프로세스가 Redo 로그를 기록하는 작업은 디스크 I/O 작업이고 커밋은 그래서 생각보다 느립니다.  
트랜잭션을 필요 이상으로 길게 정의함으로써 오랫동안 커밋하지 않는 것도 문제지만, 너무 자주 커밋하는 것도 문제가 됩니다.  
오랫동안 커밋하지 않은 채 데이터를 계속 갱신하면 undo 공간이 부족해져 시스템 장애 상황을 유발할 수 있고 루프를 돌면서 간간히 커밋하면 시스템 자체 성능이 매우 느려집니다.  
그래서 트랜잭션을 논리적으로 잘 정의함으로써 불필요한 커밋이 발생하지 않도록 구현해야 합니다.  
## 데이터베이스 Call과 성능
![call](https://user-images.githubusercontent.com/23313008/174942988-f5f624de-b16e-431e-b6a3-aa4d2b0b936a.jpeg)  
SQL은 세단계로 나누어 실행됩니다.  
1. Parse Call: SQL문의 파싱과 최적화를 수행하는 단계 SQL과 실행계획을 라이브러리 캐시에서 찾으면 최적화 단계는 생략될 수 있습니다.  
2. Execute Call: SQL을 실행하는 단계입니다. DML의 경우 이 단계에서 끝나지만 SELECT문은 Fetch 단계를 거칩니다.  
3. Fetch Call: 데이터를 읽어서 사용자에게 결과집합을 전송하는 과정으로 SELECT 문에만 나타납니다. 전송할 데이터가 많을 경우 Fetch Call은 여러번 수행됩니다.  
Call이 어디서 발생하느냐에 따라 User Call과 Recursive Call로 나뉘어질 수 있습니다.  
![무제](https://user-images.githubusercontent.com/23313008/174943893-5419bfd9-786f-4e0e-8a5f-fff2e38d7ef3.png)  
## User Call
DBMS 외부로부터 유입되는 Call 입니다.  
대표적으로 WAS에서 보내는 콜이 User Call 입니다.  
## Recursive Call
DBMS 내부에서 발생하는 Call 입니다.  
대표적으로 SQL 파싱과 최적화 과정하는 데이터 딕셔너리 조회, PL/SQL로 작성한 사용자 정의 함수/프로시저/트리거에 내장된 SQL을 실행할 때 발생하는 Call이  
여기에 해당합니다.  
## 공통적으로
User Call이든 Recursive Call이든 SQL을 실행할 때마다 Parse, Execute, Fetch Call이 단계적으로 실행됩니다.  
Call이 많으면 당연히 성능은 느릴수 밖에 없고, 네트워크를 경유하는 User Call이 성능에 미치는 영향은 매우 큽니다.  
## 절차적 루프 처리
이전과 같이 source 테이블에 레코드 100만개가 입력돼 있다고 가정하고  
PL/SQL 프로그램에서 SOURCE 테이블을 읽어 100만번 루프를 돌면서 건건이 TARGET 테이블에 입력해보면  
**(참고: PL/SQL로 작성한 사용자 정의 함수, 프로시저/ 트리거에 내장된 SQL이 실행될 때 Recursive Call이 발생합니다.)**  
```sql
set timing on;

begin
    for s in (select * from source)
    loop
        insert into target values ( s.no, s.empno, s.ename, s.job, s.mgr, s.hiredate, s.sal, s.comm, s.deptno );
    end loop;

    commit;
end;
```
루프를 돌면서 건건이 Recursive Call이 발생했지만 네트워크를 경유하지 않는 Recursive Call이므로 그나마 29초만에 수행을 마쳤다고 합니다.  
### 번외: 커밋과 성능
```sql
set timing on;

begin
    for s in (select * from source)
    loop
        insert into target values ( s.no, s.empno, s.ename, s.job, s.mgr, s.hiredate, s.sal, s.comm, s.deptno );
        commit;
    end loop;
end;
```
위 쿼리에서 commit을 루프 안쪽으로 옮겨서 실행해보면 29초 걸리던게 1분으로 늘어났다고 합니다.  
**(참고: 커밋은 워드프로세서의 저장하기 같은 기능이라. 자주 수행하게 되면 커밋이 끝날때 까지 이전 작업을 못하므로 성능저하가 발생합니다)**  
성능도 문제이지만 이런식으로 자주 커밋하게되면 트랜잭션 원자성에도 문제가 생깁니다.  
반대로, 매우 오래 걸리는 트랜잭션을 한 번도 커밋하지 않고 진행해도 Undo 공간 부족으로 인해 시스템에 여러 부작용을 초래할 수 있습니다.  
원자성을 위해 반드시 그렇게 처리해야 한다면 Undo 공간을 늘려야 하지만, 그렇지 않다면 적당한 주기로 커밋하는 방안을 고려할 수 있습니다.  
루프 안쪽에 이런 코드를 삽입하면 됩니다.  
```sql
if mod(i, 10000) = 0 then
    commit;
end if;
```
이렇게 처리하면 맨 마지막에 한 번 커밋하는 것과 비교해 성능 차이가 크지 않다.  
## User Call 성능
```java

    public void execute() throws Exception {
        // ....
        final String SQL = "insert into ~";
        // 바인딩 코드
        PreparedStatement st = con.prepareStatement(SQL);
        st.execute();
        st.close();
        // ....
    }
```
위와같이 자바에서 쿼리를 실행했을 떄 (User Call) 218.4초 가량 걸린다고 합니다.  
위의 Recursive Call의 경우 그나마 네트워크를 경유하지 않아서 29초가량 걸렸지만 네트워크를 경유하는 경우 대력 8.5배 가량 성능이 더 안좋아지는것을 확인 할 수 있었습니다.  
## One SQL의 중요성
```sql
INSERT INTO target
SELECT * FROM SOURCE;
```
단 한번의 Call로 처리하게 되면 책에서는 1.46초 가량 걸린것을 확인할 수 있었습니다.  
위에 자바로된 User Call 예제에 비해 150배 가량 빨라진것을 확인할 수 있었습니다.  
업무 로직이 복잡하다면 절차적으로 처리할 수 밖에 없지만 그렇지 않다면 가급적 One SQL로 구현하려고 노력해야합니다.  
## Array Processing 활용
Array Processing의 핵심은 다량의 로우를 한번에 삽입할 수 있게 도와주는 기능입니다.  
예를들어 100만개의 데이터를 1만개씩 끊어서 100번만 삽입 합니다.  
이 기능을 이용한다면 One SQL로 구현하지 않고도 Call 부하를 획기적으로 줄일 수 있습니다.  
## 인덱스 및 제약 해제를 통한 대량 DML 튜닝
앞서 설명했듯 인덱스와 무결성 제약 조건은 DML 성능에 큰 영향을 끼칩니다.  
OLTP 환경에서는 이 기능을 해제할 순 없지만 배치 프로그램에서는 이 기능을 해제함으로써 큰 성능개선 효과를 얻을 수 있습니다.  
이번 테스트에서는 이전 보다 더 많은 데이터인 1000만건을 입력했다고 가정합니다.  
```sql
create table source
as
select b.no a.*
from (select * from emp where rownum <= 10) a
    ,(select rownum as no from dual connect by level <= 1000000) b;

create table target
as
select * from source where 1 = 2;

alter table target add
constraint target_pk primary key(no, empno);

create index target target_x1 on target(ename);
```
PK 제약을 생성하면 Unique 인덱스가 자동으로 생성된다. 추가로 일반 인덱스를 하나 더 만들었으므로 인덱스는 총 두개입니다.  
이 상태에서 1000만건을 입력한다면 PK 제약과 인덱스가 있는 상태에서 1분 19초가 걸렸다고 나와있습니다.  
### PK 제약과 인덱스 해제 1 - PK 제약에 Unique 인덱스를 사용한 경우
이번에는 PK 제약과 인덱스를 해제한 상태에서 입력해 보면  
```sql
truncate table target;
alter table target modify constraint target_pk disable drop index;
```
PK 제약을 비활성화하면서 인덱스도 Drop 했습니다.  
```sql
alter index target_x1 unusable;
```
이 상태에서 1000만건의 데이터를 집어넣어보면 5.84초만에 수행했다고 나와있습니다.  
임시로 PK 제약을 비활성화 한 뒤 데이터를 집어넣었잖아요?  
이후에 다시 원래대로 활성화를 시켜주면 모든 작업이 끝납니다.  
```sql
alter table target modify constraint target_pk enable NOVALIDATE;
```
NOVALIDATE 옵션은 기입력된 데이터에 대한 무결성 체크를 생략하도록 하는 옵션입니다.  
### skip_unusable_indexes
인덱스가 Unusable 상태에서 데이터를 입력하려면 skip_unusable_indexes 파라미터를 true로 설정해야 합니다.  
디폴트는 true 입니다.  
```sql
alter session set skip_unusable_indexes = true;
```
### PK 제약에 Non-Unique 인덱스를 사용한 경우
조금 전 테스트에서 X1 인덱스는 Unusable 상태로 변경했지만, PK 인덱스는 제약을 비활성화 하면서 아예 Drop 해버렸습니다.(PK 제약을 비활성화 할 떄 사용한 drop index 옵션 참조)  
PK 인덱스는 Unusable 상태에서 데이터를 입력할 수 없기 떄문입니다.  
PK 인덱스를 Drop 하지 않고 Unusable 상태에서 데이터를 입력하고 싶다면 PK 제약에 Non-Unique 인덱스를 사용하면 됩니다.  
```sql
set timing off;
truncate table target;
alter table target drop primary key drop index;

create index target_pk on target(no, empno); -- Non-Unique 인덱스 생성

alter table target add
constraint target_pk primary key(no, empno)
using index target_pk; --  PK 제약에 Non-Unique 인덱스를 사용하도록 지정
```
그리고 PK 제약을 비활성화하고, 인덱스를 Unusable 상태로 변경합니다.  
PK 제약을 비활성화했지만 인덱스는 Drop 하지 않고 남겨놨습니다.  
```sql
alter table target modify constraint target_pk disable keep index;

alter index target_pk unusable;

alter index target_x1 unusable;
```
그리고 똑같이 데이터를 insert 합니다.  
작업을 마친 이후 인덱스를 재생성하고 PK 제약을 다시 활성화 합니다.  
```sql
alter index target_x1 rebuild;

alter index target_pk rebuild;

alter table target modify constraint target_pk enable novalidate;
```
이렇게 하여도 기존 (1분 19초)보다 훨씬 더빨리 작업을 마칩니다.(책에서는 18초가 걸렸다고 나옵니다.)  
## 수정가능 조인 뷰
### 전통 적인 방식의 UPDATE
```sql
update 고객 c
set 최종거래일시 = (select max(거래일시) from 거래
                 where 고객번호 = c.고객번호
                 and 거래일시 >= trunc(add_months(sysdate, - 1)))
   ,최근거래횟수 = (select count(*) from 거래
                where 고객번호 = c.고객번호 and 거래일시 >= trunc(add_months(sysdate, -1)))
   ,최근거래금액 = (select sum(거래금액) from 거래
                 where 고객번호 = c.고객번호 and 거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
             where 고객번호 = c.고객번호
             and 거래일시 >= trunc(add_months(sysdate, -1)));
```
위 UPDATE문은 아래와 같이 바꿀 수 있습니다.  
```sql
update 고객 c
set (최종거래일시, 최근거래횟수, 최근거래금액) =
    (select max(거래일시), count(*), sum(거래금액)
    from 거래
    where 고객번호 = c.고객번호
    and rㅓ래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
              where 고객번호 = c.고객번호
              and 거래일시 >= trunc(add_months(sysdate, -1)));
```
바뀐 쿼리에도 비효율이 없는것은 아닙니다. 한달 이내 고객별 거래 데이터를 두 번 조회하기 때문입니다.  
이는 총 고객수와 한 달 이내 거래 고객 수에 따라 성능이 좌우됩니다.  
총 고객수가 아주 많다면 Exists 서브쿼리를 아래와 같이 해시 세미 조인으로 유도하는 것을 고려할 수 있습니다.  
```sql
update 고객 c
set (최종거래일시, 최근거래횟수, 최근거래금액) =
    (select max(거래일시), count(*), sum(거래금액)
    from 거래
    where 고객번호 = c.고객번호
    and rㅓ래일시 >= trunc(add_months(sysdate, -1)))
where exists (select /*+ unnest hash_sj */ 'x' from 거래
              where 고객번호 = c.고객번호
              and 거래일시 >= trunc(add_months(sysdate, -1)));
```
만약 한 달 이내 거래를 발생시킨 고객이 많아 UPDATE 발생량이 많다면, 아래와 같이 변경하는 것을 고려 할 수 있지만  
모든 고객 레코드에 LOCK이 걸리는 것은 물론 이전과 같은 값으로 갱신되는 비중이 높을수록 Redo 로그 발생량이 증가해 오히려 비효율적일 수 있습니다.  
```sql
update 고객 c
set (최종거래일시, 최근거래횟수, 최근거래금액) = 
    (select nvl(max(거래일시), c.최종거래일시), decode(count(*), 0, c.최근거래횟수, count(*)),
                nvl(sum(거래금액), c.최근거래금액)
    from 거래
    where 고객번호 = c.고객번호
    and 거래일시 >= trunc(add_months(sysdate, -1)))
```
이처럼 다른 테이블과 조인이 필요할 때 전통적인 UPDATE문을 사용하면 비효율을 완전히 해소할 수 없습니다.  
## 수정가능 조인 뷰
### 조인 뷰
from 절에 2개 이상의 테이블을 가진 뷰를 말합니다.  
  
처리 범위가 넓은(조인 테이블이 많은 경우?) UPDATE 문의 경우 WHERE 절이나 SET 절에서 테이블에 반복적으로 Random Access 해야하므로 수정가능 조인 뷰를 활용합니다.  
전통적인 UPDATE문은 여러번 조인을 해야 하는 비효율이 발생하지만(SET, WHERE 할 떄 Random Access 발생), 수정가능 조인뷰는 업데이트할 테이블을 넣는 대신에 바꿔야할 데이터만 들고와서 수정하는 쿼리 입니다.  
수정가능 조인 뷰는 1:M 관계에서 M 쪽 집합에만 수정이 가능하다는 특징이 있습니다.  
```sql
UPDATE(
    SELECT t1.r a, t2.c b
    FROM atable t1, btable t2
    WHERE t1.id = t2.id
) t
SET 
    t.a = 'aa',
    t.b = 'bb;
```
1쪽 집합과 조인되는 M쪽 집합(키 보존 테이블)에만 입력, 수정, 삭제가 가능한데, 키 보존 테이블에만 변경이 가능하다는 것입니다.  
하지만 쿼리가 실행되면서 어느쪽이 키 보존 테이블인지 알 수 없는 경우 (둘다 키 보존테이블이 아닌경우에)에 오류가(ORA-01779) 발생합니다.  
이 경우 한쪽 집합에 PK 제약을 설정하거나 unique 인덱스를 생성하야 수정가능 조인 뷰를 통한 DML 이 가능합니다.  
## 키 보존 테이블이란?
키 보존 테이블이란 조인된 결과집합을 통해서도 중복 값 없이 Unique하게 식별이 가능한 테이블을 말합니다.  
(뷰에 ROWID를 제공하는 테이블)  
조인되는 두 테이블중 1쪽 테이블에 유니크 인덱스를 제거하게 되면 키 보존 테이블이 없기떄문에 뷰에서 rowid 를 출력할 수 없게 됩니다.  
### ORA-01799 오류 회피
아래와 같이 dept 테이블에 AVG_SAL 컬럼을 추가해보겠습니다.  
```sql
alter table dept add avg_sal number(7, 2);
```
아래는 EMP로부터 부서별 평균 급여를 계산해서 방금 추가한 컬럼에 반영하는 UPDATE 문입니다.  
```sql
update
(select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
from (select deptno, round(avg(sal), 2) avg_sal from emp group by deptno) e, dept d
where d.deptno = e.deptno)
set d_avg_sal = e_avg_sal;
```
11g 이하 버전에서 위 UPDATE문을 실행하면 ORA-01779 에러가 발생합니다.  
EMP 테이블을 deptno로 group by 했으므로 DEPTNO 컬럼으로 조인한 DEPT 테이블은 키가 보존되는데도 옵티마이저가 불필요한 제약을 가한것입니다.  
이럴 때 10g에선 bypass_ujvc 힌트를 이용해 제약을 회피할 수 있었습니다.(11g부턴 사용할 수 없습니다.)  
수정가능 조인 뷰 검사를 생략하라고 옵티마이저에게 지시하는 힌트입니다.  
```sql
update /*+ bypass_ujvc */
(select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
from (select deptno, round(avg(sal), 2) avg_sal from emp group by deptno) e, dept d
where d.deptno = e.deptno)
set d_avg_sal = e_avg_sal;
```
11g부터는 위 힌트를 사용할 순 없고 MERGE 문으로 바꿔줘야합니다.  
수정가능 조인 뷰는 12c버전에서 오히려 기능이 개선됐습니다.  
즉, 12c부터는 힌트를 사용하지 않아도 위 UPDATE 문이 잘 실행됩니다.  
GROUP BY 한 집합과 조인한 테이블은 키가 보존된다는 사실을 오라클이 인정하기 시작한것입니다.  
이러한 기능 개선은 수정가능 조인 뷰의 활용성을 높여줍니다.  
예를들어 고객 t 테이블 고객번호에 유니크 인덱스가 없으면 아래 쿼리는 어떠한 버전에서도 실행할 수 없습니다.  
```sql
update (
    select o.주문금액, o.할인금액, c.고객등급
    from 주문_t o, 고객_t c
    where o.고객번호 = c.고객번호
    and o.주문금액 >= 100000
    and c.고객등급 = 'A'
)
SET 할인금액 = 주문금액 * 0.2, 주문금액 = 주문금액 * 0.8;
```
12c에서는 아래와 같이 고객_t 테이블을 Group By 처리함으로써 ORA-01779 에러를 회피할 수 있습니다.  
```sql
update (
    select o.주문금액, o.할인금액
    from 주문_t o
    , (select 고객번호 from 고객_t where 고객등급 = 'A' group by 고객번호) c
    where o.고객번호 = c.고객번호
    and o.주문금액 >= 1000000
)
set 할인금액 = 주문금액 * 0.2, 주문금액 = 주문금액 * 0.8;
```
배치 프로그램이나 데이터 이행 프로그램에서 사용하는 중간 임시 테이블에는 일일이 PK 제약이나 인덱스를 생성하지 않으므로  
이 패턴이 유용할 수 있습니다.  

결론적으로 뷰는 from 절에 여러개의 테이블이 있는 것이고 두 테이블을 조인하여 업데이트 할 때는 1쪽 테이블에 유니크 인덱스가 있어야 한다는 것입니다.  
## MERGE문 활용
MERGE 문은 UPSERT(UPDATE + INSERT)라고도 불리웁니다.  
이유는 Source 테이블 기준으로 Target 테이블과 Left Outer 방식으로 조인해서 조인에 성공하면 UPDATE 실패하면 INSERT 하기 때문입니다.  
![무제](https://user-images.githubusercontent.com/23313008/175480635-83653b8f-8372-44fd-909c-d9288676a87b.png)
busercontent.com/23313008/174943893-5419bfd9-786f-4e0e-8a5f-fff2e38d7ef3.png)  
### 예제
#### 1. 전일 발생한 변경 데이터를 기간계 시스템으로부터 추출
```sql
create table customer_delta
as
select * from customer
where mod_dt >= trunc(sysdate)-1
and mod_dt < trunc(sysdate);
```
#### 2. CUSTOMER_DELTA 테이블을 DW(데이터 웨어하우스) 시스템으로 전송
#### 3. DW 시스템으로 적재
```sql
merge into customer t using customer_delta s on (t.cust_id=s.cust_id)
when matched then update
    set t.cust_nm = s.cust_nm, t.email = s.email
when not matched then insert
    (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
    (s.cust_idm s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
```
### Optional Calueses
update와 insert를 선택적으로 처리할 수 있습니다.
```sql 
when matched then update
    set t.cust_nm = s.cust_nm, t.email = s.email
when not matched then insert
    (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
    (s.cust_idm s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
```
이 기능을 통해 아래와 같이 수정가능 조인 뷰 기능을 대체할 수 있게 되었습니다.
```sql
<수정가능 조인 뷰>
update
    (select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
    from (select deptno, round(avg(sal, 2)) avg_sal from emp group by dept no) e, dept d
    where d.deptno = e.deptno)
set d_avg_sal = e_avg_sal;

<Merge 문>
merge into dept d
using (select deptni, round(avg(sal), 2) avg_sal from emp group by deptno) e
on (d.deptno = e.deptno)
when matched then update set d.avg_sal = e.avg_sal;
```
### Conditional Operations
ON 절에 기술한 조인문 외에 추가로 조건절을 기술할 수도 있습니다.  
```sql
merge into dept d
using (select deptni, round(avg(sal), 2) avg_sal from emp group by deptno) e
on (d.deptno = e.deptno)
when matched then 
    update set d.avg_sal = e.avg_sal
    where d.avg_sag > 1000; <- 이부분
```
### DELETE Clause
이미 저장된 데이터를 조건에 따라 지우는 기능도 제공합니다.  
```sql
merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
when matched then
    update set t.cust_nm = s.cust_nm, t.email = s.email ...
    delete where t.withdraw_dt is not null -- 탈퇴일시가 null이 아닌 레코드 삭제
```
- 위 예시에서 UPDATE가 이루어진 결과로서 탈퇴일시가 Null이 아닌 레코드만 삭제합니다.(탈퇴일시가 Null이 아니었어도 MERGE문을 수행한 결과가 null이면 삭제하지 않습니다.)  
- 조인에 성공한 데이터만 삭제합니다.  
- 즉, Source 테이블에서 삭제된 데이터가 있어도 조인되지 못하면 Target 테이블에도 지우고 싶을텐데 그 역할까지는 못한다는 것입니다.  
### Merge Into 활용 예
저장하려는 레코드가 기존에 있던 것이면 UPDATE를 수행하고, 그렇지 않으면 INSERT 하랴고 합니다.  
그럴 때 아래와 같이 처리하면 SQL을 항상 두 번씩 (SELECT 한번, INSERT 또는 UPDATE 한번) 수행합니다.  
```sql
select count(*) into :cnt from dept where deptno = :val1;

if :cnt = 0 then
    insert into dept(deptno, dname, loc) values (:val1, :val2, :val3);
else
    update dept set dname = :val2, loc = :val3 where deptno = :val1;
end if;
```
아래와 같이 하면 SQL을 최대 두 번 수행합니다.  
```sql
update dept set dname = :val2, loc = :val3 where deptno = :val1;

if sql%rowcount = 0 then
    insert into dept(deptni, dname, loc) values(:val1, :val2, :val3);
end if;
```
하지만 아래와 같이 Merge 문을 사용하면 SQL을 한 번만 수행합니다.  
```sql
merge into dept a
using (select :val1 deptno, :val2 dname, :val3 loc from dual) b
on (b.deptno = a.deptno)
when matched then
    update set dname = b.dname, loc = b.loc
when not matched then
    insert (a.deptno, a.dname, a.loc) values (b.deptno, b.dname, b.loc).
```
