# Chapter 04 확장 실습 답안 템플릿

> **과제:** 관계형 데이터베이스와 SQL 시작하기  
> **사용 방법:** 이 파일을 내려받아 본인의 GitHub 저장소에 `chapter04_answer.md`라는 이름으로 저장한 뒤 실습하면서 바로 작성합니다.  
> **제출 방법:** LMS에는 파일을 직접 업로드하지 않고, **본인 GitHub 저장소의 `chapter04_answer.md` 파일 URL**을 제출합니다.

---

## 제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

```text
GitHub 계정 또는 별칭:candle1030
과제 작성일:2026-09-10
사용한 AI 도구:제미나이
```

---

# 1. 실습 환경과 시작 상태 확인

다음을 실행합니다.

```sql
SELECT current_database();
SELECT current_user;
SELECT current_schema();
SHOW search_path;
SHOW transaction_read_only;
```

| 확인 항목 | 실제 결과 | 의미 |
| --- | --- | --- |
| current_database() | ai_database_book | 현재 연결된 DB |
| current_user | postgres | 현재 접속한 계정 |
| current_schema() | public | 현재 스키마 |
| search_path | "$user", public | 검색경로 |
| transaction_read_only | off | 읽기 전용 여부(쓰기 가능) |

- [o] 현재 DB가 `ai_database_book`이다.
- [o] 변경 가능한 연결인지 확인했다.
- [o] 실행할 SQL 범위를 확인했다.
- [o] Auto-commit 상태를 확인했다.

### 변경 SQL을 실행하기 전에 현재 DB와 실행 범위를 확인해야 하는 이유

```실수로 엉뚱한 실무 데이터베이스나 다른 프로젝트 DB에 접속한 상태로 UPDATE나 DELETE 문을 실행하여 소중한 데이터를 날려버리는 대참사를 방지하기 위해서이다.

```

---

# 2. `public.students` 구조 생성

## 2-1. 실행 전 예상

```text
테이블 이름: public.students
한 행의 의미: 학생 한 명의 정보
예상 행 수: 0 (구조 생성 직후에는 데이터가 없음)
기본키: id
필수 열: name, email, created_at
중복을 막는 열: email
자동 생성 열: id (IDENTITY), created_at (DEFAULT CURRENT_TIMESTAMP)
```

## 2-2. 실행 파일

```text
code/chapter04/01_create_students.sql
```

## 2-3. 실행 후 확인

```text
테이블 생성 성공 여부: 성공
실제 행 수: 0
DBeaver에서 확인한 위치: Schemas -> public -> Tables -> students
```

### 각 열의 역할

| 열 | 타입 | NULL 가능? | 역할 |
| --- | --- | --- | --- |
| id | integer | 불가능(PK) | 내부 식별자 |
| name | VARCHAR(50) | 불가능 (NOT NULL) | 학생 이름 |
| email | VARCHAR(100) | 불가능 (UNIQUE, NOT NULL) | 학생 이메일 (중복 불가) |
| major | VARCHAR(100) | 가능 | 전공 |
| grade | INTEGER | 가능 | 학년 |
| created_at | TIMESTAMPTZ | 불가능 (DEFAULT) | 등록 시각 (기본값 현재 시각) |

### `id`를 학번이나 학생 수로 해석하면 안 되는 이유

```id는 행을 구분하기 위한 내부 식별자일 뿐이므로, 중간에 데이터가 삭제되거나 입력이 실패하면 빈 번호가 생길 수 있어 실제 학생 수나 학번과 일치하지 않을 수 있기 때문이다.

```

### 증거 화면

권장 경로:

```text
assignments/chapter04/images/step02_table.png
```

assignments/chapter04/images/2.png

---

# 3. 샘플 데이터 6명 입력

## 3-1. 실행 전 예상

```text
현재 행 수: 이미 데이터가 존재하거나 0명인 상태
실행 후 예상 행 수: 6명 추가 (총 6명 또는 중복 에러 발생 시 기존 데이터 유지)
예상되는 NULL 포함 학생: 윤서진 (major, grade가 생략되어 NULL)
```

## 3-2. 실행 파일

```text
code/chapter04/02_insert_students.sql
```

## 3-3. 실제 결과

```text
실제 행 수: 중복 키 에러(23505)로 인해 전체 삽입 차단됨
이준호 grade: 기존 입력 시도 데이터 기준 적용 안 됨
박서연 존재 여부: 기 입력 데이터 유지
윤서진 major: [NULL]
윤서진 grade: [NULL]
특이사항: 이미 동일한 이메일 데이터가 존재하여 UNIQUE 제약조건("students_email_key")에 의해 중복 입력이 정상적으로 방지됨을 확인 함.
```

### 예상과 실제 비교

```text
예상과 실제가 일치했는가: 다름
다르다면 이유: 이미 데이터가 테이블에 들어가 있는 상태에서 다시 INSERT를 실행하여 이메일 중복 제약조건 오류가 발생했기 때문임.
```

### `created_at` 값이 여러 행에서 같을 수 있는 이유

```하나의 명시적 트랜잭션 안에서 여러 행이 한 번에 입력되었고, CURRENT_TIMESTAMP는 트랜잭션 시작 시각을 기준으로 값을 채우기 때문이다.

```

---

# 4. SELECT 복습과 결과 검증

각 문제는 **SQL 실행 전에 예상 행 수를 먼저 작성**합니다.

| 번호 | 조회 문제 | 예상 행 수 | 실제 행 수 | 일치? | 다르면 이유 |
| ---: | --- | ---: | ---: | --- | --- |
| 1 | 전체 학생 | 7 | 7 | o | - |
| 2 | 이름·이메일만 조회 | 7 | 7 | o | - |
| 3 | 특정 전공 | 2 | 2 | o | - |
| 4 | 특정 학년 이상 | 2 | 2 | o | - |
| 5 | 두 전공 중 하나 | 3 | 3 | o | - |
| 6 | `grade IS NULL` | 1 | 1 | o | - |
| 7 | 전공 `DISTINCT` | 5 | 5 | o | - |
| 8 | 정렬 후 상위 3명 | 3 | 3 | o | - |

## 4-1. 내가 직접 작성한 SQL 2개

```sql
-- SQL 1
SELECT * 
FROM public.students 
WHERE grade = 2 
ORDER BY id;

```

```text
이 SQL의 한 행 의미: 2학년에 재학 중인 학생 한 명의 상세 정보
예상 행 수: 2
실제 행 수: 2
```

```sql
-- SQL 2
SELECT name, email, major 
FROM public.students 
WHERE email LIKE '%@example.com' 
ORDER BY email;

```

```text
이 SQL의 한 행 의미: @example.com 이메일을 사용하는 학생 한 명의 이름, 이메일, 전공 정보
예상 행 수: 7
실제 행 수: 7
```

## 4-2. `= NULL` 대신 `IS NULL`을 사용하는 이유

```NULL은 값이 없거나 모르는 상태(Unknown)를 뜻하므로 일반 값처럼 '=' 기호로 비교하면 결과를 판정할 수 없어 조회가 되지 않는다. 따라서 NULL 상태를 올바르게 판정하기 위해서는 전용 연산자인 IS NULL을 반드시 사용해야 한다.

```

## 4-3. `ORDER BY` 없이 결과 순서를 믿으면 안 되는 이유

```데이터베이스 엔진은 내부 저장 방식이나 인덱스 상태, 실행 계획에 따라 매번 똑같은 순서로 데이터를 반환한다는 보장을 하지 않는다. 따라서 조회 결과의 정렬 순서를 고정하려면 반드시 ORDER BY 절을 명시해야 한다.

```

## 4-4. `DISTINCT`가 원본 데이터를 삭제하는 기능인가요?

```아니다. DISTINCT는 원본 테이블의 데이터를 삭제하는 것이 아니라, SELECT로 조회된 결과 집합(Result Set)에서 중복된 행만 제거하여 보여주는 조회 전용 키워드이다.

```

### 증거 화면

권장 경로:

```text
assignments/chapter04/images/step04_select.png
```

assignments/chapter04/images/step04_select.png

---

# 5. 내 가상 학생 2명 추가

실명·실제 이메일 대신 가상 데이터를 사용합니다.


## 5-1. 실행 전 계획

```text
학생 A
이름:김테스
이메일:test1@example.com
전공:소프트웨어학
학년:3

학생 B
이름:이코딩
이메일:test2@example.com
전공:인공지능학
학년 또는 NULL:1

현재 행 수:
추가 후 예상 행 수:
```

## 5-2. 내가 실행한 INSERT

```INSERT INTO public.students (name, email, major, grade)
VALUES
    ('김테스', 'test1@example.com', '소프트웨어학', 3),
    ('이코딩', 'test2@example.com', '인공지능학', 1);

```

## 5-3. 실제 결과

```text
RETURNING 또는 확인 SELECT 결과:SELECT * FROM public.students WHERE name IN ('김테스', '이코딩'); 조회 결과 두 학생의 정보가 테이블에 정확히 삽입된 것을 확인 완료함.
실제 전체 행 수: 9명 (기존 7명 + 가상 학생 2명 추가)
예상과 일치 여부: 일치함 (2개의 행이 오류 없이 정상 삽입됨)
```

### 내가 일부 값을 NULL로 둔 이유 또는 NULL을 사용하지 않은 이유

```가상 학생을 추가할 때 모든 필수 열(name, email)과 선택 열(major, grade)에 유효한 값을 모두 제공하였으므로, 의도적으로 NULL을 남겨두지 않고 완전한 데이터를 채워 넣었다.

```

---

# 6. 안전한 UPDATE

내가 추가한 가상 학생 한 명만 수정합니다.

## 6-1. 먼저 대상 확인 SELECT

```SELECT * 
FROM public.students 
WHERE name = '김테스';

```

```text
예상 대상 행 수:1
실제 대상 행 수:1
```

## 6-2. UPDATE

```UPDATE public.students
SET grade = 4, major = '소프트웨어공학'
WHERE name = '김테스';

```

```text
예상 영향 행 수: 1
실제 영향 행 수: 1
RETURNING 결과: (별도 RETURNING 구문을 쓰지 않았다면 생략 또는 Updated Rows: 1 확인)
```

## 6-3. UPDATE 후 재조회

```SELECT * 
FROM public.students 
WHERE name = '김테스';

```

### `WHERE` 없는 UPDATE를 실행하면 위험한 이유

```WHERE 절을 누락하면 테이블에 존재하는 모든 학생의 데이터가 지정한 값으로 일괄 변경되어 원본 데이터가 손상되거나 유실될 수 있기 때문이다.

```

### 증거 화면

권장 경로:

```text
assignments/chapter04/images/step06_update.png
```

assignments/chapter04/images/step06_update.png
assignments/chapter04/images/step06_update2.png

---

# 7. 안전한 DELETE

내가 추가한 가상 학생 한 명을 삭제합니다.

## 7-1. 삭제 전 확인

```SELECT * 
FROM public.students 
WHERE name = '이코딩';

```

```text
예상 대상 행 수:1
실제 대상 행 수:1
```

## 7-2. DELETE

```DELETE FROM public.students 
WHERE name = '이코딩';

```

```text
예상 영향 행 수: 1
실제 영향 행 수: 1
RETURNING 결과: (별도 RETURNING 구문을 쓰지 않았다면 생략 또는 Deleted Rows: 1 확인)
```

## 7-3. 삭제 후 재조회

```SELECT * 
FROM public.students 
WHERE name = '이코딩';

```

```삭제 후 같은 조건의 SELECT 결과 행 수: 0

```

### `DELETE` 성공 메시지만 보고 끝내지 않고 다시 SELECT해야 하는 이유

```성공 메시지는 쿼리 실행 자체가 완료되었다는 뜻일 뿐, 실제로 의도한 특정 조건의 데이터가 정확히 겨냥되어 지워졌는지는 직접 SELECT로 조회하여 확인해야 데이터 누락이나 오삭제 사고를 방지할 수 있기 때문이다.

```

---

# 8. 본문 기준 UPDATE·DELETE 상태 검증

`04_update_delete_students.sql`을 본문 시작 상태에서 실행했다면 다음을 확인합니다.

```최종 학생 수: 7 (가상 학생 추가 및 기존 데이터 상태에 따라 다름)
이준호 grade: 3 (본문 기준 기본값 또는 수정 여부에 따름)
박서연 존재 여부: 존재함 (또는 삭제 실습에 따라 다름)
```

본문 기준 기대 상태와 비교합니다.

```text
학생 수 = 5
이준호 grade = 4
박서연 = 0행
```

### 내 실제 결과가 기준과 다르다면 원인

```앞선 단계에서 가상 학생을 추가하거나 일부 데이터를 수정 및 삭제하는 과정에서 본문의 초기 가정과 달리 데이터가 누적되거나 변경되었기 때문에, 최종 조회 결과의 행 수나 특정 학생의 정보(grade, 존재 여부 등)가 본문의 기대 상태와 다를 수 있다.

```

---

# 9. 의도한 실패 2개 관찰

> 실패 테스트는 데이터베이스 규칙이 실제로 데이터를 보호하는지 확인하는 실험입니다.

## 9-1. 중복 이메일 `UNIQUE` 오류

내가 사용한 SQL:

```INSERT INTO public.students (name, email, major, grade)
VALUES ('중복테스트', 'test1@example.com', '컴퓨터공학', 2);

```

```text
오류 메시지 핵심 단서: duplicate key value violates unique constraint "students_email_key" (중복된 키 값이 고유 제약 조건을 위반함)
왜 실패해야 맞는가: 이메일은 각 학생을 식별하는 고유한 값이므로 중복될 경우 데이터의 신뢰성과 식별성이 깨지기 때문이다.
어떤 규칙이 작동했는가: email 컬럼에 설정된 UNIQUE 제약조건
실패 후 기존 데이터가 어떻게 유지되었는가: INSERT 작업이 거부되고 트랜잭션이 차단되어 기존 테이블의 데이터는 변경 없이 온전하게 유지되었다.
```

## 9-2. 이름 `NULL` 입력 `NOT NULL` 오류

내가 사용한 SQL:

```sql
INSERT INTO public.students (name, email, major, grade)
VALUES (NULL, 'noname@example.com', '수학과', 1);
```

```text
오류 메시지 핵심 단서: null value in column "name" of relation "students" violates not-null constraint (name 컬럼의 null 값이 널 제약 조건을 위반함)
왜 실패해야 맞는가: 학생의 이름은 반드시 존재해야 하는 필수 정보이므로 값이 없는 상태로 저장되는 것을 막아야 하기 때문이다.
어떤 규칙이 작동했는가: name 컬럼에 설정된 NOT NULL 제약조건
```

### 실패한 INSERT 뒤 자동 생성 `id` 번호에 빈 구간이 생길 수 있어도 문제라고 단정할 수 없는 이유

```text
SERIAL이나 시퀀스(Sequence)를 통해 자동 증가하는 id는 행이 삽입될 때마다 번호가 소모되며, INSERT 시도가 실패하더라도 이미 할당된 시퀀스 번호는 롤백되지 않고 건너뛰어지기 때문이다. 데이터의 순차적 정합성 자체에는 문제가 없으므로 빈 번호 구간(Gaps)은 시스템 작동상 자연스러운 현상이다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter04/images/step09_constraint_error.png
```

`여기에 제약조건 오류 화면을 삽입하세요.`
assignments/chapter04/images/step09_constraint_error.png
---

# 10. `verify_students.sql`로 최종 상태 확인

실행 파일:

```text
code/chapter04/verify_students.sql
```

```text
현재 전체 학생 수: 7 (가상 학생 추가 및 삭제 실습 반영 결과)
NULL 개수: 1 (전공 또는 학년 등 누락된 데이터 기준)
이준호 grade: 3 (본문 기준 기본값)
박서연 존재 여부: 존재함 (또는 삭제 실습에 따라 다름)
현재 데이터 상태에서 예상과 다른 부분: 실습 과정에서 개별적으로 추가 및 삭제한 데이터가 반영되어 있어 초기 본문의 정적 상태와 일부 차이가 있을 수 있음.
```

### 검증 SQL을 따로 두면 좋은 이유

```text
데이터베이스 작업을 진행하면서 여러 번의 INSERT, UPDATE, DELETE로 인해 데이터가 의도치 않게 변경되었을 때, 전체 상태와 정합성을 객관적인 기준으로 빠르게 재확인하여 오류를 진단하고 복구할 수 있기 때문이다.
```

---

# 11. AI를 SQL 작성자가 아니라 검토자로 활용

먼저 본인이 SQL을 작성한 뒤 AI에게 검토를 요청합니다.

## 11-1. 내가 작성한 SQL

```sql
SELECT name, major, grade
FROM public.students
WHERE major = '컴퓨터공학'
ORDER BY grade DESC;
```

## 11-2. AI에게 전달한 핵심 요청

```text
작성한 SQL에 누락되거나 예외 상황(예: NULL 값 처리)에서 발생할 수 있는 문제가 없는지 검토하고 개선안을 제안해 줘.
```

## 11-3. AI 검토 결과

| AI 제안 | 수용 / 수정 / 거절 | 실제 검증 결과 | 나의 이유 |
| --- | --- | --- | --- |
| 동점자 처리를 위해 ORDER BY에 id 추가 제안 | 수용 | 학년이 같은 학생들의 정렬 순서가 안정적으로 고정됨 | 정렬의 일관성을 높이기 위해 수용함 |
| major 조건 비교 시 대소문자/공백 예외 대비 안내 | 거절 | 현재 데이터는 일관되게 입력되어 있어 우선 반영 안 함 | 현재 실습 데이터 환경에서는 불필요하여 거절함 |
| SELECT 절에 식별용 id 컬럼 포함 권장 | 수용 | 조회된 결과에서 특정 학생을 식별하기 더 용이해짐 | 관리와 확인이 편리해져서 수용함 |

### AI가 예상한 영향 행 수와 실제 결과가 같았나요?

```text
조회(SELECT) 쿼리이므로 영향 행 수 대신 조회된 결과 행 수가 예상했던 3개와 실제 결과 3개로 완벽하게 일치하였다.
```

### AI 답변을 실행 전에 검토해야 하는 이유

```text
AI가 제안하는 코드가 항상 현재 데이터베이스의 실제 스키마나 제약조건, 데이터 분포를 정확히 반영하는 것은 아니므로, 의도치 않은 오류나 비효율적인 연산이 포함되지 않았는지 실행 전에 사람이 직접 논리적 타당성을 검증해야 하기 때문이다.
```

---

# 12. 내 서비스 테이블 하나 확장 설계

Chapter 01~03에서 정한 개인 서비스에서 **테이블 하나**를 선택합니다.

```text
서비스 이름: 온라인 스터디 관리 서비스 (StudyHub)
테이블 이름: public.tasks
한 행의 의미: 사용자가 등록한 개별 학습 과제나 할 일의 상세 내용과 진행 상태
```

| 열 이름 | 저장할 값 | 타입 후보 | NULL 가능? | UNIQUE 후보? | 이유 |
| --- | --- | --- | --- | --- | --- |
| id | 과제 고유 번호 | SERIAL | NO | YES (PK) | 각 할 일을 중복 없이 식별하기 위함 |
| user_id | 작성자 회원 번호 | INT | NO | NO | 어떤 사용자의 할 일인지 연결하기 위함 |
| title | 할 일 제목 | VARCHAR(100) | NO | NO | 과제의 핵심 내용을 기록하기 위함 |
| due_date | 과제 마감일 | DATE | YES | NO | 완료해야 하는 기한을 관리하기 위함 |
| is_completed | 완료 여부 | BOOLEAN | NO | NO | 과제의 수행 상태를 추적하기 위함 |

```text
PK 후보: id
업무 식별자 후보: (user_id, title) (동일 사용자가 중복된 제목의 과제를 등록하는 것을 방지)
아직 미확정인 규칙: 마감일이 지난 과제의 자동 상태 변경 규칙 및 알림 발송 시점
```

## 선택: CREATE TABLE 초안

> 아직 확정되지 않은 업무 규칙은 억지로 제약조건으로 만들지 않습니다.

```sql
DROP TABLE IF EXISTS public.tasks;

CREATE TABLE public.tasks (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    due_date DATE,
    is_completed BOOLEAN NOT NULL DEFAULT FALSE
);
```

### AI에게 검토받은 뒤 수정한 부분

```text
- 완료 여부(is_completed) 컬럼에 기본값(DEFAULT FALSE)을 추가하여 신규 데이터를 삽입할 때 완료 상태를 매번 직접 입력하지 않아도 되도록 개선함.
- 향후 사용자 테이블과 연결할 수 있도록 user_id 컬럼을 설계에 미리 포함함.
```

---

# 13. 최종 성찰

아래 문장은 본인의 말로 작성합니다.

```text
1. SQL 실행 성공과 올바른 대상 선택이 다른 이유는
   문법적 오류 없이 쿼리가 정상 실행되어 완료되었다 하더라도, 내가 의도하지 않은 다른 데이터나 전체 행을 대상으로 연산이 수행되었을 수 있기 이다.

2. UPDATE와 DELETE 전에 SELECT를 먼저 해야 하는 이유는
   수정이나 삭제 대상이 되는 데이터가 정확히 내가 원하는 그 행들만 정확히 겨냥되고 있는지 사전에 확인하여 오동작을 방지하기 이다.

3. 영향받은 행 수를 확인해야 하는 이유는
   내가 의도한 개수만큼의 데이터만 실제로 변경 또는 삭제되었는지를 수치로 즉시 검증하여 대규모 데이터 유실 사고를 예방하기 이다.

4. UNIQUE 또는 NOT NULL 오류를 '보호 장치가 정상 동작한 결과'라고 볼 수 있는 이유는
   데이터베이스가 사전에 정의된 무결성 규칙을 위반하는 잘못된 데이터의 유입을 스스로 차단하여 기존 데이터를 안전하게 지켜냈기 이다.

5. AI가 SQL을 만들어 주더라도 내가 반드시 확인해야 하는 것은
   작성된 코드가 현재의 실제 데이터베이스 스키마와 제약조건, 그리고 비즈니스 로직의 의도에 정확히 부합하는지 여부이다.

---

# 14. 제출 체크리스트

- [O] `chapter04_answer.md`를 본인 저장소에 만들었다.
- [O] 현재 DB와 실행 환경을 확인했다.
- [O] `public.students`를 생성했다.
- [O] 샘플 6명 입력 결과를 검증했다.
- [O] SELECT 문제에서 실행 전 예상 행 수를 작성했다.
- [O] 가상 학생 2명을 추가했다.
- [O] UPDATE 전후를 SELECT로 확인했다.
- [O] DELETE 전후를 SELECT로 확인했다.
- [O] UNIQUE 오류를 관찰했다.
- [O] NOT NULL 오류를 관찰했다.
- [O] `verify_students.sql`로 상태를 확인했다.
- [O] AI 제안을 실제 SQL 결과와 비교했다.
- [O] 개인 서비스 테이블 하나를 확장 설계했다.
- [O] 핵심 캡처는 3~4장 정도로 제한했다.
- [O] 비밀번호·개인정보가 캡처에 없다.
- [O] Markdown 이미지가 GitHub 웹 화면에서 정상 표시된다.
- [O] commit/push를 완료했다.

---

# 15. LMS 제출 URL

아래 형식의 **본인 GitHub 파일 URL**을 LMS에 제출합니다.

```text
https://github.com/<본인-GitHub-ID>/<본인-저장소>/blob/main/assignments/chapter04/chapter04_answer.md
```

내 제출 URL:

```text

```

> 교수자 템플릿 URL이나 저장소 메인 URL이 아니라 **작성 완료된 본인 `chapter04_answer.md` 파일 화면 URL**을 제출합니다.