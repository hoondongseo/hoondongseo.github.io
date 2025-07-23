---
title: "Node.js + Express로 완전한 JWT 인증 시스템 구축하기 Part 1: 회원가입부터 로그인까지"
categories:
- MiniProject
- SimpleAuthSystem
tag: [MiniProject, auth, 회원가입, 로그인, Node.js, Express.js,MongoDB, Mongoose, JWT, bcrypt]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

> 회원가입부터 로그인까지, 단계별로 따라하며 안전한 인증 API 서버 만들기

## 🎯 프로젝트 개요

이번 포스트에서는 **Node.js**와 **Express**를 사용해서 완전한 인증 시스템을 구축해보겠습니다.

### 🛠️ 기술 스택

-   **Backend**: Node.js, Express.js
-   **Database**: MongoDB, Mongoose
-   **Authentication**: JWT (JSON Web Token)
-   **Password Encryption**: bcrypt
-   **API Testing**: Thunder Client

### 📋 구현할 기능

-   ✅ 사용자 회원가입 (비밀번호 암호화)
-   ✅ 사용자 로그인 (JWT 토큰 발급)
-   ✅ 입력값 검증 및 에러 처리
-   ✅ MongoDB 데이터 저장

---

## 🚀 Step 1: 프로젝트 초기 설정

### 1-1. 프로젝트 구조 생성

```bash
mkdir SimpleAuthSystem
cd SimpleAuthSystem
mkdir backend
cd backend
npm init -y
```

### 1-2. 필요한 패키지 설치

```bash
# 프로덕션 의존성
npm install express mongoose bcryptjs jsonwebtoken cors dotenv

# 개발 의존성
npm install -D nodemon
```

### 1-3. package.json 스크립트 추가

```json
{
    "scripts": {
        "start": "node app.js",
        "dev": "nodemon app.js"
    }
}
```

### 📁 최종 프로젝트 구조

```
backend/
├── models/
│   └── User.js
├── routes/
│   └── auth.js
├── .env
├── app.js
└── package.json
```

---

## 💾 Step 2: User 모델 설계

### 2-1. User 스키마 정의

`models/User.js` 파일을 생성합니다:

```javascript
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

// 사용자 스키마 정의
const userSchema = new mongoose.Schema({
    username: {
        type: String,
        required: [true, "사용자명을 입력해주세요"],
        unique: true,
        trim: true,
        minlength: [3, "사용자명은 최소 3글자 이상이어야 합니다"],
        maxlength: [20, "사용자명은 최대 20글자까지 가능합니다"],
    },

    email: {
        type: String,
        required: [true, "이메일을 입력해주세요"],
        unique: true,
        lowercase: true,
        trim: true,
    },

    password: {
        type: String,
        required: [true, "비밀번호를 입력해주세요"],
        minlength: [6, "비밀번호는 최소 6글자 이상이어야 합니다"],
    },

    createdAt: {
        type: Date,
        default: Date.now,
    },
});
```

### 🔍 User 모델 설명

1. **스키마 정의**: 사용자 데이터 구조를 정의
    - `username`: 사용자명 (3-20글자, 유니크)
    - `email`: 이메일 (유효성 검사, 유니크)
    - `password`: 비밀번호 (최소 6글자)
    - `createdAt`: 계정 생성일
2. **보안 기능**:
    - **비밀번호 암호화**: `bcrypt`로 해시화 (salt rounds: 12)
    - **비밀번호 검증**: `comparePassword` 메서드
    - **JSON 응답에서 비밀번호 제외**: 보안상 중요!

### 2-2. 비밀번호 자동 암호화 미들웨어

```javascript
// 비밀번호 저장 전 암호화
userSchema.pre("save", async function (next) {
    // 비밀번호가 변경되지 않았다면 다음으로 넘어감
    if (!this.isModified("password")) return next();

    try {
        // 비밀번호 해시화 (salt rounds: 12)
        const salt = await bcrypt.genSalt(12);
        this.password = await bcrypt.hash(this.password, salt);
        next();
    } catch (error) {
        next(error);
    }
});

// 비밀번호 검증 메서드
userSchema.methods.comparePassword = async function (candidatePassword) {
    return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model("User", userSchema);
```

### 🔐 암호화 동작 원리

| 입력 단계       | 값                               |
| --------------- | -------------------------------- |
| **사용자 입력** | `"123456"`                       |
| **Salt 생성**   | `"$2a$12$xyz123..."`             |
| **DB 저장**     | `"$2a$12$xyz123...복잡한해시값"` |

---

## 🌐 Step 3: MongoDB 설정 및 연결

### 3-1. MongoDB Atlas 계정 생성 및 클러스터 설정

MongoDB를 사용하기 위해서는 먼저 MongoDB Atlas에서 무료 클러스터를 생성해야 합니다.

#### **1단계: MongoDB Atlas 가입**

1. [MongoDB Atlas 공식 사이트](https://www.mongodb.com/atlas) 접속
2. 본인의 계정 생성

#### **2단계: 클러스터 생성**

1. **`DATABASE - Clusters`** 메뉴에서 **+Create** 클릭
2. **"FREE"** 옵션 선택
3. **Name**: 원하는 이름 입력 (예: auth-cluster)
4. **Cloud Provider**: AWS 선택
5. **Region**: 가장 가까운 지역 선택 (예: ap-northeast-2, Seoul)
6. **"Create Deployment"** 버튼 클릭

#### **3단계: 데이터베이스 사용자 생성**

1. **`SECURITY - Database Access`** 메뉴 클릭
2. **"Add New Database User"** 클릭
3. **Authentication Method**: Password 선택
4. **Username**: 사용자명 입력 (예: `authuser`)
5. **Password**: 강력한 비밀번호 생성 및 복사 (나중에 필요!)
6. **Database User Privileges**: "Read and write to any database" 선택
7. **"Add User"** 클릭

#### **4단계: 네트워크 접근 허용**

1. **`SECURITY - Network Access`** 메뉴 클릭
2. **"Add IP Address"** 클릭
3. **"Allow Access From Anywhere"** 클릭 (개발용, 실제 운영에서는 특정 IP만 허용)
4. **"Confirm"** 클릭

#### **5단계: 연결 문자열 복사**

1. **`DATABASE - Clusters`** 메뉴로 돌아가기

2. 클러스터에서 **"Connect"** 버튼 클릭

3. **"Drivers"** 선택

4. **Driver**: Node.js, **Version**: 4.1 or later 선택

5. 연결 문자열 복사:

    ```
    mongodb+srv://<username>:<password>@cluster-name.xxxxx.mongodb.net/?retryWrites=true&w=majority&appName=auth-cluster
    ```

6. `<username>`과 `<password>`, `cluster-name`을 실제 값으로 교체

### 🔍 연결 문자열 예시

```properties
# 원본 (교체 전)
mongodb+srv://<username>:<password>@auth-cluster.au04pzg.mongodb.net/?retryWrites=true&w=majority&appName=auth-cluster

# 실제 사용 (교체 후)
mongodb+srv://authuser:mySecurePassword123@auth-cluster.au04pzg.mongodb.net/?retryWrites=true&w=majority&appName=auth-cluster
```

### ⚠️ 주의사항

-   **비밀번호에 특수문자가 있다면** URL 인코딩 필요
    -   `@` → `%40`
    -   `#` → `%23`
    -   `&` → `%26`
-   **연결 문자열은 절대 Git에 커밋하지 마세요!**
-   **실제 운영환경에서는 특정 IP만 허용하세요**

---

## 🌐 Step 3: Express 서버 구축

### Express란 무엇인가?

**웹 요청을 받고 → 처리하고 → 응답하는 웹 서버를 쉽게 만들어주는 Node.js 도구**

마치 카페에서 손님의 주문을 받고, 커피를 만들어서, 완성된 커피를 전달하는 것처럼, 웹에서 사용자의 요청을 받고, 필요한 작업을 하고, 결과를 돌려주는 역할을 해요!

### 3-1. 환경변수 설정

`.env` 파일 생성:

```properties
# MongoDB 연결 문자열 (위에서 복사한 연결 문자열 사용)
MONGO_URI=mongodb+srv://authuser:mySecurePassword123@auth-cluster.au04pzg.mongodb.net/?retryWrites=true&w=majority&appName=auth-cluster

# JWT 토큰 암호화 키 (운영환경에서는 반드시 변경!)
JWT_SECRET=your-super-secret-jwt-key-at-least-32-characters-long-change-this-in-production

# 서버 포트
PORT=5000

# 환경 설정
NODE_ENV=development
```

### 🔐 JWT_SECRET 생성 방법

강력한 JWT 시크릿 키를 생성하려면:

```bash
# Node.js에서 랜덤 문자열 생성
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

# 또는 온라인 생성기 사용
# https://www.allkeysgenerator.com/Random/Security-Encryption-Key-Generator.aspx
```

### ⚠️ 보안 주의사항

-   **`.env` 파일을 Git에 커밋하지 마세요!**
-   **`.gitignore`에 `.env` 추가 필수 (깃허브에 올릴 경우에만 해당, `backend` 폴더에 `.gitignore` 파일 생성):**

### 3-2. Express 서버 설정

`app.js` 파일 생성:

```javascript
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const mongoose = require("mongoose");

// Express 앱 생성
const app = express();

// 미들웨어 설정
app.use(cors()); // CORS 허용
app.use(express.json()); // JSON 데이터 파싱

// MongoDB 연결
mongoose
    .connect(process.env.MONGO_URI)
    .then(() => console.log("MongoDB 연결 성공"))
    .catch((err) => console.error("DB 연결 실패:", err));

// 라우트 연결
const authRoutes = require("./routes/auth");
app.use("/api/auth", authRoutes);

// 기본 라우트
app.get("/", (req, res) => {
    res.json({ message: "🚀 서버가 실행 중입니다!" });
});

// 서버 시작
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
    console.log(`서버가 포트 ${PORT}에서 실행 중입니다!`);
});
```

### 🖥️ 서버 실행 결과

```bash
> npm run dev

서버가 포트 5000에서 실행 중입니다!
MongoDB 연결 성공
```

---

## 📝 Step 4: 회원가입 API 구현

### 4-1. 인증 라우터 생성

`routes/auth.js` 파일 생성:

```javascript
const express = require("express");
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const router = express.Router();

// 회원가입 API
router.post("/register", async (req, res) => {
    try {
        const { username, email, password } = req.body;

        // 1. 입력값 검증
        if (!username || !email || !password) {
            return res.status(400).json({
                success: false,
                message: "모든 필드를 입력해주세요.",
            });
        }

        // 2. 기존 사용자 확인
        const existingUser = await User.findOne({
            $or: [{ email }, { username }],
        });

        if (existingUser) {
            return res.status(409).json({
                success: false,
                message: "이미 존재하는 사용자입니다.",
            });
        }

        // 3. 새 사용자 생성
        const user = new User({ username, email, password });
        await user.save(); // 이때 자동으로 비밀번호 암호화됨!

        // 4. 성공 응답 (비밀번호 제외)
        res.status(201).json({
            success: true,
            message: "회원가입이 성공적으로 완료되었습니다!",
            user: {
                id: user._id,
                username: user.username,
                email: user.email,
            },
        });
    } catch (error) {
        console.error("회원가입 에러:", error);
        res.status(500).json({
            success: false,
            message: "서버 오류가 발생했습니다.",
        });
    }
});

module.exports = router;
```

### 4-2. Thunder Client 설치 및 설정

API를 테스트하기 위해 VS Code의 Thunder Client 확장을 사용하겠습니다.

#### **Thunder Client 설치 방법**

1. **VS Code 확장 마켓플레이스** 열기
    - `Ctrl + Shift + X` (Windows/Linux) 또는 `Cmd + Shift + X` (Mac)
2. **Thunder Client 검색 및 설치**

    - 검색창에 `Thunder Client` 입력
    - **rangav.vscode-thunder-client** 선택
    - **"Install"** 버튼 클릭

3. **Thunder Client 열기**
    - `Ctrl + Shift + P` 눌러서 명령 팔레트 열기
    - `Thunder Client: New Request` 입력 후 선택
    - 또는 왼쪽 사이드바에서 번개 아이콘 클릭

#### **Thunder Client 기본 사용법**

| 기능            | 설정 방법                                      |
| --------------- | ---------------------------------------------- |
| **HTTP Method** | 드롭다운에서 `POST` 선택                       |
| **URL**         | `http://localhost:5000/api/auth/register` 입력 |
| **Headers**     | `Content-Type: application/json` 추가          |
| **Body**        | `JSON` 탭에서 요청 데이터 입력                 |

#### **🔧 Thunder Client 설정 예시**

```
Method: POST
URL: http://localhost:5000/api/auth/register
Headers:
  Content-Type: application/json
Body (JSON):
{
  "username": "testuser",
  "email": "test@example.com",
  "password": "123456"
}
```

### ⚡ Thunder Client vs 다른 도구 비교

| 도구               | 장점                          | 단점                   |
| ------------------ | ----------------------------- | ---------------------- |
| **Thunder Client** | ✅ VS Code 내장, 가벼움, 무료 | ❌ 고급 기능 제한      |
| **Postman**        | ✅ 강력한 기능, 팀 협업       | ❌ 무겁고, 로그인 필요 |
| **Insomnia**       | ✅ 직관적 UI, GraphQL 지원    | ❌ 별도 설치 필요      |

### 4-3. API 테스트 - 회원가입

#### 📤 요청 (Request)

```http
POST http://localhost:5000/api/auth/register
Content-Type: application/json

{
  "username": "testuser",
  "email": "test@example.com",
  "password": "123456"
}
```

#### 📥 응답 (Response)

```json
{
    "success": true,
    "message": "회원가입이 성공적으로 완료되었습니다!",
    "user": {
        "id": "687f320610a810dd8cbbbdcb",
        "username": "testuser",
        "email": "test@example.com"
    }
}
```

#### 💾 MongoDB에 저장된 실제 데이터

```json
{
    "_id": "687f320610a810dd8cbbbdcb",
    "username": "testuser",
    "email": "test@example.com",
    "password": "$2a$12$xyz...암호화된비밀번호",
    "createdAt": "2025-07-22T15:30:45.123Z"
}
```

---

## 🔐 Step 5: 로그인 API 구현

### 5-1. JWT 로그인 API 추가

`routes/auth.js`에 로그인 엔드포인트 추가:

```javascript
// 로그인 API
router.post("/login", async (req, res) => {
    try {
        const { email, password } = req.body;

        // 1. 입력값 검증
        if (!email || !password) {
            return res.status(400).json({
                success: false,
                message: "이메일과 비밀번호를 입력해주세요.",
            });
        }

        // 2. 사용자 찾기
        const user = await User.findOne({ email });
        if (!user) {
            return res.status(401).json({
                success: false,
                message: "이메일 또는 비밀번호가 올바르지 않습니다.",
            });
        }

        // 3. 비밀번호 검증
        const isPasswordValid = await user.comparePassword(password);
        if (!isPasswordValid) {
            return res.status(401).json({
                success: false,
                message: "이메일 또는 비밀번호가 올바르지 않습니다.",
            });
        }

        // 4. JWT 토큰 생성
        const token = jwt.sign(
            { userId: user._id },
            process.env.JWT_SECRET,
            { expiresIn: "7d" } // 7일간 유효
        );

        // 5. 성공 응답
        res.json({
            success: true,
            message: "로그인 성공!",
            data: {
                user: {
                    id: user._id,
                    username: user.username,
                    email: user.email,
                },
                token,
            },
        });
    } catch (error) {
        console.error("로그인 에러:", error);
        res.status(500).json({
            success: false,
            message: "서버 오류가 발생했습니다.",
        });
    }
});
```

### 5-2. API 테스트 - 로그인

#### 📤 요청 (Request)

```http
POST http://localhost:5000/api/auth/login
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "123456"
}
```

#### 📥 응답 (Response)

```json
{
    "success": true,
    "message": "로그인 성공!",
    "data": {
        "user": {
            "id": "687f320610a810dd8cbbbdcb",
            "username": "testuser",
            "email": "test@example.com"
        },
        "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiI2ODdmMzIwNjEwYTgxMGRkOGNiYmJkY2IiLCJpYXQiOjE3MzcxMjM0NTYsImV4cCI6MTczNzgxNjI1Nn0.abc123def456..."
    }
}
```

---

## 🧪 API 테스트 시나리오

### ✅ 성공 케이스 테스트

| API          | 테스트 케이스          | 결과                  |
| ------------ | ---------------------- | --------------------- |
| **회원가입** | 유효한 정보로 가입     | ✅ 201 Created        |
| **로그인**   | 올바른 이메일/비밀번호 | ✅ 200 OK + JWT Token |

### ❌ 에러 케이스 테스트

| API          | 테스트 케이스        | HTTP Status        | 응답 메시지                                 |
| ------------ | -------------------- | ------------------ | ------------------------------------------- |
| **회원가입** | 필수 필드 누락       | `400 Bad Request`  | "모든 필드를 입력해주세요."                 |
| **회원가입** | 중복 이메일/사용자명 | `409 Conflict`     | "이미 존재하는 사용자입니다."               |
| **로그인**   | 존재하지 않는 이메일 | `401 Unauthorized` | "이메일 또는 비밀번호가 올바르지 않습니다." |
| **로그인**   | 잘못된 비밀번호      | `401 Unauthorized` | "이메일 또는 비밀번호가 올바르지 않습니다." |

---

## 🔒 보안 고려사항

### 1. 비밀번호 보안

-   ✅ **bcrypt 사용**: 단방향 해시 + 솔트
-   ✅ **높은 솔트 라운드**: 12라운드 사용
-   ✅ **평문 저장 금지**: DB에 해시값만 저장

### 2. JWT 토큰 보안

-   ✅ **강력한 시크릿 키**: 64자리 랜덤 문자열
-   ✅ **적절한 만료시간**: 7일 설정
-   ✅ **환경변수 사용**: 하드코딩 금지

### 3. 에러 메시지 보안

-   ✅ **정보 노출 방지**: "이메일 또는 비밀번호가 올바르지 않습니다"
-   ✅ **일관된 메시지**: 사용자 존재 여부 추측 불가

---

## 📊 완성된 기능 요약

### ✨ 구현 완료된 기능

-   [x] **사용자 회원가입**

    -   입력값 검증 (필수 필드, 길이 제한)
    -   중복 사용자 체크 (이메일, 사용자명)
    -   비밀번호 자동 암호화 (bcrypt)

-   [x] **사용자 로그인**

    -   이메일/비밀번호 검증
    -   JWT 토큰 발급 (7일 유효)
    -   보안적인 에러 메시지

-   [x] **데이터베이스 연동**

    -   MongoDB 연결
    -   Mongoose 스키마 설계
    -   실제 데이터 저장/조회

-   [x] **API 서버**
    -   Express.js 서버 구축
    -   CORS 설정
    -   JSON 요청/응답 처리

### 🎯 API 엔드포인트

| Method | Endpoint             | 기능           | 인증 필요 |
| ------ | -------------------- | -------------- | --------- |
| `POST` | `/api/auth/register` | 회원가입       | ❌        |
| `POST` | `/api/auth/login`    | 로그인         | ❌        |
| `GET`  | `/`                  | 서버 상태 확인 | ❌        |

---

## 🚀 다음 단계 미리보기

### Part 2에서 다룰 내용:

-   🛡️ **JWT 인증 미들웨어** 구현
-   🔐 **보호된 라우트** 만들기 (`/api/auth/me`)
-   🔄 **토큰 갱신** 기능
-   📝 **사용자 정보 수정** API

### Part 3에서 다룰 내용:

-   ⚛️ **React 프론트엔드** 연동
-   📱 **로그인/회원가입 폼** 제작
-   🎨 **UI/UX 개선**
-   🔄 **상태 관리** (Context API)

---

## 💡 핵심 포인트 정리

### 🎓 배운 것들

1. **Node.js 백엔드** 아키텍처 설계
2. **bcrypt**를 이용한 안전한 비밀번호 관리
3. **JWT**를 이용한 stateless 인증
4. **MongoDB** 데이터 모델링
5. **Express.js** API 서버 구축
6. **에러 처리** 및 **보안** 고려사항

### ⚠️ 주의사항

-   환경변수 (`.env`) 파일은 **절대 Git에 커밋하지 마세요**
-   JWT 시크릿 키는 **운영환경에서 반드시 변경**하세요
-   비밀번호는 **절대 평문으로 저장하지 마세요**
-   API 응답에서 **민감한 정보를 제외**하세요

---

## 📚 참고 자료

-   [Express.js 공식 문서](https://expressjs.com/)
-   [Mongoose 공식 문서](https://mongoosejs.com/)
-   [bcrypt npm 패키지](https://www.npmjs.com/package/bcryptjs)
-   [jsonwebtoken npm 패키지](https://www.npmjs.com/package/jsonwebtoken)
-   [JWT.io - JWT 토큰 디버거](https://jwt.io/)

---

## 🎉 마무리

이번 포스트에서는 **Node.js + Express**를 사용해서 완전한 JWT 인증 시스템을 구축해보았습니다.

**회원가입부터 로그인까지**, 실제 운영 환경에서 사용할 수 있는 수준의 보안과 에러 처리를 구현했습니다.

다음 편에서는 **토큰 인증 미들웨어**와 **보호된 라우트**를 구현해서 더욱 완성도 높은 인증 시스템을 만들어보겠습니다!

**궁금한 점이나 개선사항이 있다면 댓글로 남겨주세요!** 🙋‍♂️

---

### 🔗 소스코드

전체 소스코드는 [GitHub 저장소](https://github.com/hoondongseo/SimpleAuthSystem)에서 확인하실 수 있습니다.

```bash
git clone https://github.com/hoondongseo/SimpleAuthSystem.git
cd SimpleAuthSystem/backend
npm install
npm run dev
```
