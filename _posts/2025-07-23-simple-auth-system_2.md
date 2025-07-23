---
title: "Node.js JWT 인증 시스템 완성하기 Part 2: 미들웨어, 리프레시 토큰, 고급 보안 기능"
categories:
- MiniProject
- SimpleAuthSystem
tag: [MiniProject, auth, 회원가입, 로그인, Node.js, Express.js,MongoDB, Mongoose, JWT, bcrypt, middleware, Token]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

> Part 1에서 구축한 기본 JWT 인증을 한 단계 업그레이드! 실무급 보안과 편의성을 모두 잡는 완전한 인증 시스템

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 2**입니다.

- **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple-auth-system_1/)** ← 먼저 읽어주세요!
- **Part 2: 고급 JWT 인증 기능** ← 현재 글

---

## 🎯 Part 2에서 다룰 내용

Part 1에서 기본적인 회원가입과 로그인 API를 만들었다면, 이번에는 **실무에서 필수**인 고급 기능들을 구현해보겠습니다.

### 🔥 구현할 기능들

- ✅ **JWT 인증 미들웨어** - 보호된 라우트 구현
- ✅ **리프레시 토큰 시스템** - 보안과 편의성 동시에
- ✅ **토큰 갱신 API** - 자동 로그인 연장
- ✅ **안전한 로그아웃** - 토큰 무효화

### 🤔 왜 이런 기능들이 필요할까요?

#### **Part 1의 한계점들:**
- JWT 토큰 만료시간이 7일 → 보안 위험
- 토큰 탈취 시 대응 방법 없음
- 사용자가 매번 로그인해야 하는 불편함
- 로그아웃 해도 토큰이 유효한 문제

#### **Part 2의 해결책:**
- **짧은 Access Token** (15분) + **긴 Refresh Token** (30일)
- **토큰 갱신 시스템**으로 자동 연장
- **DB 저장**으로 토큰 관리 및 무효화 가능

---

## 🛡️ Step 1: JWT 인증 미들웨어 구현

먼저 토큰을 검증하고 보호된 라우트를 만들 수 있는 미들웨어를 구현해보겠습니다.

### 1-1. 미들웨어 파일 생성

`middleware/auth.js` 파일을 생성합니다:

```javascript
const jwt = require("jsonwebtoken");
const User = require("../models/User");

// JWT 토큰 검증 미들웨어
const authenticateToken = async (req, res, next) => {
  try {
    // 1. Authorization 헤더에서 토큰 추출
    const authHeader = req.headers.authorization;
    const token = authHeader && authHeader.split(" ")[1]; // "Bearer TOKEN"

    if (!token) {
      return res.status(401).json({
        success: false,
        message: "액세스 토큰이 필요합니다."
      });
    }

    // 2. 토큰 검증
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // 3. 사용자 정보 조회
    const user = await User.findById(decoded.userId).select("-password");
    if (!user) {
      return res.status(401).json({
        success: false,
        message: "유효하지 않은 토큰입니다."
      });
    }

    // 4. req 객체에 사용자 정보 추가
    req.user = user;
    next();

  } catch (error) {
    console.error("토큰 검증 에러:", error);

    if (error.name === "TokenExpiredError") {
      return res.status(401).json({
        success: false,
        message: "토큰이 만료되었습니다."
      });
    }

    if (error.name === "JsonWebTokenError") {
      return res.status(401).json({
        success: false,
        message: "유효하지 않은 토큰입니다."
      });
    }

    res.status(500).json({
      success: false,
      message: "서버 오류가 발생했습니다."
    });
  }
};

module.exports = { authenticateToken };
```

### 1-2. 보호된 라우트 추가

`routes/auth.js`에 미들웨어를 import하고 보호된 라우트를 추가합니다:

```javascript
const { authenticateToken } = require("../middleware/auth");

// GET /api/auth/me - 현재 사용자 정보 조회
router.get("/me", authenticateToken, (req, res) => {
  try {
    // authenticateToken 미들웨어가 req.user에 사용자 정보를 넣어줌!
    res.json({
      success: true,
      message: "사용자 정보 조회 성공",
      user: {
        id: req.user._id,
        username: req.user.username,
        email: req.user.email,
        createdAt: req.user.createdAt
      }
    });
  } catch (error) {
    console.error("사용자 정보 조회 에러:", error);
    res.status(500).json({
      success: false,
      message: "서버 오류가 발생했습니다."
    });
  }
});
```

### 🧪 미들웨어 테스트

#### 📤 요청 (Thunder Client)
```http
GET http://localhost:5000/api/auth/me
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 📥 성공 응답
```json
{
  "success": true,
  "message": "사용자 정보 조회 성공",
  "user": {
    "id": "687f320610a810dd8cbbbdcb",
    "username": "testuser",
    "email": "test@example.com",
    "createdAt": "2025-01-18T..."
  }
}
```

---

## 🔄 Step 2: 리프레시 토큰 시스템 설계

### 2-1. 리프레시 토큰이란?

#### **기존 방식 (Part 1):**
```
로그인 → JWT 토큰 발급 (7일) → 만료 시 재로그인
```

#### **개선된 방식 (Part 2):**
```
로그인 → Access Token (15분) + Refresh Token (30일)
       → Access 만료 시 Refresh로 새 토큰 발급
       → 30일 후에만 재로그인 필요
```

### 2-2. 이중 토큰 구조의 장점

| 구분 | Access Token | Refresh Token |
|------|--------------|---------------|
| **유효기간** | 15분 (짧음) | 30일 (길음) |
| **용도** | API 접근 | 토큰 갱신 |
| **저장위치** | 메모리/로컬스토리지 | DB + 클라이언트 |
| **보안성** | 높음 (짧은 수명) | 중간 (DB 검증) |

### 2-3. User 모델 업데이트

`models/User.js`에 refreshToken 필드와 관련 메서드를 추가합니다:

```javascript
const userSchema = new mongoose.Schema({
  username: { ... },
  email: { ... },
  password: { ... },
  
  // 리프레시 토큰 필드 추가
  refreshToken: {
    type: String,
    default: null
  },
  
  createdAt: { ... }
});

// 리프레시 토큰 저장
userSchema.methods.saveRefreshToken = async function(token) {
  this.refreshToken = token;
  return await this.save();
};

// 리프레시 토큰 삭제 (로그아웃 시)
userSchema.methods.clearRefreshToken = async function() {
  this.refreshToken = null;
  return await this.save();
};
```

---

## 🔐 Step 3: 로그인 API 개선 (이중 토큰 발급)

기존 로그인 API를 수정해서 두 종류의 토큰을 동시에 발급하도록 개선합니다.

### 3-1. 개선된 로그인 API

```javascript
// 로그인 API (개선 버전)
router.post("/login", async (req, res) => {
  try {
    const { email, password } = req.body;

    // ... 입력값 검증 및 사용자 확인 로직 ...

    // 4. JWT 토큰 생성 (이중 토큰)
    const accessToken = jwt.sign(
      { userId: user._id, type: "access" },
      process.env.JWT_SECRET,
      { expiresIn: "15m" }  // 15분
    );

    const refreshToken = jwt.sign(
      { userId: user._id, type: "refresh" },
      process.env.JWT_SECRET,
      { expiresIn: "30d" }  // 30일
    );

    // 5. 리프레시 토큰을 DB에 저장
    await user.saveRefreshToken(refreshToken);

    // 6. 성공 응답 (두 토큰 모두 반환)
    res.json({
      success: true,
      message: "로그인 성공!",
      data: {
        user: {
          id: user._id,
          username: user.username,
          email: user.email
        },
        accessToken,    // 15분 유효
        refreshToken    // 30일 유효
      }
    });

  } catch (error) {
    // ... 에러 처리 ...
  }
});
```

### 🧪 개선된 로그인 테스트

#### 📤 요청
```http
POST http://localhost:5000/api/auth/login
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "123456"
}
```

#### 📥 응답
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
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

## 🔄 Step 4: 토큰 갱신 API 구현

Access Token이 만료되었을 때 Refresh Token으로 새로운 Access Token을 발급받는 API를 구현합니다.

### 4-1. 토큰 갱신 API

```javascript
// POST /api/auth/refresh - 토큰 갱신
router.post("/refresh", async (req, res) => {
  try {
    const { refreshToken } = req.body;

    // 1. 리프레시 토큰 확인
    if (!refreshToken) {
      return res.status(401).json({
        success: false,
        message: "리프레시 토큰이 필요합니다."
      });
    }

    // 2. 토큰 검증
    const decoded = jwt.verify(refreshToken, process.env.JWT_SECRET);

    // 3. DB에서 사용자 및 저장된 토큰 확인
    const user = await User.findById(decoded.userId);
    if (!user || user.refreshToken !== refreshToken) {
      return res.status(401).json({
        success: false,
        message: "유효하지 않은 리프레시 토큰입니다."
      });
    }

    // 4. 새로운 액세스 토큰 발급
    const newAccessToken = jwt.sign(
      { userId: user._id, type: "access" },
      process.env.JWT_SECRET,
      { expiresIn: "15m" }
    );

    // 5. 성공 응답
    res.json({
      success: true,
      message: "토큰 갱신 성공",
      accessToken: newAccessToken
    });

  } catch (error) {
    console.error("토큰 갱신 에러:", error);

    if (error.name === "TokenExpiredError") {
      return res.status(401).json({
        success: false,
        message: "리프레시 토큰이 만료되었습니다."
      });
    }

    if (error.name === "JsonWebTokenError") {
      return res.status(401).json({
        success: false,
        message: "유효하지 않은 리프레시 토큰입니다."
      });
    }

    res.status(500).json({
      success: false,
      message: "서버 오류가 발생했습니다."
    });
  }
});
```

### 🧪 토큰 갱신 테스트

#### 📤 요청
```http
POST http://localhost:5000/api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### 📥 응답
```json
{
  "success": true,
  "message": "토큰 갱신 성공",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## 🚪 Step 5: 안전한 로그아웃 API

로그아웃 시 DB에서 Refresh Token을 삭제하여 완전히 무효화하는 API를 구현합니다.

### 5-1. 로그아웃 API

```javascript
// POST /api/auth/logout - 로그아웃
router.post("/logout", authenticateToken, async (req, res) => {
  try {
    // req.user는 authenticateToken 미들웨어가 제공
    await req.user.clearRefreshToken();

    res.json({
      success: true,
      message: "로그아웃이 완료되었습니다."
    });

  } catch (error) {
    console.error("로그아웃 에러:", error);
    res.status(500).json({
      success: false,
      message: "서버 오류가 발생했습니다."
    });
  }
});
```

### 🧪 로그아웃 테스트

#### 📤 요청
```http
POST http://localhost:5000/api/auth/logout
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 📥 응답
```json
{
  "success": true,
  "message": "로그아웃이 완료되었습니다."
}
```

---

## 🧪 전체 시스템 통합 테스트

### 🎯 완전한 테스트 시나리오

#### **1단계: 로그인**
- 이메일/비밀번호로 로그인
- `accessToken`, `refreshToken` 받기

#### **2단계: 보호된 리소스 접근**
- `accessToken`으로 `/me` API 호출
- 사용자 정보 조회 성공

#### **3단계: 토큰 갱신**
- `refreshToken`으로 새 `accessToken` 발급
- 기존 `accessToken` 만료되어도 계속 사용 가능

#### **4단계: 로그아웃**
- `accessToken`으로 로그아웃
- DB에서 `refreshToken` 삭제 확인

### 📊 테스트 결과 요약

| API | 테스트 케이스 | 결과 |
|-----|--------------|------|
| **로그인** | 이중 토큰 발급 | ✅ Pass |
| **/me** | Access Token 인증 | ✅ Pass |
| **토큰 갱신** | Refresh Token으로 갱신 | ✅ Pass |
| **로그아웃** | Refresh Token 삭제 | ✅ Pass |

---

## 🔒 보안 강화 포인트

### 1. 토큰 수명 관리
- **Access Token**: 15분 → 탈취 위험 최소화
- **Refresh Token**: 30일 → 사용자 편의성 확보

### 2. DB 기반 토큰 검증
- Refresh Token을 DB에 저장
- 로그아웃 시 즉시 무효화 가능
- 의심스러운 활동 시 강제 로그아웃 가능

### 3. 토큰 타입 분리
- Access/Refresh 토큰 구분
- 각각 다른 용도로 사용 제한

### 4. 에러 처리 강화
- 토큰별 상세한 에러 메시지
- 보안상 민감한 정보 노출 방지

---

## 📊 최종 API 명세서

### 🎯 완성된 API 엔드포인트

| Method | Endpoint | 기능 | 인증 필요 | 토큰 타입 |
|--------|----------|------|-----------|-----------|
| `POST` | `/api/auth/register` | 회원가입 | ❌ | - |
| `POST` | `/api/auth/login` | 로그인 | ❌ | - |
| `GET` | `/api/auth/me` | 사용자 정보 조회 | ✅ | Access Token |
| `POST` | `/api/auth/refresh` | 토큰 갱신 | ❌ | Refresh Token |
| `POST` | `/api/auth/logout` | 로그아웃 | ✅ | Access Token |

### 🔄 토큰 흐름도

```
1. 로그인
   ↓
2. Access Token (15분) + Refresh Token (30일) 발급
   ↓
3. Access Token으로 API 호출
   ↓
4. Access Token 만료 시
   ↓
5. Refresh Token으로 새 Access Token 발급
   ↓
6. 반복 (30일간)
   ↓
7. 로그아웃 시 Refresh Token 삭제
```

---

## 📁 최종 프로젝트 구조

```
backend/
├── middleware/
│   └── auth.js          # JWT 인증 미들웨어
├── models/
│   └── User.js          # refreshToken 필드 추가
├── routes/
│   └── auth.js          # 모든 인증 API
├── .env                 # 환경 변수
├── app.js              # Express 서버
└── package.json        # 의존성 관리
```

---

## 🚀 다음 단계 미리보기

### Part 3에서 다룰 내용:
- ⚛️ **React 프론트엔드** 구축
- 🔄 **자동 토큰 갱신** 구현 (axios interceptor)
- 🎨 **로그인/회원가입 UI** 제작
- 📱 **반응형 사용자 대시보드**
- 🔐 **Protected Routes** (React Router)

### 고급 기능 아이디어:
- 📧 **이메일 인증** 시스템
- 🔑 **비밀번호 재설정** 기능
- 👥 **역할 기반 권한** (RBAC)
- 🌐 **소셜 로그인** (Google, GitHub)

---

## 💡 핵심 포인트 정리

### 🎓 Part 2에서 배운 것들

1. **JWT 미들웨어** 패턴으로 코드 재사용성 향상
2. **이중 토큰 구조**로 보안과 편의성 동시 확보
3. **DB 기반 토큰 관리**로 완전한 세션 제어
4. **RESTful API 설계** 원칙에 따른 엔드포인트 구성
5. **실무급 에러 처리** 및 보안 고려사항

### ⚠️ 운영 시 주의사항

- **JWT_SECRET**은 충분히 복잡하게 생성
- **HTTPS** 환경에서만 토큰 전송
- **Rate Limiting** 적용으로 무차별 대입 공격 방지
- **로그 모니터링**으로 의심스러운 활동 감지
- **정기적인 Refresh Token 로테이션** 고려

---

## 📚 참고 자료

- [JWT 공식 문서](https://jwt.io/)
- [Express.js 미들웨어 가이드](https://expressjs.com/en/guide/using-middleware.html)
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [OWASP JWT 보안 가이드](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

---

## 🎉 마무리

Part 2에서는 Part 1의 기본 JWT 인증을 **실무급 수준**으로 끌어올렸습니다!

이제 여러분이 만든 인증 시스템은:
- ✅ **보안성**: 짧은 Access Token으로 위험 최소화
- ✅ **편의성**: 자동 토큰 갱신으로 끊김 없는 사용자 경험
- ✅ **확장성**: 미들웨어 패턴으로 코드 재사용성 확보
- ✅ **실무성**: 로그아웃, 토큰 관리 등 필수 기능 완비

**Part 3에서는 React 프론트엔드와 연동해서 완전한 풀스택 애플리케이션을 완성해보겠습니다!**

---

### 🔗 소스코드

전체 소스코드는 [GitHub 저장소](https://github.com/hoondongseo/SimpleAuthSystem)에서 확인하실 수 있습니다.

```bash
# Part 2 브랜치 클론
git clone https://github.com/hoondongseo/SimpleAuthSystem.git
cd SimpleAuthSystem
git checkout feature/refresh-token
cd backend
npm install
npm run dev
```

---

**다음 편도 기대해주세요!** 🚀
