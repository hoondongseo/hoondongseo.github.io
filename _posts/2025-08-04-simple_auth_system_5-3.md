---
title: "React JWT 인증 시스템 Part 5-3: 로딩 스피너 & Toast 알림으로 UX 완성하기"
categories:
- MiniProject
- SimpleAuthSystem
tag: [MiniProject, React, UX, LoadingSpinner, Toast, 애니메이션, 사용자경험, 인증시스템]
author_profile: false
sidebar:
    nav: "docs"
search: true

---

> Eclipse 스타일 로딩 스피너와 부드러운 Toast 알림으로 완성하는 모던 웹 UX

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 5-2**입니다.

-   **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple_auth_system_1/)**
-   **[Part 2: 고급 JWT 인증 기능](https://hoondongseo.github.io/posts/simple_auth_system_2/)**
-   **[Part 3: React 프론트엔드 구현](https://hoondongseo.github.io/posts/simple_auth_system_3/)**
-   **[Part 4: 대시보드 및 로그인 상태 관리](https://hoondongseo.github.io/posts/simple_auth_system_4/)**
-   **[Part 5-1: 자동 토큰 갱신 & React Router](https://hoondongseo.github.io/posts/simple_auth_system_5-1/)**
-   **[Part 5-2: 다크모드 UI & 반응형 디자인](https://hoondongseo.github.io/posts/simple_auth_system_5-2/)**
-   **Part 5-3: 로딩 스피너 & Toast 알림으로 UX 완성하기** ← 현재 글

## 🎯 프로젝트 개요

이번 포스트에서는 **React 인증 시스템**에 **로딩 스피너**와 **Toast 알림 시스템**을 구현해서 사용자 경험을 한층 더 향상시켜보겠습니다.

### 🛠️ 기술 스택

- **Frontend**: React.js, CSS3 Animation
- **UI Components**: LoadingSpinner, Toast
- **Animation**: Eclipse 스타일 회전, Fade-in/out
- **State Management**: useState, useContext
- **HTTP Client**: Axios Interceptors

### 📋 구현할 기능

- ✅ Eclipse 스타일 LoadingSpinner 컴포넌트
- ✅ Toast 알림 시스템 (Success/Error/Info/Warning)
- ✅ 부드러운 fade-in/fade-out 애니메이션  
- ✅ 로그인/회원가입 폼 UX 개선
- ✅ API 인터셉터 버그 수정

---

## 🚀 Step 1: LoadingSpinner 컴포넌트 구현

### 1-1. 컴포넌트 구조 설계

```bash
frontend/src/components/common/UI/
├── LoadingSpinner.js
└── LoadingSpinner.css
```

### 1-2. LoadingSpinner 컴포넌트 생성

`components/common/UI/LoadingSpinner.js` 파일을 생성합니다:

```jsx
import React from "react";
import "./LoadingSpinner.css";

const LoadingSpinner = ({ size = "medium", color = "primary" }) => {
	return (
		<div
			className={`loading-spinner loading-spinner--${size} loading-spinner--${color}`}
		>
			<div className="spinner-ring">
				<div></div>
			</div>
		</div>
	);
};

export default LoadingSpinner;
```

### 🔍 LoadingSpinner 설계 포인트

1. **Props 기반 설계**: `size`와 `color` props로 유연한 사용
   - `size`: small/medium 크기 옵션
   - `color`: primary/white 색상 옵션
2. **CSS 클래스 기반**: 확장 가능한 스타일링 구조
3. **재사용성**: 모든 컴포넌트에서 동일한 패턴으로 사용

### 1-3. Eclipse 스타일 CSS 애니메이션

`components/common/UI/LoadingSpinner.css` 파일을 생성합니다:

```css
.loading-spinner {
	display: inline-block;
	position: relative;
}

.spinner-ring {
	display: inline-block;
	position: relative;
	width: 40px;
	height: 40px;
}

.spinner-ring div {
	box-sizing: border-box;
	display: block;
	position: absolute;
	width: 32px;
	height: 32px;
	margin: 4px;
	border: 3px solid transparent;
	border-radius: 50%;
	animation: spin 1s linear infinite;
}

@keyframes spin {
	0% { transform: rotate(0deg); }
	100% { transform: rotate(360deg); }
}

/* 색상 변형 */
.loading-spinner--white .spinner-ring div {
	border-top: 3px solid #ffffff;
	border-right: 3px solid rgba(255, 255, 255, 0.3);
	border-bottom: 3px solid rgba(255, 255, 255, 0.3);
	border-left: 3px solid rgba(255, 255, 255, 0.3);
}

/* 크기 변형 */
.loading-spinner--small .spinner-ring {
	width: 20px;
	height: 20px;
}

.loading-spinner--small .spinner-ring div {
	width: 16px;
	height: 16px;
	margin: 2px;
	border-width: 2px;
}
```

### 🎨 Eclipse 애니메이션 특징

| 특징 | 설명 |
|-----|------|
| **Eclipse 효과** | 한 방향만 진한 색상으로 표시 |
| **부드러운 회전** | `linear` 타이밍으로 일정한 속도 |
| **Chrome 스타일** | 브라우저 로딩 스피너에서 영감 |
| **반응형 크기** | small/medium 옵션으로 다양한 용도 |

---

## 🔔 Step 2: Toast 알림 시스템 구축

### 2-1. Toast 컴포넌트 아키텍처

```bash
frontend/src/components/common/UI/
├── Toast.js
└── Toast.css
```

### 2-2. Toast 컴포넌트 구현

`components/common/UI/Toast.js` 파일을 생성합니다:

```jsx
import React, { useEffect, useState, useCallback } from "react";
import "./Toast.css";

const Toast = ({
	message,
	type = "info",
	isVisible,
	onClose,
	duration = 3000,
}) => {
	const [isHiding, setIsHiding] = useState(false);

	const handleClose = useCallback(() => {
		setIsHiding(true);
		setTimeout(() => {
			setIsHiding(false);
			onClose();
		}, 300); // CSS 트랜지션과 동일한 시간
	}, [onClose]);

	useEffect(() => {
		if (isVisible && duration > 0) {
			const timer = setTimeout(() => {
				handleClose();
			}, duration);

			return () => clearTimeout(timer);
		}
	}, [isVisible, duration, handleClose]);

	if (!isVisible && !isHiding) return null;

	return (
		<div
			className={`toast toast--${type} ${
				isVisible && !isHiding ? "toast--visible" : ""
			} ${isHiding ? "toast--hiding" : ""}`}
		>
			<div className="toast-content">
				<span className="toast-icon">
					{type === "success" && "✅"}
					{type === "error" && "❌"}
					{type === "info" && "ℹ️"}
					{type === "warning" && "⚠️"}
				</span>
				<span className="toast-message">{message}</span>
			</div>
			<button className="toast-close" onClick={handleClose}>
				×
			</button>
		</div>
	);
};

export default Toast;
```

### 🎯 Toast 핵심 기능

1. **자동 닫기**: `duration` 설정으로 자동 사라짐
2. **수동 닫기**: 우상단 X 버튼으로 즉시 닫기  
3. **부드러운 애니메이션**: fade-in/fade-out 효과
4. **타입별 스타일**: success/error/info/warning 구분
5. **상태 관리**: `isHiding` state로 애니메이션 제어

### 2-3. Toast CSS 스타일링

`components/common/UI/Toast.css` 파일을 생성합니다:

```css
.toast {
	position: fixed;
	top: 20px;
	right: 20px;
	z-index: 1000;
	min-width: 300px;
	max-width: 500px;
	padding: 1rem 1.5rem;
	border-radius: 12px;
	box-shadow: 0 10px 25px rgba(0, 0, 0, 0.2);
	transform: translateX(100%);
	opacity: 0;
	transition: all 0.3s ease-in-out;
	backdrop-filter: blur(10px);
}

.toast--visible {
	transform: translateX(0);
	opacity: 1;
}

.toast--hiding {
	transform: translateX(100%);
	opacity: 0;
}

.toast-content {
	display: flex;
	align-items: center;
	gap: 0.75rem;
	padding-right: 1.5rem;
}

.toast-icon {
	font-size: 1.2rem;
	flex-shrink: 0;
}

.toast-message {
	flex: 1;
	font-size: 0.95rem;
	font-weight: 500;
	line-height: 1.4;
}

.toast-close {
	position: absolute;
	top: 0.5rem;
	right: 0.5rem;
	background: none;
	border: none;
	font-size: 1.2rem;
	cursor: pointer;
	padding: 0.25rem;
	color: inherit;
	opacity: 0.7;
	transition: opacity 0.2s ease;
	border-radius: 4px;
	width: 24px;
	height: 24px;
	display: flex;
	align-items: center;
	justify-content: center;
}

.toast-close:hover {
	opacity: 1;
}

/* 타입별 그라데이션 배경 */
.toast--success {
	background: linear-gradient(135deg, #059669 0%, #10b981 100%);
	color: white;
}

.toast--error {
	background: linear-gradient(135deg, #dc2626 0%, #ef4444 100%);
	color: white;
}

.toast--info {
	background: linear-gradient(135deg, #2563eb 0%, #3b82f6 100%);
	color: white;
}

.toast--warning {
	background: linear-gradient(135deg, #d97706 0%, #f59e0b 100%);
	color: white;
}
```

### 🎨 Toast 디자인 특징

| 특징 | 설명 |
|-----|------|
| **Slide 애니메이션** | 오른쪽에서 슬라이드인 |
| **그라데이션 배경** | 각 타입별 시각적 구분 |
| **블러 효과** | `backdrop-filter`로 모던한 느낌 |
| **반응형 크기** | min/max-width로 다양한 메시지 길이 대응 |

---

## 🔐 Step 3: 인증 폼 UX 개선

### 3-1. LoginForm 개선

기존의 단순한 에러 메시지를 Toast 시스템으로 교체합니다:

```jsx
// components/Auth/LoginForm.jsx
import React, { useState, useContext } from "react";
import { useNavigate } from "react-router-dom";
import { AuthContext } from "../context/AuthContext";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import Toast from "../common/UI/Toast";
import "./Auth.css";

const LoginForm = () => {
	const [formData, setFormData] = useState({
		email: "",
		password: "",
	});
	const [loading, setLoading] = useState(false);
	const [toast, setToast] = useState({
		isVisible: false,
		message: "",
		type: "info",
	});

	const { login } = useContext(AuthContext);
	const navigate = useNavigate();

	const showToast = (message, type = "info") => {
		setToast({
			isVisible: true,
			message,
			type,
		});
	};

	const closeToast = () => {
		setToast((prev) => ({
			...prev,
			isVisible: false,
		}));
	};

	const handleSubmit = async (e) => {
		e.preventDefault();
		setLoading(true);
		closeToast();

		try {
			await login(formData.email, formData.password);
			// 성공 시에는 Toast 없이 즉시 대시보드로 이동
			navigate("/dashboard");
		} catch (err) {
			showToast(
				err.response?.data?.message || "로그인에 실패했습니다.",
				"error"
			);
		} finally {
			setLoading(false);
		}
	};

	return (
		<div className="auth-container">
			{/* ... 기존 폼 구조 ... */}
			
			<button
				type="submit"
				className="auth-button"
				disabled={loading}
			>
				{loading ? (
					<>
						<LoadingSpinner size="small" color="white" />
						<span style={{ marginLeft: "8px" }}>
							로그인 중...
						</span>
					</>
				) : (
					"로그인"
				)}
			</button>

			{/* Toast 메시지 */}
			<Toast
				message={toast.message}
				type={toast.type}
				isVisible={toast.isVisible}
				onClose={closeToast}
				duration={3000}
			/>
		</div>
	);
};

export default LoginForm;
```

### 🎯 LoginForm 개선 사항

| 기능 | Before | After |
|-----|--------|-------|
| **로딩 상태** | 버튼 텍스트만 변경 | ✅ LoadingSpinner + 텍스트 |
| **에러 표시** | 정적 div 메시지 | ✅ Toast 알림 |
| **성공 처리** | Toast 표시 후 이동 | ✅ 즉시 이동 (빠른 UX) |
| **Form 보존** | 에러 시 입력 사라짐 | ✅ 입력 내용 유지 |

### 3-2. RegisterForm 개선

회원가입 폼도 동일한 패턴으로 개선합니다:

```jsx
// components/Auth/RegisterForm.jsx 주요 부분
const handleSubmit = async (e) => {
	e.preventDefault();
	setLoading(true);

	try {
		await api.post("/auth/register", formData);
		showToast("회원가입이 완료되었습니다! 로그인해주세요.", "success");
		setTimeout(() => {
			navigate("/login");
		}, 2000);
	} catch (err) {
		showToast(
			err.response?.data?.message || "회원가입에 실패했습니다.",
			"error"
		);
	} finally {
		setLoading(false);
	}
};

// 버튼 JSX
<button
	type="submit"
	className="auth-button"
	disabled={loading}
>
	{loading ? (
		<>
			<LoadingSpinner size="small" color="white" />
			<span style={{ marginLeft: "8px" }}>
				가입 중...
			</span>
		</>
	) : (
		"회원가입"
	)}
</button>
```

### ✨ RegisterForm 특별한 점

- 🎉 **성공 Toast**: 회원가입 완료 알림 후 자동 이동
- ⏱️ **적절한 지연**: 2초 후 로그인 페이지로 이동  
- 🔄 **일관된 패턴**: LoginForm과 동일한 UX 패턴

---

## 🛠️ Step 4: API 인터셉터 버그 수정

### 4-1. 문제 상황

개발 중 심각한 버그를 발견했습니다:

**문제**: 로그인 실패 시 입력한 내용이 1초 후에 모두 사라지고 페이지가 새로고침됨

### 4-2. 원인 분석

문제의 원인은 `api.js`의 response interceptor에 있었습니다:

```javascript
// 문제가 있던 코드
api.interceptors.response.use(
	(response) => response,
	async (error) => {
		if (error.response?.status === 401 && !originalRequest._retry) {
			// 모든 401 에러에 대해 강제 리다이렉트
			window.location.href = "/";  // 문제 지점
		}
		return Promise.reject(error);
	}
);
```

### 🔍 분석 결과

| 단계 | 발생 상황 |
|-----|----------|
| 1. 로그인 실패 | 백엔드에서 401 상태코드 반환 |
| 2. Interceptor 감지 | 401을 감지하고 강제 리다이렉트 실행 |
| 3. 페이지 새로고침 | `window.location.href = "/"` 실행 |
| 4. Form 초기화 | 페이지가 새로고침되면서 입력 내용 사라짐 |

### 4-3. 해결 방법

로그인/회원가입 요청은 401이 정상적인 응답이므로 interceptor를 건너뛰도록 수정:

```javascript
// ✅ 수정된 코드
api.interceptors.response.use(
	(response) => response,
	async (error) => {
		if (error.response?.status === 401 && !originalRequest._retry) {
			// 로그인/회원가입 요청은 401이 정상적인 응답이므로 interceptor 건너뛰기
			if (originalRequest.url?.includes('/auth/login') || 
				originalRequest.url?.includes('/auth/register')) {
				return Promise.reject(error);
			}

			originalRequest._retry = true;
			// 나머지 토큰 갱신 로직...
		}
		return Promise.reject(error);
	}
);
```

### 🎯 핵심 개선

| 구분 | 설명 |
|-----|------|
| **정확한 구분** | 인증 요청과 일반 API 요청 분리 |
| **보안 유지** | 일반 API의 토큰 갱신 로직은 그대로 유지 |
| **UX 향상** | 로그인 실패 시 form 데이터 보존 |

---

## 🎯 Step 5: Dashboard UX 완성

### 5-1. Dashboard 로그아웃 기능 개선

마지막으로 Dashboard의 로그아웃 기능도 동일한 패턴으로 개선합니다:

```jsx
// components/Dashboard/Dashboard.jsx
import React, { useContext, useState } from "react";
import { AuthContext } from "../context/AuthContext";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import Toast from "../common/UI/Toast";
import "./Dashboard.css";

const Dashboard = () => {
	const { user, logout } = useContext(AuthContext);
	const [loading, setLoading] = useState(false);
	const [toast, setToast] = useState({
		isVisible: false,
		message: "",
		type: "info",
	});

	const showToast = (message, type = "info") => {
		setToast({
			isVisible: true,
			message,
			type,
		});
	};

	const closeToast = () => {
		setToast((prev) => ({
			...prev,
			isVisible: false,
		}));
	};

	const handleLogout = async () => {
		setLoading(true);
		closeToast();

		try {
			await logout();
			showToast("로그아웃되었습니다.", "success");
		} catch (error) {
			console.error("로그아웃 에러:", error);
			showToast("로그아웃 중 오류가 발생했습니다.", "error");
		} finally {
			setLoading(false);
		}
	};

	return (
		<div className="dashboard-container">
			{/* ... 기존 사용자 정보 표시 ... */}
			
			<div className="dashboard-actions">
				<button 
					onClick={handleLogout} 
					className="logout-button"
					disabled={loading}
				>
					{loading ? (
						<>
							<LoadingSpinner size="small" color="white" />
							<span style={{ marginLeft: "8px" }}>
								로그아웃 중...
							</span>
						</>
					) : (
						"로그아웃"
					)}
				</button>
			</div>

			{/* Toast 메시지 */}
			<Toast
				message={toast.message}
				type={toast.type}
				isVisible={toast.isVisible}
				onClose={closeToast}
				duration={2000}
			/>
		</div>
	);
};

export default Dashboard;
```

### 5-2. 전체 앱 일관성 확보

모든 페이지에 동일한 패턴 적용 완료:

| 컴포넌트 | LoadingSpinner | Toast | 적용 기능 |
|----------|----------------|-------|-----------|
| **LoginForm** | ✅ | ✅ | 로그인 시도 |
| **RegisterForm** | ✅ | ✅ | 회원가입 시도 |
| **Dashboard** | ✅ | ✅ | 로그아웃 시도 |

### 🎨 통일된 UX 패턴

- 🔄 **모든 비동기 작업에 LoadingSpinner 표시**
- 🔔 **모든 사용자 피드백을 Toast로 통일**
- ⏱️ **적절한 지연시간과 자동 닫기 설정**
- 🎨 **일관된 색상과 애니메이션**

---

## 🧪 완전한 UX 테스트 시나리오

### ✅ 성공 케이스 테스트

| 페이지 | 시나리오 | LoadingSpinner | Toast | 결과 |
|-------|----------|----------------|-------|------|
| **LoginForm** | 올바른 로그인 | ✅ 표시 | ❌ 없음 | 즉시 Dashboard 이동 |
| **RegisterForm** | 성공적 회원가입 | ✅ 표시 | ✅ 성공 메시지 | 2초 후 로그인 페이지 |
| **Dashboard** | 로그아웃 성공 | ✅ 표시 | ✅ 성공 메시지 | 홈으로 리다이렉트 |

### ❌ 에러 케이스 테스트

| 페이지 | 시나리오 | LoadingSpinner | Toast | Form 데이터 |
|-------|----------|----------------|-------|-------------|
| **LoginForm** | 잘못된 비밀번호 | ✅ → ❌ | ✅ 에러 메시지 | ✅ 유지됨 |
| **RegisterForm** | 중복 이메일 | ✅ → ❌ | ✅ 에러 메시지 | ✅ 유지됨 |
| **Dashboard** | 네트워크 오류 | ✅ → ❌ | ✅ 에러 메시지 | N/A |

---

## 🔒 성능 및 접근성 고려사항

### 1. 성능 최적화

- ✅ **useCallback 사용**: Toast의 handleClose 함수 최적화
- ✅ **적절한 duration**: 3초(일반), 2초(빠른 작업)
- ✅ **메모리 누수 방지**: useEffect cleanup 함수 사용

### 2. 접근성 (a11y)

- ✅ **적절한 색상 대비**: 그라데이션 배경에 흰색 텍스트
- ✅ **키보드 접근**: 닫기 버튼 포커스 가능
- ✅ **의미있는 아이콘**: 이모지로 시각적 구분

### 3. 사용자 경험

- ✅ **즉각적 피드백**: 버튼 클릭 시 바로 로딩 표시
- ✅ **일관된 패턴**: 모든 페이지에서 동일한 동작
- ✅ **오류 복구**: 실패 시 다시 시도 가능

---

## 📊 완성된 기능 요약

### ✨ 구현 완료된 기능

- [x] **LoadingSpinner 컴포넌트**
  - Eclipse 스타일 회전 애니메이션
  - size/color props로 유연한 사용
  - Chrome 브라우저에서 영감받은 디자인

- [x] **Toast 알림 시스템**
  - 4가지 타입 지원 (success/error/info/warning)
  - 자동/수동 닫기 기능
  - 부드러운 slide + fade 애니메이션

- [x] **인증 폼 UX 개선**
  - 로그인/회원가입에 스피너 적용
  - 기존 div 메시지를 Toast로 교체
  - 에러 시 form 데이터 보존

- [x] **API 인터셉터 버그 수정**
  - 401 에러 처리 로직 개선
  - 인증 요청과 일반 API 구분
  - 페이지 새로고침 방지

- [x] **전체 앱 UX 통일**
  - Dashboard 로그아웃 기능 개선
  - 일관된 로딩/알림 패턴
  - 통일된 색상과 애니메이션

### 🎯 최종 결과물

| 기능 | 구현 상태 | 사용 위치 |
|-----|----------|-----------|
| **LoadingSpinner** | ✅ 완료 | 로그인, 회원가입, 로그아웃 버튼 |
| **Toast System** | ✅ 완료 | 모든 사용자 피드백 |
| **Fade Animations** | ✅ 완료 | Toast 등장/사라짐 |
| **Error Handling** | ✅ 완료 | API 실패 시 적절한 메시지 |
| **Form Persistence** | ✅ 완료 | 에러 시 입력 내용 유지 |

---

## 🚀 다음 단계 미리보기

### Part 6에서 다룰 내용:

- 👤 **프로필 관리 시스템** 구현
- 📧 **이메일 인증** 기능 추가  
- 🔒 **비밀번호 변경** API
- 📸 **프로필 이미지** 업로드
- 🔐 **2단계 인증** (2FA) 구현

---

## 💡 핵심 포인트 정리

### 🎓 배운 것들

1. **컴포넌트 기반 설계**: 재사용 가능한 UI 컴포넌트 구축
2. **CSS 애니메이션**: keyframes와 transition을 활용한 부드러운 효과
3. **사용자 경험**: 로딩 상태와 피드백의 중요성
4. **상태 관리**: useState와 useEffect를 활용한 복잡한 상태 제어
5. **API 인터셉터**: 전역 에러 처리의 주의사항
6. **일관성**: 전체 앱에서 통일된 UX 패턴

### ⚠️ 주의사항

- **애니메이션 타이밍**: CSS와 JavaScript 타이밍 동기화 필수
- **메모리 누수**: useEffect cleanup 함수로 타이머 정리
- **접근성**: 색상 대비와 키보드 접근성 고려
- **성능**: 불필요한 리렌더링 방지를 위한 useCallback 사용

---

## 📚 참고 자료

- [React Hooks 공식 문서](https://reactjs.org/docs/hooks-intro.html)
- [CSS Animations MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Animations)
- [Web Content Accessibility Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Material Design - Loading](https://material.io/design/communication/loading.html)
- [Toast Notifications Best Practices](https://uxdesign.cc/toast-notification-or-dialog-box-ae32ad53106d)

---

## 🎉 마무리

이번 포스트에서는 **React 인증 시스템**에 **LoadingSpinner**와 **Toast 알림 시스템**을 구현해서 완전한 사용자 경험을 만들어보았습니다.

**Eclipse 스타일의 회전 애니메이션**부터 **부드러운 fade-out 효과**까지, 실제 운영 환경에서 사용할 수 있는 수준의 UX 컴포넌트를 구축했습니다.

특히 **API 인터셉터 버그 수정**을 통해 실제 개발에서 마주할 수 있는 예상치 못한 문제들을 해결하는 경험도 함께 했습니다.

다음 편에서는 **프로필 관리 시스템**과 **이메일 인증** 기능을 구현해서 더욱 완성도 높은 인증 시스템을 만들어보겠습니다!

**궁금한 점이나 개선사항이 있다면 댓글로 남겨주세요!** 🙋‍♂️

---

### 🔗 소스코드

전체 소스코드는 [GitHub 저장소](https://github.com/hoondongseo/SimpleAuthSystem)에서 확인하실 수 있습니다.

```bash
git clone https://github.com/hoondongseo/SimpleAuthSystem.git
cd SimpleAuthSystem/frontend
npm install
npm start
```

**🌟 이 글이 도움이 되셨다면 GitHub Star를 눌러주시고, 궁금한 점은 언제든 댓글로 남겨주세요!**
