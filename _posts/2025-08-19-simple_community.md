---
title: "MERN 스택으로 간단한 커뮤니티 만들기 - 핵심 API와 React 연동"
categories:
  - Portfolio
  - Fullstack
tags:
  [
    React,
    Node.js,
    Express,
    MongoDB,
    MERN,
    JWT,
    게시판,
    Fullstack,
  ]
author_profile: false
sidebar:
  nav: "docs"
search: true
---

> React와 Node.js(Express)를 이용해 익명/회원제 기반의 간단한 커뮤니티 사이트의 핵심 백엔드 API를 구축하고, React 프론트엔드와 연동하는 과정을 상세히 기록합니다.

## 1. 프로젝트 개요

-   **목표**: 익명 기반 커뮤니티의 핵심 기능을 MERN 스택으로 구현
-   **주요 기능**:
    -   **인증**: 익명(IP 기반) 및 일반 회원가입/로그인 (JWT)
    -   **게시글**: 전체 CRUD (생성, 조회, 수정, 삭제)
    -   **댓글**: 게시글별 CRUD (생성, 조회, 삭제)

## 2. 기술 스택

| 구분     | 기술                                   |
| :------- | :------------------------------------- |
| 프론트엔드 | React, React Router, Axios             |
| 백엔드   | Node.js, Express                       |
| 데이터베이스 | MongoDB (Mongoose ODM)                 |
| 인증     | JSON Web Token (jsonwebtoken), bcrypt  |
| 환경 관리  | dotenv                                 |

## 3. 백엔드 API 구현

### 3.1. 모델링 (Mongoose Schemas)

-   **`User.js`**: 일반 사용자와 익명 사용자를 `isAnonymous` 필드로 구분. 익명 유저는 `ip`를, 일반 유저는 `username`, `email`, `password`를 필수 필드로 가집니다.
-   **`Post.js`**: `title`, `content`와 함께 작성자(`author`)를 `User` 모델과 연결합니다.
-   **`Comment.js`**: `content`, 작성자(`author`), 그리고 소속된 게시글(`post`)을 각각의 모델과 연결합니다.

### 3.2. 라우팅 (Express Router)

API 엔드포인트는 기능별로 모듈화하여 관리합니다.

-   **인증 (`/api/auth`)**:
    -   `POST /register`: 회원가입
    -   `POST /login`: 로그인
    -   `POST /anonymous`: 익명 토큰 발급
-   **게시글 (`/api/posts`)**:
    -   `GET /`: 전체 목록 조회
    -   `GET /:id`: 상세 조회
    -   `POST /`: 생성 (인증 필요)
    -   `PUT /:id`: 수정 (인증 및 소유권 확인)
    -   `DELETE /:id`: 삭제 (인증 및 소유권 확인)
-   **댓글 (`/api/posts/:postId/comments`)**:
    -   `GET /`: 특정 게시글의 댓글 목록 조회
    -   `POST /`: 댓글 생성 (인증 필요)
    -   `DELETE /:commentId`: 댓글 삭제 (인증 및 소유권 확인)

### 3.3. 인증 미들웨어

`Authorization: Bearer <token>` 헤더로 전달된 JWT를 검증하고, 유효한 경우 `req.user`에 사용자 정보를 주입하여 보호된 라우트에서 사용합니다.

```javascript
// middleware/authMiddleware.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: '토큰이 없습니다.' });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET || 'secretkey');
    next();
  } catch (err) {
    res.status(401).json({ error: '토큰이 유효하지 않습니다.' });
  }
};
```

## 4. 프론트엔드 구현 (React)

### 4.1. API 통신 설정

`axios`를 사용해 API 클라이언트를 설정합니다. 요청 인터셉터를 이용해 `localStorage`에 저장된 토큰을 모든 요청 헤더에 자동으로 추가합니다.

```javascript
// src/api/axios.js
import axios from 'axios';

const instance = axios.create({
  baseURL: 'http://localhost:5000/api',
});

instance.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default instance;
```

### 4.2. 페이지 라우팅

`react-router-dom`을 사용해 SPA(Single Page Application)의 페이지 전환을 관리합니다.

```jsx
// src/App.js
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
// ... Page 컴포넌트 import

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<LoginPage />} />
        <Route path="/register" element={<RegisterPage />} />
        <Route path="/posts" element={<PostsPage />} />
        <Route path="/posts/new" element={<NewPostPage />} />
        <Route path="/posts/:id" element={<PostDetailPage />} />
        <Route path="/posts/:id/edit" element={<EditPostPage />} />
      </Routes>
    </Router>
  );
}
```

### 4.3. 핵심 페이지 기능

-   **`LoginPage.js`**: 일반 로그인과 익명 로그인 기능을 제공하고, 성공 시 토큰을 `localStorage`에 저장 후 게시글 목록으로 이동합니다.
-   **`PostsPage.js`**: `useEffect`로 컴포넌트 마운트 시 게시글 목록을 불러와 화면에 렌더링합니다.
-   **`PostDetailPage.js`**: 게시글 내용과 함께 댓글 목록을 불러옵니다. 댓글 작성 폼과 삭제 버튼을 통해 실시간으로 댓글을 추가/삭제할 수 있습니다.
-   **`NewPostPage.js` / `EditPostPage.js`**: 게시글을 작성하고 수정하는 폼 UI와 API 호출 로직을 구현합니다.

## 5. 결론 및 향후 과제

현재까지 익명/회원제 인증을 기반으로 한 게시판의 모든 핵심 CRUD 기능이 백엔드와 프론트엔드 양단에 구현되었습니다.

-   **향후 과제**:
    1.  **UI/UX 개선**: 기본 스타일을 넘어 CSS 프레임워크(e.g., Material-UI, Tailwind CSS)를 적용하여 사용성을 높입니다.
    2.  **파일 업로드**: `multer`를 이용해 게시글에 이미지 첨부 기능을 추가합니다.
    3.  **상태 관리**: 전역 상태 관리 라이브러리(Context API, Redux)를 도입하여 사용자 인증 상태를 더 효율적으로 관리합니다.
    4.  **배포**: Vercel(프론트엔드)과 Railway(백엔드) 등을 이용해 실제 서비스로 배포합니다.
