# Entity Framework Core 학습 가이드

이 저장소는 Entity Framework Core를 체계적으로 학습하기 위한 문서와 예제 코드를 담고 있습니다.

---

## 목차

### Part 1: 기초

#### 01. EF Core 소개
- ORM(Object-Relational Mapping)이란?
- EF Core vs EF6 비교
- EF Core의 장점과 단점
- 지원되는 데이터베이스 프로바이더

#### 02. 개발 환경 설정
- .NET SDK 설치
- 필요한 NuGet 패키지
- 첫 번째 EF Core 프로젝트 생성
- DbContext 기본 구성

#### 03. 모델 정의 기초
- 엔티티 클래스 작성
- Data Annotations vs Fluent API
- 기본 키(Primary Key) 설정
- 속성(Property) 구성

---

### Part 2: 데이터 모델링

#### 04. 관계(Relationships) 설정
- 일대다(One-to-Many) 관계
- 일대일(One-to-One) 관계
- 다대다(Many-to-Many) 관계
- 자기 참조 관계(Self-Referencing)

#### 05. Fluent API 심화
- OnModelCreating 메서드
- 엔티티 구성 분리 (IEntityTypeConfiguration)
- 인덱스 설정
- 복합 키(Composite Key) 설정

#### 06. 상속 매핑 전략
- TPH (Table Per Hierarchy)
- TPT (Table Per Type)
- TPC (Table Per Concrete Class)
- 전략별 장단점 비교

#### 07. 값 객체와 소유 타입
- 값 변환기(Value Converters)
- 소유 타입(Owned Types)
- 복합 타입(Complex Types) - EF Core 8+

---

### Part 3: 데이터 쿼리

#### 08. LINQ 쿼리 기초
- 기본 쿼리 작성
- Where, Select, OrderBy
- First, Single, Find 차이점
- Any, All, Count 메서드

#### 09. 관련 데이터 로딩
- 즉시 로딩(Eager Loading) - Include/ThenInclude
- 지연 로딩(Lazy Loading)
- 명시적 로딩(Explicit Loading)
- 분할 쿼리(Split Queries)

#### 10. 고급 쿼리 기법
- 그룹화(GroupBy)와 집계 함수
- 조인(Join) 연산
- 서브쿼리
- 원시 SQL 쿼리 (FromSqlRaw, FromSqlInterpolated)

#### 11. 쿼리 필터와 전역 필터
- 쿼리 필터(Query Filters)
- Soft Delete 구현
- Multi-Tenancy 패턴

---

### Part 4: 데이터 조작

#### 12. 변경 추적(Change Tracking)
- 엔티티 상태(Entity States)
- AsNoTracking 사용
- 변경 감지 전략
- ChangeTracker API

#### 13. 데이터 저장
- Add, Update, Remove
- SaveChanges와 SaveChangesAsync
- AddRange, UpdateRange, RemoveRange
- 트랜잭션 처리

#### 14. 동시성 제어
- 낙관적 동시성(Optimistic Concurrency)
- RowVersion/Timestamp 사용
- 동시성 충돌 처리
- 비관적 잠금(Pessimistic Locking)

---

### Part 5: 마이그레이션

#### 15. 마이그레이션 기초
- 마이그레이션이란?
- Add-Migration, Update-Database
- 마이그레이션 되돌리기
- 마이그레이션 스크립트 생성

#### 16. 마이그레이션 고급
- 데이터 시딩(Data Seeding)
- 사용자 정의 마이그레이션
- 프로덕션 환경 마이그레이션 전략
- 여러 DbContext 관리

---

### Part 6: 성능 최적화

#### 17. 쿼리 성능 최적화
- 생성된 SQL 확인하기
- N+1 문제 해결
- 프로젝션(Projection) 활용
- 컴파일된 쿼리(Compiled Queries)

#### 18. 벌크 작업
- ExecuteUpdate (EF Core 7+)
- ExecuteDelete (EF Core 7+)
- 서드파티 라이브러리 활용

#### 19. 연결 및 풀링
- 연결 문자열 구성
- 연결 풀링(Connection Pooling)
- DbContext 풀링
- 연결 복원력(Connection Resiliency)

---

### Part 7: 테스트

#### 20. 단위 테스트
- InMemory 프로바이더
- SQLite In-Memory 모드
- 테스트용 DbContext 설정
- Mock vs 실제 데이터베이스

#### 21. 통합 테스트
- TestContainers 활용
- 테스트 데이터 설정
- 트랜잭션 롤백 전략

---

### Part 8: 실전 패턴

#### 22. Repository 패턴
- Generic Repository
- Unit of Work 패턴
- Repository 패턴의 장단점
- 직접 DbContext 사용 vs Repository

#### 23. DDD와 EF Core
- Aggregate Root
- 도메인 이벤트
- 값 객체 매핑
- 풍부한 도메인 모델

#### 24. Clean Architecture와 EF Core
- 계층 분리
- DbContext 의존성 주입
- Specification 패턴

---

### Part 9: 고급 주제

#### 25. 데이터베이스 프로바이더별 기능
- SQL Server 전용 기능
- PostgreSQL 전용 기능 (Npgsql)
- MySQL 전용 기능
- SQLite 특성

#### 26. JSON 컬럼 지원
- JSON 컬럼 매핑 (EF Core 7+)
- JSON 데이터 쿼리
- 문서 데이터베이스 스타일 활용

#### 27. 시간 테이블과 감사
- Temporal Tables (SQL Server)
- 감사 로그 구현
- 자동 타임스탬프 설정

#### 28. Database-First 접근법
- Scaffold-DbContext
- 기존 데이터베이스 리버스 엔지니어링
- 모델 커스터마이징

---

### Part 10: 실습 프로젝트

#### 29. 블로그 시스템 만들기
- 요구사항 분석
- 모델 설계
- CRUD 구현
- 검색 및 페이징

#### 30. 전자상거래 도메인 모델링
- 복잡한 도메인 모델
- 주문 처리 워크플로우
- 재고 관리
- 결제 연동 고려사항

---

## 학습 순서 권장

1. **입문자**: Part 1 → Part 2 → Part 3 → Part 4 → Part 5 순서로 학습
2. **경험자**: 필요한 Part를 선택적으로 학습
3. **실무 적용**: Part 6, Part 7, Part 8 집중

## 사전 요구 지식

- C# 기본 문법
- LINQ 기초
- SQL 기본 이해
- .NET 프로젝트 구조 이해

## 참고 자료

- [공식 EF Core 문서](https://learn.microsoft.com/ef/core/)
- [EF Core GitHub 저장소](https://github.com/dotnet/efcore)
