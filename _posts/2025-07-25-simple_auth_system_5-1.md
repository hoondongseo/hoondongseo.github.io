---
title: "Node.js JWT 인증 시스템 완성하기 Part 5-1: 자동 토큰 갱신 & React Router 구현"
categories:
- MiniProject
- SimpleAuthSystem
tag: [MiniProject, auth, 회원가입, 로그인, 토큰, 라우터]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 5-1**입니다.

- **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple_auth_system_1/)**
- **[Part 2: 고급 JWT 인증 기능](https://hoondongseo.github.io/posts/simple_auth_system_2/)**
- **[Part 3: React 프론트엔드 구현](https://hoondongseo.github.io/posts/simple_auth_system_3/)**
- **[Part 4: 대시보드 및 로그인 상태 관리](https://hoondongseo.github.io/posts/simple_auth_system_4/)**
- **Part 5-1: 자동 토큰 갱신 & React Router** ← 현재 글

## 🎯 이번 편에서 구현할 것들

이번 Part 5-1에서는 사용자 경험을 한 단계 업그레이드 할 두 가지 핵심 기능을 구현합니다:

1. **자동 토큰 갱신**: API 요청 시 401 에러가 발생하면 자동으로 토큰을 갱신하고 원래 요청을 재시도
2. **React Router**: 모던 SPA 라우팅 시스템 도입으로 더 나은 사용자 경험 제공

## 🔄 자동 토큰 갱신 시스템

### 현재 문제점

기존 시스템에서는 액세스 토큰이 만료되면 사용자가 수동으로 로그아웃하고 다시 로그인해야 했습니다. 이는 매우 불편한 사용자 경험을 제공했죠.

### 해결 방법: Axios Response Interceptor

Axios의 Response Interceptor를 활용하여 모든 API 응답을 가로채고, 401 에러 발생 시 자동으로 토큰을 갱신하도록 구현합니다.

### 1. API 설정 업그레이드

**📁 frontend/src/api/api.js** - 기존 코드

```javascript
import axios from 'axios';

const API_BASE_URL = 'http://localhost:5000/api';

const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor: 모든 요청에 토큰 추가
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

export default api;
```

**📁 frontend/src/api/api.js** - 업그레이드 후

```javascript
import axios from 'axios';

const API_BASE_URL = 'http://localhost:5000/api';

// AuthContext import (나중에 추가)
let authContextRef = null;

export const setAuthContext = (authContext) => {
  authContextRef = authContext;
};

const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor: 모든 요청에 토큰 추가
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor: 401 에러 시 토큰 갱신
api.interceptors.response.use(
  (response) => response, // 성공 응답은 그대로 통과
  async (error) => {
    const originalRequest = error.config;

    // 401 에러이고, 아직 재시도하지 않은 요청인 경우
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        if (!refreshToken) {
          throw new Error('No refresh token available');
        }

        // 리프레시 토큰으로 새 액세스 토큰 요청
        const response = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refreshToken: refreshToken,
        });

        const { accessToken } = response.data;

        // 새 토큰을 localStorage에 저장
        localStorage.setItem('token', accessToken);

        // AuthContext의 토큰도 업데이트
        if (authContextRef && authContextRef.setToken) {
          authContextRef.setToken(accessToken);
        }

        // 원래 요청에 새 토큰 설정하고 재시도
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return api(originalRequest);

      } catch (refreshError) {
        // 리프레시 토큰도 만료된 경우 로그아웃 처리
        console.error('Token refresh failed:', refreshError);
        
        localStorage.removeItem('token');
        localStorage.removeItem('refreshToken');
        localStorage.removeItem('user');

        if (authContextRef && authContextRef.logout) {
          authContextRef.logout();
        }

        // 로그인 페이지로 리다이렉트 (React Router 사용)
        window.location.href = '/login';
        
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export default api;
```

### 2. AuthContext와 연동

자동 토큰 갱신이 AuthContext와 연동되도록 설정합니다.

**📁 frontend/src/context/AuthContext.js** - 업데이트

```javascript
import React, { createContext, useState, useContext, useEffect } from 'react';
import { setAuthContext } from '../api/api'; // 추가

const AuthContext = createContext();

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // API 모듈에 AuthContext 참조 전달
    setAuthContext({
      setToken,
      logout,
    });

    const storedUser = localStorage.getItem('user');
    const storedToken = localStorage.getItem('token');

    if (storedUser && storedToken) {
      setUser(JSON.parse(storedUser));
      setToken(storedToken);
    }
    setLoading(false);
  }, []);

  const login = (userData, userToken) => {
    setUser(userData);
    setToken(userToken);
    localStorage.setItem('user', JSON.stringify(userData));
    localStorage.setItem('token', userToken);
  };

  const logout = () => {
    setUser(null);
    setToken(null);
    localStorage.removeItem('user');
    localStorage.removeItem('token');
    localStorage.removeItem('refreshToken');
  };

  const value = {
    user,
    token,
    loading,
    login,
    logout,
    setToken, // 추가: 토큰만 업데이트하는 메서드
    isAuthenticated: !!user && !!token,
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};
```

## 🛣️ React Router 도입

### 기존 문제점

현재 App.js에서는 조건부 렌더링으로 컴포넌트를 전환하고 있어, 브라우저의 뒤로가기/앞으로가기 버튼이 작동하지 않고 URL이 변경되지 않습니다.

### 해결 방법: React Router DOM

모던 SPA의 표준인 React Router를 도입하여 진정한 싱글 페이지 애플리케이션을 구현합니다.

### 1. 패키지 설치 및 버전 관리

React 19와의 호환성 문제로 react-router-dom을 v6로 다운그레이드합니다.

```bash
npm install react-router-dom@^6.26.1
```

**📁 frontend/package.json** - 의존성 업데이트

```json
{
  "dependencies": {
    "react-router-dom": "^6.26.1"
    // ... 기타 의존성
  }
}
```

### 2. App.js 라우터 구조로 변경

**📁 frontend/src/App.js** - 기존 코드

```javascript
import React from 'react';
import { AuthProvider, useAuth } from './context/AuthContext';
import LoginForm from './components/Auth/LoginForm';
import RegisterForm from './components/Auth/RegisterForm';
import Dashboard from './components/Dashboard/Dashboard';
import './App.css';

function AppContent() {
  const { user, loading } = useAuth();
  const [isLogin, setIsLogin] = React.useState(true);

  if (loading) {
    return <div className="loading">로딩 중...</div>;
  }

  if (user) {
    return <Dashboard />;
  }

  return (
    <div className="App">
      {isLogin ? (
        <LoginForm onSwitchToRegister={() => setIsLogin(false)} />
      ) : (
        <RegisterForm onSwitchToLogin={() => setIsLogin(true)} />
      )}
    </div>
  );
}

function App() {
  return (
    <AuthProvider>
      <AppContent />
    </AuthProvider>
  );
}

export default App;
```

**📁 frontend/src/App.js** - 라우터 구조로 변경

```javascript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import LoginForm from './components/Auth/LoginForm';
import RegisterForm from './components/Auth/RegisterForm';
import Dashboard from './components/Dashboard/Dashboard';
import ProtectedRoute from './components/ProtectedRoute';
import './App.css';

function App() {
  return (
    <AuthProvider>
      <Router>
        <div className="App">
          <Routes>
            {/* 기본 경로: 대시보드 또는 로그인으로 리다이렉트 */}
            <Route path="/" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
            
            {/* 인증 관련 라우트 */}
            <Route path="/login" element={<LoginForm />} />
            <Route path="/register" element={<RegisterForm />} />
            
            {/* 보호된 라우트 */}
            <Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
            
            {/* 404 처리 */}
            <Route path="*" element={<Navigate to="/" replace />} />
          </Routes>
        </div>
      </Router>
    </AuthProvider>
  );
}

export default App;
```

### 3. ProtectedRoute 컴포넌트 생성

인증이 필요한 라우트를 보호하는 컴포넌트를 만듭니다.

**📁 frontend/src/components/ProtectedRoute.jsx** - 새로 생성

```javascript
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

const ProtectedRoute = ({ children }) => {
  const { isAuthenticated, loading } = useAuth();
  const location = useLocation();

  // 로딩 중일 때는 로딩 화면 표시
  if (loading) {
    return (
      <div style={% raw %}{{
        display: 'flex',
        justifyContent: 'center',
        alignItems: 'center',
        height: '100vh',
        fontSize: '1.2rem',
        color: '#666'
      }}{% endraw %}>
        로딩 중...
      </div>
    );
  }

  // 인증되지 않은 경우 로그인 페이지로 리다이렉트
  // 현재 경로를 state로 저장하여 로그인 후 원래 페이지로 돌아갈 수 있게 함
  if (!isAuthenticated) {
    return <Navigate to="/login" state={% raw %}{{ from: location }}{% endraw %} replace />;
  }

  // 인증된 경우 요청된 컴포넌트 렌더링
  return children;
};

export default ProtectedRoute;
```

### 4. 인증 폼 컴포넌트 업데이트

기존의 props 기반 네비게이션을 React Router의 `useNavigate` Hook으로 변경합니다.

**📁 frontend/src/components/Auth/LoginForm.jsx** - 업데이트

```javascript
import React, { useState } from 'react';
import { useNavigate, Link, useLocation } from 'react-router-dom'; // 추가
import { useAuth } from '../../context/AuthContext';
import api from '../../api/api';
import './Auth.css';

const LoginForm = () => { // props 제거
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const { login } = useAuth();
  const navigate = useNavigate(); // 추가
  const location = useLocation(); // 추가

  // 로그인 후 이동할 경로 결정
  const from = location.state?.from?.pathname || '/dashboard'; // 추가

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    try {
      const response = await api.post('/auth/login', formData);
      const { user, accessToken, refreshToken } = response.data;

      // 토큰 저장
      localStorage.setItem('refreshToken', refreshToken);
      
      // AuthContext에 로그인 정보 설정
      login(user, accessToken);

      // 원래 가려던 페이지 또는 대시보드로 이동
      navigate(from, { replace: true }); // 수정
    } catch (error) {
      setError(error.response?.data?.message || '로그인에 실패했습니다.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="auth-container">
      <div className="auth-card">
        <h2 className="auth-title">로그인</h2>
        <p className="auth-subtitle">계정에 로그인하세요</p>
        
        {error && <div className="error-message">{error}</div>}
        
        <form onSubmit={handleSubmit} className="auth-form">
          <div className="input-group">
            <label htmlFor="email">이메일</label>
            <input
              type="email"
              id="email"
              name="email"
              value={formData.email}
              onChange={handleChange}
              required
            />
          </div>

          <div className="input-group">
            <label htmlFor="password">비밀번호</label>
            <input
              type="password"
              id="password"
              name="password"
              value={formData.password}
              onChange={handleChange}
              required
            />
          </div>

          <button
            type="submit"
            className="auth-button"
            disabled={loading}
          >
            {loading ? '로그인 중...' : '로그인'}
          </button>
        </form>

        <div className="auth-switch">
          계정이 없으신가요?{' '}
          <Link to="/register" className="switch-button">
            회원가입
          </Link>
        </div>
      </div>
    </div>
  );
};

export default LoginForm;
```

**📁 frontend/src/components/Auth/RegisterForm.jsx** - 업데이트

```javascript
import React, { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom'; // 추가
import api from '../../api/api';
import './Auth.css';

const RegisterForm = () => { // props 제거
  const [formData, setFormData] = useState({
    username: '',
    email: '',
    password: '',
    confirmPassword: ''
  });
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const [success, setSuccess] = useState(false);

  const navigate = useNavigate(); // 추가

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    if (formData.password !== formData.confirmPassword) {
      setError('비밀번호가 일치하지 않습니다.');
      setLoading(false);
      return;
    }

    try {
      await api.post('/auth/register', {
        username: formData.username,
        email: formData.email,
        password: formData.password
      });

      setSuccess(true);
      setTimeout(() => {
        navigate('/login'); // 수정
      }, 2000);
    } catch (error) {
      setError(error.response?.data?.message || '회원가입에 실패했습니다.');
    } finally {
      setLoading(false);
    }
  };

  if (success) {
    return (
      <div className="auth-container">
        <div className="auth-card">
          <div className="success-message">
            회원가입이 완료되었습니다! 로그인 페이지로 이동합니다...
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="auth-container">
      <div className="auth-card">
        <h2 className="auth-title">회원가입</h2>
        <p className="auth-subtitle">새 계정을 만드세요</p>
        
        {error && <div className="error-message">{error}</div>}
        
        <form onSubmit={handleSubmit} className="auth-form">
          <div className="input-group">
            <label htmlFor="username">사용자명</label>
            <input
              type="text"
              id="username"
              name="username"
              value={formData.username}
              onChange={handleChange}
              required
            />
          </div>

          <div className="input-group">
            <label htmlFor="email">이메일</label>
            <input
              type="email"
              id="email"
              name="email"
              value={formData.email}
              onChange={handleChange}
              required
            />
          </div>

          <div className="input-group">
            <label htmlFor="password">비밀번호</label>
            <input
              type="password"
              id="password"
              name="password"
              value={formData.password}
              onChange={handleChange}
              required
            />
          </div>

          <div className="input-group">
            <label htmlFor="confirmPassword">비밀번호 확인</label>
            <input
              type="password"
              id="confirmPassword"
              name="confirmPassword"
              value={formData.confirmPassword}
              onChange={handleChange}
              required
            />
          </div>

          <button
            type="submit"
            className="auth-button"
            disabled={loading}
          >
            {loading ? '가입 중...' : '회원가입'}
          </button>
        </form>

        <div className="auth-switch">
          이미 계정이 있으신가요?{' '}
          <Link to="/login" className="switch-button">
            로그인
          </Link>
        </div>
      </div>
    </div>
  );
};

export default RegisterForm;
```

## 🧪 테스트 및 동작 확인

### 1. 자동 토큰 갱신 테스트

1. 로그인 후 대시보드에 접근
2. 브라우저 개발자 도구에서 localStorage의 token을 삭제
3. 새로고침 없이 API 요청 수행 (예: 프로필 정보 조회)
4. 자동으로 토큰이 갱신되고 요청이 성공하는지 확인

### 2. React Router 테스트

1. URL 직접 접근: `http://localhost:3000/dashboard`
2. 브라우저 뒤로가기/앞으로가기 버튼 동작 확인
3. 인증되지 않은 상태에서 보호된 라우트 접근 시 로그인 페이지로 리다이렉트 확인
4. 로그인 후 원래 가려던 페이지로 이동하는지 확인

## 📊 구현 결과

### Before (Part 4까지)
- ❌ 토큰 만료 시 수동 로그아웃 필요
- ❌ 조건부 렌더링으로 페이지 전환
- ❌ URL 변경 없음
- ❌ 브라우저 네비게이션 미지원

### After (Part 5-1)
- ✅ 자동 토큰 갱신으로 끊김 없는 사용자 경험
- ✅ 진정한 SPA 라우팅 시스템
- ✅ URL 기반 네비게이션
- ✅ 브라우저 뒤로가기/앞으로가기 지원
- ✅ 보호된 라우트 시스템
- ✅ 로그인 후 원래 페이지로 자동 이동

## 🔄 에러 핸들링 플로우

```
API 요청 → 401 에러 발생 → 자동 토큰 갱신 시도
    ↓
갱신 성공 → 원래 요청 재시도 → 성공
    ↓
갱신 실패 → 자동 로그아웃 → 로그인 페이지 리다이렉트
```

## 🎯 다음 편 예고

Part 5-2에서는 **다크모드 UI & 반응형 디자인**을 구현하여 모던하고 세련된 사용자 인터페이스를 만들어보겠습니다:

- 🌙 다크모드 CSS 테마 시스템
- 📱 완전한 반응형 디자인
- ✨ 고급 애니메이션 효과
- 🎨 그라디언트 배경과 현대적인 UI

---

> **💡 Pro Tip**: 이 시리즈의 전체 코드는 [GitHub Repository](https://github.com/your-username/SimpleAuthSystem)에서 확인할 수 있으며, 각 Part별로 브랜치가 분리되어 있어 단계별 학습이 가능합니다.

**🔗 브랜치**: `feature/part5-1-token-refresh-router`
