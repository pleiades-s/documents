# SQLite_API

## 데이터베이스 연결 객체와 명령 객체 

- 데이터베이스 연결 객체: 특랜잭션상태에서 데이터베이스와 연결된 객체
- 데이터베이스 명령 객체: 연결 객체로부터 파생된다. 모든 명령 객체는 하나의 연관된 연결된 객체를 갖는다. 명령 객체는 컴파일된 SQL 명령 하나를 나타내며, 내부적으로는 VDBE 바이트 코드의 형태로 표현된다. 
  - 바이트 코드, B-tree 커서, 바운드 매개변수
  
  
## 핵심 API
- 데이터베이스 연결과 관련된 부분은 생략한다.

### Prepared 쿼리의 수행
 API와 내부 모두에서 SQLite 가 모든 명령을 실행하는 표준이면서 실질적인 방법이 prepared 쿼리로 준비, 실행, 마무리의 세 단계로 처리되며 각각의 단계는 연관된 API를 제공한다. 
- **준비 (preparation - sqlite3_prepare)**: SQL 문을 VDBE 바이트 코드로 컴파일하여 실행 준비를 함. `sqlite3_stmt` 핸들을 생성하는데 해당 핸들은 바이트 코드 및 명령 실행에 필요한 모든 자원을 가진다.
- **실행 (execution - sqlite3_step)**:  VDBE 가 단계쩍으로 바이트 코드를 실행한다. `sqlite3_step()` 함수를 통해 바이트 코드를 하나씩 훑어간다. *SELECT* 명령을 수행할 때에는 `sqlite3_step()` 을 호출할 때마다 명령 핸들의 커서를 결과 셋의 다음 행에 위치시킨다. 이 때 커서가 결과 셋의 끝에 도달하지 않은 경우에는 `SQLITE_ROW` 를 반환하고, 결과 셋이 끝나면 `SQLITE_DONE` 을 반환한다. 이 외의 다른 명의 경우는 함수 최초 호출 시 VDBE 가 그 명령 전체를 처리하도록 하낟.
- **마무리 (finalization - sqlite3_finalize)**: 해당 명령을 close 하고 자원을 해제한다. 

각각의 단계는 명령 핸들의 각 상태 (준비, 활동, 마무리) 와 일치한다. 
- **준비(prepare)** 상태는 필요한 모든 자원이 할당되고 명령을 실행할 준비가 되었음을 의미한다. 하지만 실행이 시작되는 것을 보장하지는 않는다. 
- **활동(active)**  상태는 해당 명령의 실행이 시작되고 실행에 필요한 락도 획득한다. 
- **마무리(finalize)** 상태는 해당 명령이 close 되고 관련된 모든 자원이 해제되었음을 의미한다.


![image](https://user-images.githubusercontent.com/18457707/66835018-bc517300-ef99-11e9-957c-0685faf16cd1.png)

Prepared 쿼리의 수행 과정을 그림으로 나타내면 위와 같다. 


### 매개변수화 SQL
SQL 문에서 매개변수를 가질 수 있다. sQL 명령이 컴파일된 후 나중에 값이 대입(binding) 될 수 있는 플레이스 홀더가 매개 변수이다. 해당 SQL 를 예시로 확인하는 것이 이해가 쉬울 것이다.
```bash
insert into foods (id, name) values (?,?);  // 위치 매개변수
insert into episodes (id, name) (:id, :name);  // 이름 매개변수
```
- 장점: 같은 명령을 다시 컴파일하지 않고 여러 번 실행할 수 있다. 명령을 종료하지 않고 재설정만 한 후 새로운 값을 바인딩하고 다시 실행하면 된다. 
