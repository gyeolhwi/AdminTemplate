# AdminTemplate
관리자화면 초안 시나리오 구성 및 README 작성 테스트용
어떤 서비스를 만들 것인가 → 기본적인 관리자를 구성하고 싶음
1. 플로우차트 구성 : 전체적인 서비스 흐름 파악
2. 기능 명세 & 기술 스택 선정 : 무엇을 어떤 기술로 만들지 확정 ( Vite(React + TypeScript) / Java SpringBoot ) → 추후 Next.js(Express)
3. ERD설계 : 데이터 구조 잡기
4. API 명세서 작성 (보류)
5. 기능별 시나리오 구성 (Mermaid형식으로 기술)
6. UI / UX (Visily/Figma Make) : 화면 초안 그리기

```graph TD
    Start((관리자 접속)) --> Login[로그인 페이지]
    Login --> |인증 성공| Main[대시보드 메인]
    
    Main --> Menu1[사용자 관리]
    Menu1 --> Sub1_1[회원 목록/검색]
    Menu1 --> Sub1_2[회원 상세/수정]
    Menu1 --> Sub1_3[탈퇴 처리]

    Main --> Menu2[콘텐츠 관리]
    Menu2 --> Sub2_1[게시글 관리]
    Menu2 --> Sub2_2[댓글 관리]

    Main --> Menu3[파일 관리]
    Menu3 --> Sub3_1[업로드 파일 목록]
    Menu3 --> Sub3_2[미사용 파일 정리]

    Main --> Menu4[메일 관리]
    Menu4 --> Sub4_1[메일 발송]
    Menu4 --> Sub4_2[발송 이력 조회]

    Main --> Menu5[통계]
    Menu5 --> Sub5_1[방문자 통계]
    Menu5 --> Sub5_2[가입자 추이]
```

### 1.  사용자 관리 : 로그인 및 토큰 발급(JWT 인증) 시나리오

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자 (React)
    participant Ctrl as Auth Controller
    participant Svc as Auth Service
    participant DB as User Repository (DB)

    Note over Admin, DB: 로그인 프로세스

    Admin->>Ctrl: POST /api/auth/login <br/>{id, password}
    Ctrl->>Svc: 로그인 비즈니스 로직 호출
    Svc->>DB: ID로 회원 정보 조회 (findById)
    
    alt 회원이 존재하지 않음
        DB-->>Svc: null
        Svc-->>Ctrl: Error (User Not Found)
        Ctrl-->>Admin: 404 Not Found / 에러 메시지
    else 회원이 존재함
        DB-->>Svc: 회원 엔티티(암호화된 비번 포함) 반환
        Svc->>Svc: 비밀번호 일치 여부 확인 (BCrypt)
        
        alt 비밀번호 불일치
            Svc-->>Ctrl: Error (Invalid Password)
            Ctrl-->>Admin: 401 Unauthorized
        else 비밀번호 일치
            Svc->>Svc: JWT Access Token 생성
            Svc-->>Ctrl: 토큰 반환
            Ctrl-->>Admin: 200 OK + { accessToken: "..." }
        end
    end
```

### 2. 컨텐츠 관리 시나리오
```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자 (React)
    participant Ctrl as Post Controller
    participant Svc as Post Service
    participant DB as Post Repository (DB)

    Note over Admin, DB: 1. 기존 데이터 불러오기
    Admin->>Ctrl: GET /api/posts/{id}
    Ctrl->>Svc: 상세 조회 요청
    Svc->>DB: ID로 게시글 찾기
    DB-->>Svc: 게시글 데이터
    Svc-->>Ctrl: PostDTO 반환
    Ctrl-->>Admin: 데이터 렌더링 (수정 폼 채우기)

    Note over Admin, DB: 2. 내용 수정 후 저장
    Admin->>Ctrl: PUT /api/posts/{id} <br/>{title, content, ...}
    Ctrl->>Svc: 게시글 수정 요청 (Transactional)
    Svc->>DB: 게시글 존재 확인 (findById)
    
    opt 게시글이 없음
        DB-->>Svc: null
        Svc-->>Admin: 404 Not Found (삭제된 게시글)
    end

    Svc->>Svc: 엔티티 내용 업데이트 (Dirty Checking)
    Svc->>DB: 변경사항 저장 (Commit)
    DB-->>Svc: 완료
    Svc-->>Ctrl: 수정된 PostDTO 반환
    Ctrl-->>Admin: 200 OK (수정 완료 알림)
```
### 3. 메일 송신 관리 시나리오
```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자 (React)
    participant Ctrl as Mail Controller
    participant Svc as Mail Service
    participant DB as Mail Log Repo
    participant Async as 비동기 작업 (Thread)
    participant SMTP as 메일 서버 (Gmail 등)

    Admin->>Ctrl: POST /api/mails/send <br/>{receivers, subject, content}
    Ctrl->>Svc: 메일 발송 요청
    
    Svc->>DB: 발송 이력 '준비(PENDING)' 상태로 저장
    DB-->>Svc: 저장된 ID 반환
    
    Svc->>Async: 메일 발송 작업 위임 (@Async)
    Svc-->>Ctrl: 응답 (void)
    Ctrl-->>Admin: 200 OK ("메일 발송을 시작했습니다.")

    par 백그라운드 작업 (사용자는 대기하지 않음)
        Async->>SMTP: 실제 이메일 전송 요청
        SMTP-->>Async: 전송 성공 응답
        
        alt 전송 성공
            Async->>DB: 상태 업데이트 (PENDING -> SENT)
        else 전송 실패
            Async->>DB: 상태 업데이트 (PENDING -> FAILED)
        end
    end
```

## ERD
```mermaid
erDiagram
    %% 1. 사용자 관리 (Users)
    USERS {
        Long id PK "기본키 (Auto Increment)"
        String username UK "아이디 (Unique)"
        String password "암호화된 비밀번호"
        String name "사용자 이름"
        String email "이메일"
        String role "권한 (ROLE_ADMIN, ROLE_USER)"
        Boolean is_deleted "탈퇴 여부 (논리 삭제)"
        DateTime created_at "가입일"
        DateTime updated_at "수정일"
    }

    %% 2. 게시글 관리 (Posts)
    POSTS {
        Long id PK "기본키"
        Long user_id FK "작성자 ID"
        String title "제목"
        Text content "내용 (LongText)"
        Integer view_count "조회수"
        Boolean is_visible "노출 여부"
        DateTime created_at "등록일"
        DateTime updated_at "수정일"
    }

    %% 3. 댓글 관리 (Comments)
    COMMENTS {
        Long id PK "기본키"
        Long post_id FK "게시글 ID"
        Long user_id FK "작성자 ID"
        String content "댓글 내용"
        DateTime created_at "작성일"
    }

    %% 4. 파일 관리 (Files)
    FILES {
        Long id PK "기본키"
        Long uploader_id FK "업로더 ID"
        String original_name "사용자가 올린 파일명"
        String saved_name "서버에 저장된 UUID 파일명"
        String file_path "저장 경로 (S3 URL or Local Path)"
        Long file_size "파일 크기"
        String file_ext "확장자 (jpg, png 등)"
        DateTime created_at "업로드일"
    }

    %% 5. 메일 발송 이력 (MailLogs)
    MAIL_LOGS {
        Long id PK "기본키"
        Long sender_id FK "발송자(관리자) ID"
        String receiver_email "수신자 이메일"
        String subject "메일 제목"
        Text content "메일 내용"
        String status "상태 (PENDING, SENT, FAILED)"
        String error_message "실패 시 에러 로그"
        DateTime sent_at "발송 시각"
    }

    %% 6. 일별 통계 (DailyStats)
    DAILY_STATS {
        Date date PK "날짜 (YYYY-MM-DD)"
        Long total_visitors "방문자 수"
        Long new_signups "신규 가입자 수"
        Long total_posts "작성된 게시글 수"
    }

    %% 관계 정의 (Relationship)
    USERS ||--o{ POSTS : "1:N (작성)"
    USERS ||--o{ COMMENTS : "1:N (작성)"
    USERS ||--o{ FILES : "1:N (업로드)"
    USERS ||--o{ MAIL_LOGS : "1:N (발송)"
    
    POSTS ||--o{ COMMENTS : "1:N (소유)"
```


### DB구축(SQL)
```SQL
-- 1. 사용자 테이블 (Users)
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '기본키',
    username VARCHAR(50) NOT NULL UNIQUE COMMENT '로그인 아이디',
    password VARCHAR(255) NOT NULL COMMENT '암호화된 비밀번호',
    name VARCHAR(50) NOT NULL COMMENT '사용자 이름',
    email VARCHAR(100) NOT NULL COMMENT '이메일',
    role VARCHAR(20) NOT NULL DEFAULT 'ROLE_USER' COMMENT '권한',
    is_deleted BOOLEAN DEFAULT FALSE COMMENT '탈퇴 여부',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '가입일',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정일'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='사용자 정보';

-- 2. 게시글 테이블 (Posts)
CREATE TABLE posts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '기본키',
    user_id BIGINT NOT NULL COMMENT '작성자 ID',
    title VARCHAR(200) NOT NULL COMMENT '제목',
    content LONGTEXT COMMENT '내용',
    view_count INT DEFAULT 0 COMMENT '조회수',
    is_visible BOOLEAN DEFAULT TRUE COMMENT '노출 여부',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='게시글';

-- 3. 댓글 테이블 (Comments)
CREATE TABLE comments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    post_id BIGINT NOT NULL COMMENT '게시글 ID',
    user_id BIGINT NOT NULL COMMENT '작성자 ID',
    content TEXT NOT NULL COMMENT '댓글 내용',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='댓글';

-- 4. 파일 테이블 (Files)
CREATE TABLE files (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    uploader_id BIGINT NOT NULL COMMENT '업로더 ID',
    original_name VARCHAR(255) NOT NULL COMMENT '원본 파일명',
    saved_name VARCHAR(255) NOT NULL COMMENT '저장된 파일명(UUID)',
    file_path VARCHAR(500) NOT NULL COMMENT '파일 경로',
    file_size BIGINT NOT NULL COMMENT '파일 크기(Byte)',
    file_ext VARCHAR(10) NOT NULL COMMENT '확장자',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (uploader_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='첨부파일';

-- 5. 메일 발송 로그 (MailLogs)
CREATE TABLE mail_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sender_id BIGINT NOT NULL COMMENT '발송자 ID',
    receiver_email VARCHAR(100) NOT NULL COMMENT '수신자 이메일',
    subject VARCHAR(200) NOT NULL COMMENT '메일 제목',
    content TEXT COMMENT '메일 내용',
    status VARCHAR(20) DEFAULT 'PENDING' COMMENT '발송 상태(PENDING/SENT/FAILED)',
    error_message TEXT COMMENT '에러 메시지',
    sent_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '발송 시간',
    FOREIGN KEY (sender_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='메일 발송 이력';

-- 6. 일별 통계 (DailyStats)
CREATE TABLE daily_stats (
    date DATE PRIMARY KEY COMMENT '날짜(YYYY-MM-DD)',
    total_visitors BIGINT DEFAULT 0 COMMENT '총 방문자',
    new_signups BIGINT DEFAULT 0 COMMENT '신규 가입자',
    total_posts BIGINT DEFAULT 0 COMMENT '작성 게시글 수'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='일별 통계';
```
### API명세서

#### 인증/사용자(Auth & User)

| **Method** | **URI**           | **설명**      | **Request Body 예시**                          |
| ---------- | ----------------- | ----------- | -------------------------------------------- |
| **POST**   | `/api/auth/login` | 로그인 (토큰 발급) | `{ "username": "admin", "password": "123" }` |
| **POST**   | `/api/users`      | 회원가입/등록     | `{ "username": "test", "name": "김결휘", ... }` |
| **GET**    | `/api/users`      | 회원 목록 조회    | `?page=1&size=10` (쿼리 파라미터)                  |
| **PUT**    | `/api/users/{id}` | 회원 정보 수정    | `{ "name": "변경할이름", "email": "..." }`        |
#### 게시글
|**Method**|**URI**|**설명**|**Request Body 예시**|
|---|---|---|---|
|**GET**|`/api/posts`|게시글 목록 조회|`?keyword=검색어`|
|**GET**|`/api/posts/{id}`|게시글 상세 조회|-|
|**POST**|`/api/posts`|게시글 등록|`{ "title": "공지", "content": "내용", "userId": 1 }`|
|**PUT**|`/api/posts/{id}`|게시글 수정|`{ "title": "수정공지", "content": "수정내용" }`|
|**DELETE**|`/api/posts/{id}`|게시글 삭제|-|
#### 파일 및 메일
|**Method**|**URI**|**설명**|**Request Body 예시**|
|---|---|---|---|
|**POST**|`/api/files/upload`|파일 업로드|`Form-Data` 형식 (file 객체)|
|**GET**|`/api/files/{id}`|파일 다운로드|-|
|**POST**|`/api/mail/send`|메일 발송 요청|`{ "receiver": "user@abc.com", "subject": "..." }`|

### Spring Boot 프로젝트 폴더 구조
```tree
src/main/java/com/gyeolhwi/adminproject
│
├── global                  // # 프로젝트 공통 설정
│   ├── config              // (SecurityConfig, WebConfig 등 설정)
│   ├── error               // (GlobalExceptionHandler 등 에러 처리)
│   └── util                // (날짜 계산기 등 유틸 파일)
│
├── domain                  // # 핵심 기능 모음
│   ├── user                // 1. 사용자 관련
│   │   ├── controller      // (UserController)
│   │   ├── service         // (UserService)
│   │   ├── repository      // (UserRepository)
│   │   ├── entity          // (User Entity)
│   │   └── dto             // (LoginRequest, UserResponse 등)
│   │
│   ├── post                // 2. 게시글 관련
│   │   ├── controller
│   │   ├── service
│   │   ├── repository
│   │   ├── entity
│   │   └── dto
│   │
│   ├── file                // 3. 파일 관련
│   └── mail                // 4. 메일 관련
│
└── AdminProjectApplication.java  // 실행 파일
```
