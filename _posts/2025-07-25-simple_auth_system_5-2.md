---
title: "Node.js JWT 인증 시스템 완성하기 Part 5-2: 다크모드 UI & 반응형 디자인"
categories:
    - MiniProject
    - SimpleAuthSystem
tag: [MiniProject, auth, 회원가입, 로그인, 프론트엔드, 다크모드, ui]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 5-2**입니다.

-   **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple_auth_system_1/)**
-   **[Part 2: 고급 JWT 인증 기능](https://hoondongseo.github.io/posts/simple_auth_system_2/)**
-   **[Part 3: React 프론트엔드 구현](https://hoondongseo.github.io/posts/simple_auth_system_3/)**
-   **[Part 4: 대시보드 및 로그인 상태 관리](https://hoondongseo.github.io/posts/simple_auth_system_4/)**
-   **[Part 5-1: 자동 토큰 갱신 & React Router](https://hoondongseo.github.io/posts/simple_auth_system_5-1/)**
-   **Part 5-2: 다크모드 UI & 반응형 디자인** ← 현재 글

## 🎯 현재 상황 분석

Part 5-1에서 기능적인 업그레이드를 완료한 후, 이번에는 **시각적 사용자 경험**을 향상시켰습니다:

1. **🌙 다크모드 UI**: Auth.css와 Dashboard.css에 다크모드 적용
2. **📱 반응형 디자인**: 모든 기기에서 최적화된 사용자 경험
3. **✨ 고급 애니메이션**: 부드럽고 자연스러운 인터랙션
4. **🎨 현대적인 디자인**: 그라디언트 배경과 Glass morphism 효과

## 🎨 현재 CSS 파일 구조

### App.css - 기본 전역 스타일

### App.css - 다크모드 전역 스타일

현재 App.css를 다크모드 CSS 변수로 설정해줍니다.

**📁 frontend/src/App.css**

```css
/* 전역 스타일 및 CSS 변수 - 다크모드 */
:root {
	--primary-color: #3b82f6;
	--primary-hover: #2563eb;
	--success-color: #10b981;
	--error-color: #ef4444;
	--background: #0f172a;
	--surface: #1e293b;
	--text-primary: #f1f5f9;
	--text-secondary: #94a3b8;
	--border: #334155;
	--shadow: 0 1px 3px 0 rgb(0 0 0 / 0.3);
	--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.4);
}

/* 기본 레이아웃 - 모바일 퍼스트 */
* {
	margin: 0;
	padding: 0;
	box-sizing: border-box;
}

body {
	font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto",
		sans-serif;
	background-color: var(--background);
	color: var(--text-primary);
	line-height: 1.6;
}

.App {
	min-height: 100vh;
	display: flex;
	flex-direction: column;
}
```

## 🌙 Auth.css 다크모드 구현

### 완전한 다크모드 인증 폼 (파일 위치: `frontend/src/components/Auth/Auth.css`)

아래의 모든 CSS 코드를 `Auth.css` 파일에 추가해주세요:

**📁 frontend/src/components/Auth/Auth.css**

```css
/* 다크모드 인증 폼 스타일 */
.auth-container {
	display: flex;
	justify-content: center;
	align-items: center;
	min-height: 100vh;
	background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
	padding: 1rem;
	position: relative;
}

/* 배경 장식 */
.auth-container::before {
	content: "";
	position: absolute;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background: radial-gradient(
			circle at 20% 80%,
			rgba(59, 130, 246, 0.15) 0%,
			transparent 50%
		), radial-gradient(circle at 80% 20%, rgba(139, 92, 246, 0.1) 0%, transparent
				50%);
	pointer-events: none;
}

.auth-card {
	background: linear-gradient(145deg, #1e1e2e 0%, #2a2a3a 100%);
	padding: 2.5rem;
	border-radius: 20px;
	box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4), 0 4px 8px rgba(0, 0, 0, 0.2),
		inset 0 1px 0 rgba(255, 255, 255, 0.1);
	width: 100%;
	max-width: 420px;
	margin: 1rem;
	position: relative;
	backdrop-filter: blur(10px);
	border: 1px solid rgba(255, 255, 255, 0.1);
	animation: slideUp 0.5s ease-out;
}

@keyframes slideUp {
	from {
		opacity: 0;
		transform: translateY(30px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

.auth-title {
	text-align: center;
	margin-bottom: 0.5rem;
	color: #f8fafc;
	font-size: 2.2rem;
	font-weight: 700;
	background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%);
	-webkit-background-clip: text;
	-webkit-text-fill-color: transparent;
	background-clip: text;
}

.auth-subtitle {
	text-align: center;
	margin-bottom: 2.5rem;
	color: #94a3b8;
	font-size: 0.95rem;
	line-height: 1.5;
}

.auth-form {
	display: flex;
	flex-direction: column;
	gap: 1.8rem;
}

.input-group {
	display: flex;
	flex-direction: column;
	gap: 0.6rem;
	position: relative;
}

.input-group label {
	font-weight: 600;
	color: #e2e8f0;
	font-size: 0.9rem;
	letter-spacing: 0.025em;
}

.input-group input {
	padding: 1rem;
	border: 2px solid #374151;
	border-radius: 12px;
	font-size: 1rem;
	transition: all 0.3s ease;
	background: #1f2937;
	color: #f9fafb;
	font-family: inherit;
}

.input-group input:focus {
	outline: none;
	border-color: #3b82f6;
	box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.2), 0 2px 4px rgba(59, 130, 246, 0.1);
	background: #111827;
	transform: translateY(-1px);
}

.input-group input:valid {
	border-color: #10b981;
}

.auth-button {
	width: 100%;
	padding: 1rem;
	background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%);
	color: white;
	border: none;
	border-radius: 12px;
	font-size: 1.05rem;
	font-weight: 600;
	cursor: pointer;
	transition: all 0.3s ease;
	margin-top: 0.5rem;
	position: relative;
	overflow: hidden;
}

.auth-button::before {
	content: "";
	position: absolute;
	top: 0;
	left: -100%;
	width: 100%;
	height: 100%;
	background: linear-gradient(
		90deg,
		transparent,
		rgba(255, 255, 255, 0.2),
		transparent
	);
	transition: left 0.5s;
}

.auth-button:hover::before {
	left: 100%;
}

.auth-button:hover:not(:disabled) {
	transform: translateY(-2px);
	box-shadow: 0 10px 25px rgba(102, 126, 234, 0.3), 0 4px 8px rgba(102, 126, 234, 0.2);
}

.auth-button:active {
	transform: translateY(0);
}

.auth-button:disabled {
	opacity: 0.6;
	cursor: not-allowed;
	transform: none;
	background: #94a3b8;
}

.auth-switch {
	text-align: center;
	margin-top: 2rem;
	color: #64748b;
	font-size: 0.9rem;
	line-height: 1.6;
}

.switch-button {
	background: none;
	border: none;
	color: #667eea;
	cursor: pointer;
	text-decoration: none;
	font-weight: 600;
	transition: color 0.2s ease;
	border-bottom: 1px solid transparent;
}

.switch-button:hover {
	color: #764ba2;
	border-bottom-color: #764ba2;
}

.error-message {
	background: linear-gradient(135deg, #fef2f2 0%, #fee2e2 100%);
	color: #dc2626;
	padding: 1rem;
	border-radius: 12px;
	border: 1px solid #fecaca;
	margin-bottom: 1rem;
	font-size: 0.9rem;
	font-weight: 500;
	display: flex;
	align-items: center;
	gap: 0.5rem;
	animation: shake 0.5s ease-out;
}

.error-message::before {
	content: "⚠️";
	font-size: 1.1rem;
}

@keyframes shake {
	0%, 100% {
		transform: translateX(0);
	}
	25% {
		transform: translateX(-5px);
	}
	75% {
		transform: translateX(5px);
	}
}

.success-message {
	background: linear-gradient(135deg, #f0fdf4 0%, #dcfce7 100%);
	color: #16a34a;
	padding: 1rem;
	border-radius: 12px;
	border: 1px solid #bbf7d0;
	margin-bottom: 1rem;
	font-size: 0.9rem;
	font-weight: 500;
	display: flex;
	align-items: center;
	gap: 0.5rem;
	animation: slideDown 0.5s ease-out;
}

.success-message::before {
	content: "✅";
	font-size: 1.1rem;
}

@keyframes slideDown {
	from {
		opacity: 0;
		transform: translateY(-10px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}
```

### Auth.css 반응형 디자인 (계속)

```css
/* 반응형 디자인 - Auth.css에 추가 */
@media (max-width: 480px) {
	.auth-card {
		padding: 2rem;
		margin: 0.5rem;
		border-radius: 16px;
	}

	.auth-title {
		font-size: 1.8rem;
	}

	.input-group input {
		padding: 0.9rem;
	}

	.auth-button {
		padding: 0.9rem;
	}
}

@media (min-width: 768px) {
	.auth-card {
		padding: 3rem;
		max-width: 450px;
	}

	.auth-title {
		font-size: 2.4rem;
	}
}
```

## 🏠 Dashboard.css 고급 다크모드 구현

### 완전한 다크모드 대시보드 (파일 위치: `frontend/src/components/Dashboard/Dashboard.css`)

아래의 모든 CSS 코드를 `Dashboard.css` 파일에 추가해주세요:

**📁 frontend/src/components/Dashboard/Dashboard.css**

```css
/* 다크모드 대시보드 스타일 */
.dashboard-container {
	display: flex;
	justify-content: center;
	align-items: center;
	min-height: 100vh;
	background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
	padding: 1rem;
	position: relative;
}

/* 배경 장식 */
.dashboard-container::before {
	content: "";
	position: absolute;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background: radial-gradient(
			circle at 20% 80%,
			rgba(59, 130, 246, 0.15) 0%,
			transparent 50%
		), radial-gradient(circle at 80% 20%, rgba(139, 92, 246, 0.1) 0%, transparent
				50%);
	pointer-events: none;
}

.dashboard-card {
	background: linear-gradient(145deg, #1e1e2e 0%, #2a2a3a 100%);
	padding: 3rem;
	border-radius: 24px;
	box-shadow: 0 25px 50px rgba(0, 0, 0, 0.4), 0 8px 16px rgba(0, 0, 0, 0.2),
		inset 0 1px 0 rgba(255, 255, 255, 0.1);
	width: 100%;
	max-width: 550px;
	text-align: center;
	position: relative;
	border: 1px solid rgba(255, 255, 255, 0.1);
	animation: scaleIn 0.6s ease-out;
}

@keyframes scaleIn {
	from {
		opacity: 0;
		transform: scale(0.9) translateY(20px);
	}
	to {
		opacity: 1;
		transform: scale(1) translateY(0);
	}
}

.dashboard-title {
	color: #f8fafc;
	font-size: 2.8rem;
	font-weight: 800;
	margin-bottom: 0.8rem;
	background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%);
	-webkit-background-clip: text;
	-webkit-text-fill-color: transparent;
	background-clip: text;
	animation: titleSlide 0.8s ease-out 0.2s both;
}

@keyframes titleSlide {
	from {
		opacity: 0;
		transform: translateY(-20px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

.dashboard-subtitle {
	color: #94a3b8;
	font-size: 1.1rem;
	line-height: 1.6;
	animation: subtitleSlide 0.8s ease-out 0.4s both;
}

@keyframes subtitleSlide {
	from {
		opacity: 0;
		transform: translateY(10px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

.user-info {
	background: linear-gradient(135deg, #374151 0%, #4b5563 100%);
	padding: 2.5rem;
	border-radius: 20px;
	margin-bottom: 2.5rem;
	display: flex;
	align-items: center;
	gap: 2rem;
	box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2), inset 0 1px 0 rgba(255, 255, 255, 0.1);
	border: 1px solid rgba(255, 255, 255, 0.1);
	animation: cardSlide 0.8s ease-out 0.6s both;
}

@keyframes cardSlide {
	from {
		opacity: 0;
		transform: translateX(-20px);
	}
	to {
		opacity: 1;
		transform: translateX(0);
	}
}

.user-avatar {
	display: flex;
	align-items: center;
	gap: 2rem;
	min-width: 0;
	flex: 1;
}

.user-details {
	flex: 1;
	text-align: left;
	min-width: 0;
}

.avatar-icon {
	display: inline-flex;
	align-items: center;
	justify-content: center;
	width: 90px;
	height: 90px;
	background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%);
	border-radius: 50%;
	font-size: 2.8rem;
	color: white;
	box-shadow: 0 8px 16px rgba(59, 130, 246, 0.4), 0 2px 4px rgba(59, 130, 246, 0.3);
	transition: transform 0.3s ease;
	position: relative;
	overflow: hidden;
}

.avatar-icon::before {
	content: "";
	position: absolute;
	top: -50%;
	left: -50%;
	width: 200%;
	height: 200%;
	background: linear-gradient(
		45deg,
		transparent,
		rgba(255, 255, 255, 0.3),
		transparent
	);
	transform: rotate(45deg);
	transition: all 0.6s;
	opacity: 0;
}

.avatar-icon:hover {
	transform: scale(1.05);
}

.avatar-icon:hover::before {
	opacity: 1;
	transform: rotate(45deg) translateX(100%);
}

.user-name {
	color: #f1f5f9;
	font-size: 2rem;
	font-weight: 700;
	margin-bottom: 0.6rem;
	letter-spacing: -0.025em;
}

.user-email {
	color: #cbd5e1;
	font-size: 1.15rem;
	margin-bottom: 0.6rem;
	font-weight: 500;
}

.user-joined {
	color: #94a3b8;
	font-size: 1rem;
	font-weight: 400;
	display: flex;
	align-items: center;
	gap: 0.5rem;
}

.user-joined::before {
	content: "📅";
	font-size: 1rem;
}

.dashboard-actions {
	display: flex;
	justify-content: center;
	gap: 1rem;
	margin-top: 1rem;
	animation: buttonSlide 0.8s ease-out 0.8s both;
}

@keyframes buttonSlide {
	from {
		opacity: 0;
		transform: translateY(20px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

.logout-button {
	background: linear-gradient(135deg, #ef4444 0%, #dc2626 100%);
	color: white;
	border: none;
	padding: 1rem 2.5rem;
	border-radius: 12px;
	font-size: 1.1rem;
	font-weight: 600;
	cursor: pointer;
	transition: all 0.3s ease;
	position: relative;
	overflow: hidden;
	box-shadow: 0 4px 8px rgba(239, 68, 68, 0.3);
}

.logout-button::before {
	content: "";
	position: absolute;
	top: 0;
	left: -100%;
	width: 100%;
	height: 100%;
	background: linear-gradient(
		90deg,
		transparent,
		rgba(255, 255, 255, 0.2),
		transparent
	);
	transition: left 0.5s;
}

.logout-button:hover::before {
	left: 100%;
}

.logout-button:hover {
	transform: translateY(-3px);
	box-shadow: 0 12px 24px rgba(239, 68, 68, 0.5), 0 4px 8px rgba(239, 68, 68, 0.4);
}

.logout-button:active {
	transform: translateY(-1px);
	box-shadow: 0 6px 12px rgba(239, 68, 68, 0.4);
}
```

### Dashboard.css 반응형 디자인 (계속)

```css
/* 반응형 디자인 - Dashboard.css에 추가 */
@media (max-width: 640px) {
	.dashboard-card {
		padding: 2rem;
		margin: 0.5rem;
		border-radius: 20px;
	}

	.dashboard-title {
		font-size: 2.2rem;
	}

	.user-info {
		flex-direction: column;
		text-align: center;
		padding: 2rem;
		gap: 1.5rem;
	}

	.user-details {
		text-align: center;
	}

	.user-name {
		font-size: 1.6rem;
	}

	.avatar-icon {
		width: 80px;
		height: 80px;
		font-size: 2.4rem;
	}
}

@media (max-width: 480px) {
	.dashboard-container {
		padding: 0.5rem;
	}

	.dashboard-card {
		padding: 1.5rem;
	}

	.dashboard-title {
		font-size: 1.8rem;
	}

	.user-info {
		padding: 1.5rem;
	}

	.logout-button {
		padding: 0.9rem 2rem;
		font-size: 1rem;
	}
}

@media (min-width: 768px) {
	.dashboard-card {
		padding: 3.5rem;
		max-width: 600px;
	}

	.dashboard-title {
		font-size: 3rem;
	}

	.user-info {
		padding: 3rem;
		gap: 2.5rem;
	}

	.avatar-icon {
		width: 100px;
		height: 100px;
		font-size: 3rem;
	}
}
```

## 📁 파일별 적용 가이드

### 1. App.css 파일 구성

```css
/* 이 파일에는 전역 스타일과 CSS 변수만 포함 */
/* 위에서 설명한 App.css 내용을 그대로 사용 */
```

### 2. Auth.css 파일 구성

```css
/* 1. 인증 컨테이너 스타일 (.auth-container, .auth-card) */
/* 2. 폼 구조 스타일 (.auth-form, .input-group) */
/* 3. 버튼 스타일 (.auth-button) */
/* 4. 에러/성공 메시지 스타일 (.error-message, .success-message) */
/* 5. 애니메이션 (@keyframes slideUp, shake, slideDown) */
/* 6. 반응형 미디어 쿼리 */
```

### 3. Dashboard.css 파일 구성

```css
/* 1. 대시보드 컨테이너 스타일 (.dashboard-container, .dashboard-card) */
/* 2. 사용자 정보 구조 (.user-info, .user-avatar, .user-details) */
/* 3. 아바타 스타일 (.avatar-icon) */
/* 4. 대시보드 액션 (.dashboard-actions, .logout-button) */
/* 5. 애니메이션 (@keyframes scaleIn, titleSlide, subtitleSlide, cardSlide, buttonSlide) */
/* 6. 반응형 미디어 쿼리 */
```

## 📊 구현 결과

### ✅ 완성된 기능들

1. **다크모드 UI**:

    - 세련된 그라디언트 배경
    - Glass morphism 카드 효과
    - 그라디언트 텍스트 타이틀

2. **고급 애니메이션**:

    - `slideUp` 카드 등장 애니메이션
    - `scaleIn` 대시보드 카드 애니메이션
    - `titleSlide`, `subtitleSlide` 순차적 타이틀 등장
    - `cardSlide` 사용자 정보 카드 애니메이션

3. **인터랙션 효과**:

    - 버튼 시머 효과 (`::before` pseudo-element)
    - 아바타 아이콘 호버 효과
    - 입력 필드 포커스 애니메이션

4. **완전한 반응형 지원**:
    - 모바일 (480px 이하)
    - 태블릿 (640px ~ 768px)
    - 데스크톱 (768px 이상)

### 🎨 디자인 특징

-   **배경**: 3단계 그라디언트 (`#1a1a2e` → `#16213e` → `#0f3460`)
-   **카드**: Glass morphism 효과와 inset 하이라이트
-   **타이틀**: 그라디언트 텍스트 클리핑
-   **버튼**: 호버 시 시머 효과와 그림자 강화
-   **아바타**: 45도 시머 애니메이션

## 🎯 다음 편 예고

Part 5-3에서는 **로딩 스피너 & UX 개선**을 구현하여 사용자 경험을 완성합니다:

-   ⏳ 로딩 스피너 애니메이션
-   🔄 API 요청 상태 표시
-   ✅ 성공/에러 토스트 메시지
-   🎭 스켈레톤 로딩 효과

---

> **💡 Pro Tip**: backdrop-filter 속성은 Safari에서 `-webkit-backdrop-filter`를 함께 사용해야 합니다.

**🔗 브랜치**: `feature/part5-2-darkmode-ui`
