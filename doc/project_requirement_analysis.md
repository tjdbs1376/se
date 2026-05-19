## 1. 서론

### 1.1 목적 및 범위

본 문서는 조직 내 파일의 중앙 관리 및 협업을 지원하기 위한 '미니 드라이브' 시스템의 요구사항을 분석한 결과입니다. 시스템이 수행해야 할 기능, 구조적 요소, 동적 행위를 UML 다이어그램을 통해 정의합니다.

### 1.2 용어 정의

* **RBAC**: 역할 기반 접근 제어 (Role-Based Access Control).


* **CRC 카드**: 클래스-책임-협업자 카드 (Class-Responsibility-Collaboration Card).



---

## 2. 기능 모델링 (Functional Modeling)

### 2.1 유스케이스 다이어그램

사용자 관점에서 시스템의 기능을 식별하여 도식화하였습니다. 직관적인 이해를 돕기 위해 유스케이스를 8개의 주요 그룹으로 구성하였습니다.

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR
    User(("👤내부 사용자"))
    External(("🌐외부 협력자"))
    Admin(("🧑‍💼관리자"))

    subgraph MiniDrive["미니 드라이브 시스템 (System Boundary)"]
        direction TB
        UC1_1(["회원 가입하기"])
        UC1_2(["로그인하기"])
        UC1_3(["로그인 실패 처리하기"])
        UC2(["파일 전송하기<br/>(업로드/다운로드)"])
        UC3(["파일/폴더 관리하기<br/>(수정/삭제/복구)"])
        UC4(["파일 검색하기"])
        UC5(["공유 및 권한 설정하기"])
        UC6(["버전 관리 및 복구하기"])
        UC7(["외부 링크로 접근하기"])
        UC8(["시스템 및 계정 관리하기"])
    end

    User --- UC1_1
    User --- UC1_2
    User --- UC2
    User --- UC3
    User --- UC4
    User --- UC5
    User --- UC6

    External --- UC7
    Admin --- UC8

    %% Relationships
    UC1_3 -. extend .-> UC1_2
    UC2 -. include .-> UC1_2
    UC3 -. include .-> UC1_2
    UC5 -. include .-> UC1_2
```

* **액터 식별**: 역할을 중심으로 내부 사용자, 외부 협력자, 관리자를 식별하였습니다.


* **포함 관계**: 파일 전송, 파일 관리 및 공유 기능은 사용자 인증이 필요하므로 로그인하기 유스케이스와 `<<include>>` 관계를 설정하였습니다.


### 2.2 유스케이스 설명서: 파일 업로드하기

유스케이스의 식별 정보와 상세 흐름을 나타냅니다.

| 항목 | 상세 내용 |
|---|---|
| **ID / 이름** | UC-002 / 파일 업로드하기 (File Upload) |
| **주요 액터** | 내부 사용자 |
| **사전 조건 (Precondition)** | 사용자는 로그인된 상태여야 하며 업로드 권한을 가지고 있어야 한다. |
| **정상 시나리오** | 1. 사용자는 업로드할 파일을 선택하고 전송 버튼을 클릭한다.<br>2. 시스템은 로그인 세션 및 접근 권한을 확인한다.<br>3. 시스템은 잔여 저장 용량을 검사한다.<br>4. 시스템은 파일 확장자와 크기를 검사한다.<br>5. 시스템은 파일을 저장하고 메타데이터를 DB에 기록한다.<br>6. 시스템은 업로드 완료 메시지를 출력한다. |
| **예외 처리** | • 용량 부족 시: 저장 공간 부족 메시지를 출력하고 업로드를 중단한다.<br>• 차단 확장자 발견 시: 업로드 불가 파일 형식 메시지를 출력한다. |                                                                                                 |

---

## 3. 구조 모델링 (Structural Modeling)

### 3.1 클래스 다이어그램

업무 수행 과정에서 필요한 주요 도메인 객체를 중심으로 식별하고 관계를 정의하였으며, 시스템 흐름 표현을 위해 일부 제어 객체를 함께 포함하였습니다. 문장 분석법을 통해 명사는 클래스로, 동사는 연산으로 매핑하였습니다.

```mermaid
classDiagram
    class User {
        -String userId
        -String email
        -long storageLimit
        +login() void
        +checkQuota() boolean
        +verifyToken() boolean
    }
    class FileEntity {
        -String fileId
        -String fileName
        -long size
        -int version
        +insertMetadata() void
        +moveToTrash() void
        +restore() void
    }
    class StorageManager {
        -String storagePath
        +saveFile(data) String
        +deleteFile(path) void
    }
    class AuthService {
        -int failCount
        +authenticate(id, pw) boolean
        +verifyToken(token) boolean
        +lockAccount(userId) void
    }
    class FileController {
        +uploadFile(file) void
        +checkValidity(file) boolean
    }

    User "1" --> "*" FileEntity : 소유 (Association)
    FileController ..> AuthService : 의존 (Dependency)
    FileController ..> StorageManager : 의존 (Dependency)

```

* **가시성**: 속성은 `-` (Private), 연산은 `+` (Public)로 구분하였습니다.


* **기수성**: 한 명의 사용자는 여러 개의 파일을 소유할 수 있으므로 `1 대 *` 관계를 설정하였습니다.



### 3.2 CRC 카드 명세 (FileEntity)

클래스의 책임과 협업 관계를 명세합니다.

* **전면부 (Front)**:


* **Class Name**: FileEntity (ID: 02, Type: Concrete)
* **Description**: 파일의 메타데이터 및 물리적 저장 상태를 관리한다.
* **Responsibilities**: 메타데이터 저장, 휴지통 이동, 버전 복구.
* **Collaborators**: User, StorageManager, AuthService.


* **후면부 (Back)**:


* **Attributes**: fileId, fileName, size, version.
* **Relationships**: User (Owned by), StorageManager (Associated).



---

## 4. 행위 모델링 (Behavioral Modeling)

### 4.1 순차 다이어그램: 파일 업로드하기

객체 간의 메시지 패싱을 시간 흐름 중심으로 나타냅니다. ABCD 규칙(Actor-Boundary-Control-Data)에 따라 객체를 나열하였습니다.

```mermaid
sequenceDiagram
    autonumber
    actor User as 내부 사용자 (Actor)
    participant UI as Upload UI (Boundary)
    participant Ctrl as File Controller (Control)
    participant Auth as AuthService (Control)
    participant Storage as Storage Manager (Data)
    participant DB as File DB (Data)

    User->>UI: 파일 선택 및 업로드 클릭
    UI->>Ctrl: uploadFile(file, token)
    
    Ctrl->>Auth: verifyToken(token)
    Auth-->>Ctrl: Valid Token 확인

    Note over Ctrl: 용량 및 확장자 체크 (FR-003)
    Ctrl->>Ctrl: checkValidity()

    Ctrl->>Storage: saveFile(data)
    Storage-->>Ctrl: success (path 반환)

    Ctrl->>DB: insertMetadata(path, size, ver)
    DB-->>Ctrl: OK (fileId 발급)

    Ctrl-->>UI: 업로드 완료 메시지 전송
    UI-->>User: 결과 화면 표시

```

### 4.2 상태기계 다이어그램: 파일 상태

파일 객체의 생성부터 소멸까지의 상태 변화를 모델링하였습니다.

```mermaid
stateDiagram-v2
    [*] --> 업로드됨: 업로드 완료
    업로드됨 --> 휴지통보관: 삭제 요청 (moveToTrash)
    휴지통보관 --> 업로드됨: 복구 요청 (restore)
    휴지통보관 --> [*]: 30일 경과 (영구 삭제)
    업로드됨 --> 무결성검사중: 무결성 체크 이벤트 발생
    무결성검사중 --> 업로드됨: 체크 완료

```

---

## 5. 산출물 간 일관성 점검 (Consistency Verification)

설계 단계로 진행하기 전 분석 모델 간의 일관성을 점검합니다.

* **기능 모델 vs 구조 모델**: 유스케이스 설명서에 명시된 `AuthService`, `User`, `FileEntity` 클래스가 클래스 다이어그램에 모두 정의되어 있습니다.


* **기능 모델 vs 행위 모델**: '파일 업로드' 유스케이스가 하나의 순차 다이어그램으로 표현되었으며, 액터 정보가 일치합니다.


* **구조 모델 vs 행위 모델**: 순차 다이어그램에서 객체 간 전달되는 메시지(`verifyToken`, `insertMetadata` 등)가 클래스 다이어그램의 연산으로 정의되어 있습니다.



---
