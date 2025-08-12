---
title: "React JWT 인증 시스템 Part 6-3: 비밀번호 변경 시스템 구현 - 실시간 강도 검증과 보안 강화"
categories:
    - MiniProject
    - SimpleAuthSystem
tag:
    [
        MiniProject,
        React,
        비밀번호변경,
        보안,
        JWT,
        인증시스템,
        PasswordChange,
        UI/UX,
        보안강화,
        실시간검증,
    ]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

> 실시간 비밀번호 강도 검증과 5단계 보안 요구사항으로 완성하는 안전한 비밀번호 변경 시스템

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 6-3**입니다.

-   **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple_auth_system_1/)**
-   **[Part 2: 고급 JWT 인증 기능](https://hoondongseo.github.io/posts/simple_auth_system_2/)**
-   **[Part 3: React 프론트엔드 구현](https://hoondongseo.github.io/posts/simple_auth_system_3/)**
-   **[Part 4: 대시보드 및 로그인 상태 관리](https://hoondongseo.github.io/posts/simple_auth_system_4/)**
-   **[Part 5-1: 자동 토큰 갱신 & React Router](https://hoondongseo.github.io/posts/simple_auth_system_5-1/)**
-   **[Part 5-2: 다크모드 UI & 반응형 디자인](https://hoondongseo.github.io/posts/simple_auth_system_5-2/)**
-   **[Part 5-3: 로딩 스피너 & Toast 알림으로 UX 완성하기](https://hoondongseo.github.io/posts/simple_auth_system_5-3/)**
-   **[Part 6-1 & 6-2: 프로필 관리 시스템 & 이메일 인증으로 완성하는 사용자 관리](https://hoondongseo.github.io/posts/simple_auth_system_6-1,6-2/)**
-   **Part 6-3: 비밀번호 변경 시스템 구현 - 실시간 강도 검증과 보안 강화** ← 현재 글

## 🎯 프로젝트 개요

이번 포스트에서는 **React JWT 인증 시스템**에 **안전한 비밀번호 변경 시스템**을 구현해보겠습니다. Part 6-2의 이메일 인증 시스템에 이어서, 사용자가 안전하게 비밀번호를 변경할 수 있는 완전한 시스템을 구축합니다. **실시간 비밀번호 강도 검증**부터 **5단계 보안 요구사항**, **부드러운 애니메이션**까지 프로덕션 레벨의 보안 시스템을 만들어보겠습니다.

### 🛠️ 기술 스택

-   **Backend**: Node.js, Express.js, MongoDB, bcryptjs
-   **Frontend**: React.js, CSS3 Animation
-   **Security**: JWT Tokens, Password Strength Validation
-   **UI/UX**: Dark Theme, Real-time Feedback, Smooth Animations
-   **State Management**: useState, useContext

### 📋 구현할 기능

-   ✅ 안전한 비밀번호 변경 API (백엔드)
-   ✅ 데이터베이스 스키마 업데이트 (username → name)
-   ✅ 실시간 비밀번호 강도 검증 (프론트엔드)
-   ✅ 5단계 보안 요구사항 (8자 이상, 대소문자, 숫자, 특수문자)
-   ✅ PasswordChange 컴포넌트 구현
-   ✅ Profile 시스템 통합 및 UI/UX 개선
-   ✅ 다크테마 스타일링 및 애니메이션 효과

---

## 🚀 Step 1: 백엔드 비밀번호 변경 API 구현

### 1-1. 비밀번호 변경 엔드포인트 개발

`backend/routes/auth.js`에 새로운 비밀번호 변경 API를 추가했습니다:

```javascript
// 비밀번호 변경 API
router.put("/change-password", authenticateToken, async (req, res) => {
	try {
		console.log("비밀번호 변경 요청 데이터:", req.body);
		const { currentPassword, newPassword, confirmPassword } = req.body;

		// 1. 입력값 검증
		if (!currentPassword || !newPassword || !confirmPassword) {
			console.log("입력값 검증 실패");
			return res.status(400).json({
				success: false,
				message: "모든 필드를 입력해주세요.",
			});
		}

		// 2. 새 비밀번호와 확인 비밀번호 일치 검사
		if (newPassword !== confirmPassword) {
			console.log("비밀번호 확인 불일치");
			return res.status(400).json({
				success: false,
				message: "새 비밀번호가 일치하지 않습니다.",
			});
		}

		// 3. 새 비밀번호 강도 검증
		if (newPassword.length < 6) {
			return res.status(400).json({
				success: false,
				message: "새 비밀번호는 최소 6자 이상이어야 합니다.",
			});
		}

		// 4. 현재 사용자 조회 (비밀번호 포함)
		const user = await User.findById(req.user._id).select("+password");
		if (!user) {
			return res.status(404).json({
				success: false,
				message: "사용자를 찾을 수 없습니다.",
			});
		}

		// 5. 현재 비밀번호 검증
		const isCurrentPasswordValid = await bcrypt.compare(
			currentPassword,
			user.password
		);

		if (!isCurrentPasswordValid) {
			console.log("현재 비밀번호 불일치");
			return res.status(400).json({
				success: false,
				message: "현재 비밀번호가 올바르지 않습니다.",
			});
		}

		// 6. 새 비밀번호가 현재 비밀번호와 같은지 확인
		const isSamePassword = await bcrypt.compare(newPassword, user.password);
		if (isSamePassword) {
			return res.status(400).json({
				success: false,
				message: "새 비밀번호는 현재 비밀번호와 달라야 합니다.",
			});
		}

		// 7. 새 비밀번호 해싱
		const saltRounds = 10;
		const hashedNewPassword = await bcrypt.hash(newPassword, saltRounds);

		// 8. 비밀번호 업데이트
		await User.findByIdAndUpdate(req.user._id, {
			password: hashedNewPassword,
			updatedAt: new Date(),
		});

		// 9. 성공 응답
		res.json({
			success: true,
			message: "비밀번호가 성공적으로 변경되었습니다.",
		});
	} catch (error) {
		console.error("비밀번호 변경 에러:", error);
		res.status(500).json({
			success: false,
			message: "서버 오류가 발생했습니다.",
		});
	}
});
```

### 1-2. API 보안 특징

-   **JWT 토큰 인증**: `authenticateToken` 미들웨어 사용
-   **현재 비밀번호 검증**: bcrypt를 통한 안전한 비교
-   **중복 비밀번호 방지**: 새 비밀번호가 현재와 같은지 확인
-   **bcrypt 해싱**: 새 비밀번호도 안전하게 해싱하여 저장

---

## 🚀 Step 2: 데이터베이스 스키마 변경

### 2-1. User 모델 업데이트

사용자 경험 개선을 위해 `username` 필드를 `name`으로 변경했습니다:

```javascript
// backend/models/User.js - 완전한 스키마
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const crypto = require("crypto");

const userSchema = new mongoose.Schema(
	{
		name: {
			type: String,
			required: [true, "이름은 필수입니다"],
			trim: true,
		},
		email: {
			type: String,
			required: [true, "이메일은 필수입니다"],
			unique: true,
			lowercase: true,
		},
		password: {
			type: String,
			required: [true, "비밀번호는 필수입니다"],
			minlength: 6,
			select: false, // 기본적으로 조회 시 제외
		},
		// 이메일 인증 관련 필드
		isEmailVerified: {
			type: Boolean,
			default: false,
		},
		emailVerificationToken: {
			type: String,
			select: false,
		},
		emailVerificationExpires: {
			type: Date,
			select: false,
		},
		refreshTokens: [
			{
				token: String,
				createdAt: {
					type: Date,
					default: Date.now,
				},
			},
		],
	},
	{
		timestamps: true, // createdAt, updatedAt 자동 생성
	}
);

// 비밀번호 해싱 미들웨어
userSchema.pre("save", async function (next) {
	if (!this.isModified("password")) return next();
	this.password = await bcrypt.hash(this.password, 12);
	next();
});

// 비밀번호 비교 메서드
userSchema.methods.comparePassword = async function (candidatePassword) {
	return await bcrypt.compare(candidatePassword, this.password);
};

// 리프레시 토큰 저장 메서드
userSchema.methods.saveRefreshToken = async function (token) {
	this.refreshTokens.push({
		token: token,
		createdAt: new Date(),
	});
	return await this.save();
};

// 리프레시 토큰 제거 메서드
userSchema.methods.clearRefreshToken = async function () {
	this.refreshTokens = [];
	return await this.save();
};

// 이메일 인증 토큰 생성
userSchema.methods.generateEmailVerificationToken = function () {
	const token = crypto.randomBytes(32).toString("hex");

	this.emailVerificationToken = crypto
		.createHash("sha256")
		.update(token)
		.digest("hex");

	// 24시간 후 만료
	this.emailVerificationExpires = Date.now() + 24 * 60 * 60 * 1000;

	return token; // 해싱되지 않은 토큰 반환 (이메일 링크용)
};

// 이메일 인증 처리
userSchema.methods.verifyEmail = function () {
	this.isEmailVerified = true;
	this.emailVerificationToken = undefined;
	this.emailVerificationExpires = undefined;
};

// 이메일 인증 토큰 유효성 검사
userSchema.methods.isEmailVerificationTokenValid = function () {
	return this.emailVerificationExpires > Date.now();
};

const User = mongoose.model("User", userSchema);
module.exports = User;
```

### 2-2. 변경 이유

1. **사용자 친화적**: "사용자명"보다 "이름"이 더 직관적
2. **중복 허용**: 같은 이름을 가진 사용자들도 가입 가능
3. **이메일 유니크**: 이메일만으로 사용자 구분 (더 실용적)

### 2-3. 관련 API 업데이트

api.js 파일에 비밀번호 변경 함수를 추가하고 전체 구조를 정리했습니다:

```javascript
// frontend/src/components/services/api.js - 완전한 코드
import axios from "axios";

const api = axios.create({
	baseURL: "http://localhost:5000/api",
	headers: { "Content-Type": "application/json" },
});

// 요청 인터셉터로 Authorization 헤더 자동 추가
api.interceptors.request.use((config) => {
	const token = localStorage.getItem("accessToken");
	if (token) config.headers.Authorization = `Bearer ${token}`;
	return config;
});

// 응답 인터셉터로 자동 토큰 갱신
api.interceptors.response.use(
	(response) => {
		// 성공 응답은 그대로 반환
		return response;
	},
	async (error) => {
		const originalRequest = error.config;

		// 401 에러이고, 아직 재시도하지 않은 요청이면
		if (error.response?.status === 401 && !originalRequest._retry) {
			// 로그인/회원가입 요청은 401이 정상적인 응답이므로 interceptor 건너뛰기
			if (
				originalRequest.url?.includes("/auth/login") ||
				originalRequest.url?.includes("/auth/register")
			) {
				return Promise.reject(error);
			}

			originalRequest._retry = true;

			try {
				// Refresh Token으로 새 Access Token 발급
				const refreshToken = localStorage.getItem("refreshToken");

				if (!refreshToken) {
					// Refresh Token도 없으면 로그아웃
					localStorage.removeItem("accessToken");
					window.location.href = "/";
					return Promise.reject(error);
				}

				const refreshResponse = await axios.post(
					"http://localhost:5000/api/auth/refresh",
					{ refreshToken }
				);

				// 새 Access Token 저장
				const newAccessToken = refreshResponse.data.accessToken;
				localStorage.setItem("accessToken", newAccessToken);

				// 원래 요청에 새 토큰 추가하고 재시도
				originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
				return api(originalRequest);
			} catch (refreshError) {
				// Refresh Token도 만료되었으면 로그아웃
				localStorage.removeItem("accessToken");
				localStorage.removeItem("refreshToken");
				window.location.href = "/";
				return Promise.reject(refreshError);
			}
		}

		return Promise.reject(error);
	}
);

// 비밀번호 변경 API
export const changePassword = async (passwordData) => {
	const response = await api.put("/auth/change-password", passwordData);
	return response.data;
};

export default api;
```

모든 관련 API에서 `username` → `name` 변경:

-   회원가입 API
-   로그인 응답
-   프로필 조회/업데이트 API

---

## 🚀 Step 3: 프론트엔드 PasswordChange 컴포넌트 구현

### 3-1. PasswordChange 컴포넌트

새로운 비밀번호 변경 전용 컴포넌트를 생성했습니다:

```jsx
// frontend/src/components/Profile/PasswordChange.jsx - 완전한 코드
import React, { useState } from "react";
import { changePassword } from "../services/api";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import "./PasswordChange.css";

const PasswordChange = ({ showToast, onCancel }) => {
	const [formData, setFormData] = useState({
		currentPassword: "",
		newPassword: "",
		confirmPassword: "",
	});
	const [loading, setLoading] = useState(false);
	const [passwordStrength, setPasswordStrength] = useState({
		level: "",
		color: "",
		requirements: [],
	});

	// 비밀번호 강도 체크
	const checkPasswordStrength = (password) => {
		const requirements = [
			{
				test: password.length >= 8,
				text: "8자 이상",
			},
			{
				test: /[A-Z]/.test(password),
				text: "대문자 포함",
			},
			{
				test: /[a-z]/.test(password),
				text: "소문자 포함",
			},
			{
				test: /\d/.test(password),
				text: "숫자 포함",
			},
			{
				test: /[!@#$%^&*(),.?":{}|<>]/.test(password),
				text: "특수문자 포함",
			},
		];

		const passedCount = requirements.filter((req) => req.test).length;

		let level, color;
		if (passedCount < 3) {
			level = "약함";
			color = "#ff4757";
		} else if (passedCount < 4) {
			level = "보통";
			color = "#ffa502";
		} else {
			level = "강함";
			color = "#2ed573";
		}

		return { level, color, requirements };
	};

	// 입력값 변경 핸들러
	const handleChange = (e) => {
		const { name, value } = e.target;
		setFormData((prev) => ({ ...prev, [name]: value }));

		// 새 비밀번호 강도 체크
		if (name === "newPassword") {
			setPasswordStrength(checkPasswordStrength(value));
		}
	};

	// 폼 제출 핸들러
	const handleSubmit = async (e) => {
		e.preventDefault();

		// 유효성 검사
		if (
			!formData.currentPassword ||
			!formData.newPassword ||
			!formData.confirmPassword
		) {
			showToast("모든 필드를 입력해주세요.", "error");
			return;
		}

		if (formData.newPassword !== formData.confirmPassword) {
			return; // Toast 없이 그냥 리턴 (폼에서 이미 에러 표시 중)
		}

		try {
			setLoading(true);
			const passwordData = {
				currentPassword: formData.currentPassword,
				newPassword: formData.newPassword,
				confirmPassword: formData.confirmPassword,
			};
			await changePassword(passwordData);
			showToast("비밀번호가 성공적으로 변경되었습니다!", "success");
			onCancel(); // 폼 닫기
		} catch (error) {
			console.error("비밀번호 변경 에러:", error);
			console.error("에러 응답:", error.response);

			let errorMessage = "비밀번호 변경에 실패했습니다.";

			if (error.response) {
				// 서버에서 응답을 받은 경우
				if (error.response.data && error.response.data.message) {
					errorMessage = error.response.data.message;
				} else if (error.response.status === 400) {
					errorMessage = "입력 정보를 확인해주세요.";
				} else if (error.response.status === 401) {
					errorMessage = "로그인이 필요합니다.";
				} else if (error.response.status === 500) {
					errorMessage = "서버 오류가 발생했습니다.";
				}
			} else if (error.request) {
				// 네트워크 오류
				errorMessage = "네트워크 연결을 확인해주세요.";
			}

			console.log("표시할 에러 메시지:", errorMessage);
			showToast(errorMessage, "error");
		} finally {
			setLoading(false);
		}
	};

	return (
		<>
			<form onSubmit={handleSubmit} className="password-change-form">
				{/* 현재 비밀번호 */}
				<div className="form-group">
					<label htmlFor="currentPassword">현재 비밀번호</label>
					<input
						type="password"
						id="currentPassword"
						name="currentPassword"
						value={formData.currentPassword}
						onChange={handleChange}
						placeholder="현재 비밀번호를 입력하세요"
						autoComplete="current-password"
						required
					/>
				</div>

				{/* 새 비밀번호 */}
				<div className="form-group">
					<label htmlFor="newPassword">새 비밀번호</label>
					<input
						type="password"
						id="newPassword"
						name="newPassword"
						value={formData.newPassword}
						onChange={handleChange}
						placeholder="새 비밀번호를 입력하세요"
						autoComplete="new-password"
						required
					/>

					{/* 비밀번호 강도 표시 */}
					{formData.newPassword && (
						<div className="password-strength">
							<div className="strength-bar">
								<div
									className="strength-fill"
									style={% raw %}{{
										width: `${
											(passwordStrength.requirements.filter(
												(req) => req.test
											).length /
												5) *
											100
										}%`,
										backgroundColor: passwordStrength.color,
									}}{% endraw %}
								></div>
							</div>
							<span
								className="strength-text"
								style={% raw %}{{ color: passwordStrength.color }}{% endraw %}
							>
								{passwordStrength.level}
								{passwordStrength.level !== "강함" && (
									<span className="strength-requirement">
										(비밀번호 변경을 위해 '강함' 등급이
										필요합니다)
									</span>
								)}
							</span>
							<ul className="requirements-list">
								{passwordStrength.requirements.map(
									(req, index) => (
										<li
											key={index}
											className={
												req.test ? "fulfilled" : ""
											}
										>
											{req.test ? "✓" : "○"} {req.text}
										</li>
									)
								)}
							</ul>
						</div>
					)}
				</div>

				{/* 새 비밀번호 확인 */}
				<div className="form-group">
					<label htmlFor="confirmPassword">새 비밀번호 확인</label>
					<input
						type="password"
						id="confirmPassword"
						name="confirmPassword"
						value={formData.confirmPassword}
						onChange={handleChange}
						placeholder="새 비밀번호를 다시 입력하세요"
						autoComplete="new-password"
						required
					/>
					{formData.confirmPassword &&
						formData.newPassword !== formData.confirmPassword && (
							<span className="error-text">
								비밀번호가 일치하지 않습니다.
							</span>
						)}
				</div>

				{/* 버튼 */}
				<div className="form-actions">
					<button
						type="button"
						onClick={onCancel}
						className="cancel-button"
						disabled={loading}
					>
						취소
					</button>
					<button
						type="submit"
						className="submit-button"
						disabled={
							loading ||
							passwordStrength.level !== "강함" ||
							formData.newPassword !== formData.confirmPassword ||
							!formData.currentPassword ||
							!formData.newPassword ||
							!formData.confirmPassword
						}
					>
						{loading ? (
							<LoadingSpinner size="small" />
						) : (
							"비밀번호 변경"
						)}
					</button>
				</div>
			</form>
		</>
	);
};

export default PasswordChange;
```

### 2. Profile 컴포넌트 통합

기존 Profile 컴포넌트에 비밀번호 변경 모드를 추가했습니다:

```jsx
// frontend/src/components/Profile/Profile.jsx
const Profile = () => {
	const [editMode, setEditMode] = useState(false);
	const [passwordChangeMode, setPasswordChangeMode] = useState(false);

	return (
		<div className="profile-container">
			<div className="profile-card">
				<div className="profile-header">
					<button
						className="back-button"
						onClick={handleBackToDashboard}
					>
						←
					</button>
					<h1
						key={
							passwordChangeMode
								? "password-title"
								: editMode
								? "edit-title"
								: "profile-title"
						}
						className="profile-title"
					>
						{passwordChangeMode
							? "비밀번호 변경"
							: editMode
							? "정보 변경"
							: "프로필"}
					</h1>
				</div>

				{editMode ? (
					<ProfileEdit {...props} />
				) : passwordChangeMode ? (
					<PasswordChange {...props} />
				) : (
					<ProfileView {...props} />
				)}
			</div>
		</div>
	);
};
```

### 3-3. ProfileEdit 컴포넌트 통합

ProfileEdit 컴포넌트를 PasswordChange와 동일한 스타일을 사용하도록 수정했습니다:

```jsx
// frontend/src/components/Profile/ProfileEdit.jsx - 완전한 코드
import React, { useState } from "react";
import api from "../services/api";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import "./PasswordChange.css"; // 중요: PasswordChange.css 사용

const ProfileEdit = ({ user, onUpdateSuccess, onCancel, showToast }) => {
	const [formData, setFormData] = useState({
		name: user.name,
		email: user.email,
	});
	const [loading, setLoading] = useState(false);
	const [errors, setErrors] = useState({});

	// 입력값 변경 핸들러
	const handleChange = (e) => {
		const { name, value } = e.target;
		setFormData((prev) => ({
			...prev,
			[name]: value,
		}));

		// 입력 시 해당 필드 에러 제거
		if (errors[name]) {
			setErrors((prev) => ({
				...prev,
				[name]: "",
			}));
		}
	};

	// 폼 유효성 검사
	const validateForm = () => {
		const newErrors = {};

		if (!formData.name.trim()) {
			newErrors.name = "이름을 입력해주세요.";
		}

		if (!formData.email.trim()) {
			newErrors.email = "이메일을 입력해주세요.";
		} else if (!/\S+@\S+\.\S+/.test(formData.email)) {
			newErrors.email = "올바른 이메일 형식을 입력해주세요.";
		}

		setErrors(newErrors);
		return Object.keys(newErrors).length === 0;
	};

	// 프로필 업데이트 제출
	const handleSubmit = async (e) => {
		e.preventDefault();

		if (!validateForm()) {
			return;
		}

		try {
			setLoading(true);
			const response = await api.put("/auth/profile", formData);

			if (response.data.success) {
				onUpdateSuccess(response.data.user);
				showToast("프로필이 성공적으로 업데이트되었습니다!", "success");
			}
		} catch (error) {
			console.error("프로필 업데이트 에러:", error);

			let errorMessage = "프로필 업데이트에 실패했습니다.";

			if (error.response?.data?.message) {
				errorMessage = error.response.data.message;
			} else if (error.response?.status === 409) {
				errorMessage = "이미 사용 중인 이메일입니다.";
			}

			showToast(errorMessage, "error");
		} finally {
			setLoading(false);
		}
	};

	return (
		<form onSubmit={handleSubmit} className="password-change-form">
			{/* 이름 필드 */}
			<div className="form-group">
				<label htmlFor="name">이름</label>
				<input
					type="text"
					id="name"
					name="name"
					value={formData.name}
					onChange={handleChange}
					placeholder="이름을 입력하세요"
					required
				/>
				{errors.name && (
					<span className="error-text">{errors.name}</span>
				)}
			</div>

			{/* 이메일 필드 */}
			<div className="form-group">
				<label htmlFor="email">이메일</label>
				<input
					type="email"
					id="email"
					name="email"
					value={formData.email}
					onChange={handleChange}
					placeholder="이메일을 입력하세요"
					required
				/>
				{errors.email && (
					<span className="error-text">{errors.email}</span>
				)}
			</div>

			{/* 버튼 */}
			<div className="form-actions">
				<button
					type="button"
					onClick={onCancel}
					className="cancel-button"
					disabled={loading}
				>
					취소
				</button>
				<button
					type="submit"
					className="save-button"
					disabled={loading || Object.keys(errors).length > 0}
				>
					{loading ? <LoadingSpinner /> : "저장"}
				</button>
			</div>
		</form>
	);
};

export default ProfileEdit;
```

### 3-4. Profile 메인 컴포넌트 전체 업데이트

실제 Profile.jsx 컴포넌트의 완전한 구조입니다:

```jsx
// frontend/src/components/Profile/Profile.jsx - 완전한 실제 코드
import React, { useState, useEffect, useContext, useCallback } from "react";
import { useNavigate } from "react-router-dom";
import { AuthContext } from "../context/AuthContext";
import api from "../services/api";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import Toast from "../common/UI/Toast";
import ProfileEdit from "./ProfileEdit";
import PasswordChange from "./PasswordChange";
import "./Profile.css";

const Profile = () => {
	const { token } = useContext(AuthContext);
	const navigate = useNavigate();
	const [user, setUser] = useState(null);
	const [loading, setLoading] = useState(true);
	const [editMode, setEditMode] = useState(false);
	const [passwordChangeMode, setPasswordChangeMode] = useState(false);
	const [toast, setToast] = useState({
		isVisible: false,
		message: "",
		type: "info",
	});

	const fetchUserProfile = useCallback(async () => {
		try {
			setLoading(true);
			const response = await api.get("/auth/me");

			if (response.data.success) {
				setUser(response.data.user);
			} else {
				showToast("사용자 정보를 가져올 수 없습니다.", "error");
			}
		} catch (error) {
			console.error("프로필 조회 에러:", error);
			showToast("서버 오류가 발생했습니다.", "error");
		} finally {
			setLoading(false);
		}
	}, []);

	// 토스트 메시지 표시
	const showToast = (message, type = "info") => {
		setToast({
			isVisible: true,
			message,
			type,
		});
	};

	// 토스트 메시지 닫기
	const closeToast = () => {
		setToast((prev) => ({
			...prev,
			isVisible: false,
		}));
	};

	// 대시보드로 돌아가기
	const handleBackToDashboard = () => {
		navigate("/dashboard");
	};

	// 프로필 업데이트 성공 핸들러
	const handleProfileUpdate = (updatedUser) => {
		setUser(updatedUser);
		setEditMode(false);
		showToast("프로필이 성공적으로 업데이트되었습니다!", "success");
	};

	// 편집 모드 취소
	const handleEditCancel = () => {
		setEditMode(false);
	};

	// 비밀번호 변경 모드 취소
	const handlePasswordChangeCancel = () => {
		setPasswordChangeMode(false);
	};

	// 컴포넌트 마운트 시 사용자 정보 가져오기
	useEffect(() => {
		if (token) {
			fetchUserProfile();
		}
	}, [token, fetchUserProfile]);

	if (loading) {
		return (
			<div className="profile-container">
				<LoadingSpinner />
			</div>
		);
	}

	if (!user) {
		return (
			<div className="profile-container">
				<div className="profile-error">
					사용자 정보를 불러올 수 없습니다.
				</div>
			</div>
		);
	}

	return (
		<div className="profile-container">
			<div className="profile-card">
				<div className="profile-header">
					<button
						className="back-button"
						onClick={handleBackToDashboard}
						title="대시보드로 돌아가기"
					>
						←
					</button>
					<h1
						key={
							passwordChangeMode
								? "password-title"
								: editMode
								? "edit-title"
								: "profile-title"
						}
						className="profile-title"
					>
						{passwordChangeMode
							? "비밀번호 변경"
							: editMode
							? "정보 변경"
							: "프로필"}
					</h1>
				</div>

				{editMode ? (
					<div key="edit-mode" className="profile-content-transition">
						<ProfileEdit
							user={user}
							onUpdateSuccess={handleProfileUpdate}
							onCancel={handleEditCancel}
							showToast={showToast}
						/>
					</div>
				) : passwordChangeMode ? (
					<div
						key="password-mode"
						className="profile-content-transition"
					>
						<PasswordChange
							showToast={showToast}
							onCancel={handlePasswordChangeCancel}
						/>
					</div>
				) : (
					<div
						key="profile-mode"
						className="profile-content-transition"
					>
						<div className="profile-info">
							<div className="info-group">
								<label>이름</label>
								<div className="info-value">{user.name}</div>
							</div>

							<div className="info-group">
								<label>이메일</label>
								<div className="info-value">{user.email}</div>
							</div>

							<div className="info-group">
								<label>가입일</label>
								<div className="info-value">
									{new Date(
										user.createdAt
									).toLocaleDateString("ko-KR")}
								</div>
							</div>

							<div className="info-group">
								<label>최근 수정일</label>
								<div className="info-value">
									{new Date(
										user.updatedAt || user.createdAt
									).toLocaleDateString("ko-KR")}
								</div>
							</div>
						</div>

						<div className="profile-actions">
							<button
								className="edit-button"
								onClick={() => setEditMode(true)}
							>
								정보 변경
							</button>
							<button
								className="password-change-button"
								onClick={() => setPasswordChangeMode(true)}
							>
								비밀번호 변경
							</button>
						</div>
					</div>
				)}
			</div>

			{toast.isVisible && (
				<Toast
					message={toast.message}
					type={toast.type}
					isVisible={toast.isVisible}
					onClose={closeToast}
				/>
			)}
		</div>
	);
};

export default Profile;
    	return Object.keys(newErrors).length === 0;
    };

    // 프로필 업데이트 제출
    const handleSubmit = async (e) => {
    	e.preventDefault();

    	if (!validateForm()) {
    		return;
    	}

    	// 변경사항이 없는 경우
    	if (formData.name === user.name && formData.email === user.email) {
    		showToast("변경된 내용이 없습니다.", "info");
    		return;
    	}

    	try {
    		setLoading(true);
    		const response = await api.put("/auth/profile", formData);

    		if (response.data.success) {
    			onUpdateSuccess(response.data.user);
    		} else {
    			showToast(
    				response.data.message || "프로필 업데이트에 실패했습니다.",
    				"error"
    			);
    		}
    	} catch (error) {
    		console.error("프로필 업데이트 에러:", error);

    		if (error.response?.data?.message) {
    			showToast(error.response.data.message, "error");
    		} else {
    			showToast("서버 오류가 발생했습니다.", "error");
    		}
    	} finally {
    		setLoading(false);
    	}
    };

    return (
    	<form onSubmit={handleSubmit} className="profile-edit-form">
    		{/* 중요: form-group 클래스 사용 (input-group에서 변경) */}
    		<div className="form-group">
    			<label htmlFor="name">이름</label>
    			<input
    				type="text"
    				id="name"
    				name="name"
    				value={formData.name}
    				onChange={handleChange}
    				className={errors.name ? "error" : ""}
    				placeholder="이름을 입력하세요"
    			/>
    			{errors.name && (
    				<span className="error-text">{errors.name}</span>
    			)}
    		</div>

    		<div className="form-group">
    			<label htmlFor="email">이메일</label>
    			<input
    				type="email"
    				id="email"
    				name="email"
    				value={formData.email}
    				onChange={handleChange}
    				className={errors.email ? "error" : ""}
    				placeholder="이메일을 입력하세요"
    			/>
    			{errors.email && (
    				<span className="error-text">{errors.email}</span>
    			)}
    		</div>

    		<div className="button-group">
    			<button
    				type="button"
    				onClick={onCancel}
    				className="cancel-button"
    				disabled={loading}
    			>
    				취소
    			</button>
    			<button
    				type="submit"
    				className="save-button"
    				disabled={loading}
    			>
    				{loading ? <LoadingSpinner /> : "저장"}
    			</button>
    		</div>
    	</form>
    );

};

export default ProfileEdit;

```

---

## 🚀 Step 4: UI/UX 개선 및 스타일링

### 4-1. 통합된 스타일링

ProfileEdit와 PasswordChange 컴포넌트가 동일한 스타일을 사용하도록 통합했습니다:

```jsx
// ProfileEdit.jsx에 PasswordChange.css import 추가
import "./PasswordChange.css";

// 클래스명도 통일
<div className="form-group"> // input-group → form-group
```

### 2. 애니메이션 시스템

React key를 활용한 부드러운 전환 애니메이션을 구현했습니다:

```css
/* Profile.css */
@keyframes titleChange {
	0% {
		opacity: 0;
		transform: translateY(-8px);
	}
	100% {
		opacity: 1;
		transform: translateY(0);
	}
}

@keyframes contentSlideIn {
	0% {
		opacity: 0;
		transform: translateY(15px);
	}
	100% {
		opacity: 1;
		transform: translateY(0);
	}
}

.profile-content-transition {
	animation: contentSlideIn 0.4s ease-out;
}
```

### 3. 버튼 효과 통일

모든 버튼에 일관된 호버 효과를 적용했습니다:

```css
.edit-button::before,
.password-change-button::before,
.submit-button::before {
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

.edit-button:hover::before {
	left: 100%;
}
```

### 4. 반응형 디자인 최적화

모바일과 데스크톱에서 모두 최적화된 레이아웃을 제공합니다:

```css
@media (max-width: 768px) {
	.profile-actions {
		flex-direction: column;
		gap: 0.8rem;
	}

	.edit-button,
	.password-change-button {
		width: 100%;
	}
}
```

---

## 🚀 Step 5: 보안 강화 및 검증 시스템

### 5-1. 실시간 비밀번호 강도 검증

사용자가 입력하는 동안 실시간으로 비밀번호 강도를 표시합니다:

```jsx
{
	formData.newPassword && (
		<div className="password-strength">
			<div className="strength-bar">
				<div
					className="strength-fill"
					style={% raw %}{{
						width: `${
							(passwordStrength.requirements.filter(
								(req) => req.test
							).length /
								5) *
							100
						}%`,
						backgroundColor: passwordStrength.color,
					}}{% endraw %}
				/>
			</div>
			<span
				className="strength-text"
				style={% raw %}{{ color: passwordStrength.color }}{% endraw %}
			>
				{passwordStrength.level}
				{passwordStrength.level !== "강함" && (
					<span className="strength-requirement">
						(비밀번호 변경을 위해 '강함' 등급이 필요합니다)
					</span>
				)}
			</span>
			<ul className="requirements-list">
				{passwordStrength.requirements.map((req, index) => (
					<li key={index} className={req.test ? "fulfilled" : ""}>
						{req.test ? "✓" : "○"} {req.text}
					</li>
				))}
			</ul>
		</div>
	);
}
```

### 2. 5단계 보안 요구사항

-   ✅ **8자 이상**: 최소 길이 요구사항
-   ✅ **대문자 포함**: A-Z 문자 필수
-   ✅ **소문자 포함**: a-z 문자 필수
-   ✅ **숫자 포함**: 0-9 숫자 필수
-   ✅ **특수문자 포함**: 특수기호 필수

### 3. 강함 등급 필수

비밀번호 변경 버튼은 "강함" 등급(5개 조건 모두 충족)일 때만 활성화됩니다:

```jsx
<button
	type="submit"
	className="submit-button"
	disabled={
		loading ||
		passwordStrength.level !== "강함" ||
		formData.newPassword !== formData.confirmPassword ||
		!formData.currentPassword
	}
>
	{loading ? <LoadingSpinner size="small" /> : "비밀번호 변경"}
</button>
```

### 4. 서버사이드 검증

클라이언트 검증과 별도로 서버에서도 모든 입력을 재검증합니다:

-   현재 비밀번호 bcrypt 검증
-   새 비밀번호 강도 검사
-   중복 비밀번호 방지
-   SQL 인젝션 등 보안 위협 차단

## 기술적 세부사항

### 1. 완전한 PasswordChange.css 파일

다크테마 기반의 완전한 스타일링 파일입니다. 이 파일을 정확히 복사해야 합니다:

```css
/* frontend/src/components/Profile/PasswordChange.css */
/* PasswordChange 컴포넌트 다크모드 스타일 */
.password-change-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 24px;
	border-bottom: 1px solid rgba(255, 255, 255, 0.1);
	padding-bottom: 16px;
}

.password-change-header h3 {
	margin: 0;
	color: #e0e6ed;
	font-size: 1.5rem;
	font-weight: 600;
	text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.close-button {
	background: rgba(255, 255, 255, 0.1);
	border: 1px solid rgba(255, 255, 255, 0.2);
	font-size: 1.2rem;
	cursor: pointer;
	color: #b0b7c3;
	padding: 8px;
	border-radius: 8px;
	transition: all 0.3s ease;
	display: flex;
	align-items: center;
	justify-content: center;
	width: 32px;
	height: 32px;
}

.close-button:hover {
	background: rgba(255, 255, 255, 0.2);
	color: #e0e6ed;
	transform: scale(1.1);
}

.password-change-form {
	display: flex;
	flex-direction: column;
	gap: 20px;
}

.form-group {
	display: flex;
	flex-direction: column;
	gap: 8px;
}

.form-group label {
	font-weight: 600;
	color: #b0b7c3;
	font-size: 0.9rem;
	margin-bottom: 4px;
}

.form-group input {
	padding: 12px 16px;
	border: 2px solid rgba(255, 255, 255, 0.1);
	border-radius: 8px;
	font-size: 1rem;
	background: rgba(255, 255, 255, 0.05);
	color: #e0e6ed;
	transition: all 0.3s ease;
}

.form-group input::placeholder {
	color: #6c7378;
}

.form-group input:focus {
	outline: none;
	border-color: #3b82f6;
	background: rgba(59, 130, 246, 0.1);
	box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.2);
}

/* 비밀번호 강도 표시 */
.password-strength {
	margin-top: 12px;
	padding: 12px;
	background: rgba(255, 255, 255, 0.03);
	border-radius: 8px;
	border: 1px solid rgba(255, 255, 255, 0.08);
}

.strength-bar {
	width: 100%;
	height: 6px;
	background: rgba(255, 255, 255, 0.1);
	border-radius: 3px;
	overflow: hidden;
	margin-bottom: 8px;
}

.strength-fill {
	height: 100%;
	transition: width 0.3s ease, background-color 0.3s ease;
	border-radius: 3px;
}

.strength-text {
	font-size: 0.85rem;
	font-weight: 600;
	margin-bottom: 8px;
	display: block;
}

.strength-requirement {
	font-size: 0.75rem;
	font-weight: 400;
	color: #ffa502;
	margin-left: 8px;
	font-style: italic;
}

.requirements-list {
	list-style: none;
	padding: 0;
	margin: 0;
	display: grid;
	grid-template-columns: 1fr 1fr;
	gap: 6px;
}

.requirements-list li {
	font-size: 0.8rem;
	color: #6c7378;
	padding: 2px 0;
	transition: color 0.2s;
}

.requirements-list li.fulfilled {
	color: #22c55e;
}

/* 에러 텍스트 */
.error-text {
	color: #ef4444;
	font-size: 0.8rem;
	margin-top: 4px;
	font-weight: 500;
}

/* 폼 액션 버튼 */
.form-actions {
	display: flex;
	gap: 12px;
	justify-content: flex-end;
	margin-top: 24px;
	padding-top: 20px;
	border-top: 1px solid rgba(255, 255, 255, 0.1);
}

.cancel-button,
.submit-button {
	flex: 1;
	padding: 12px 20px;
	border-radius: 8px;
	font-size: 0.9rem;
	font-weight: 600;
	cursor: pointer;
	transition: all 0.3s ease;
	display: flex;
	align-items: center;
	gap: 8px;
	justify-content: center;
	position: relative;
	overflow: hidden;
}

.cancel-button {
	background: linear-gradient(135deg, #6b7280 0%, #4b5563 100%);
	border: none;
	color: white;
	box-shadow: 0 4px 8px rgba(107, 114, 128, 0.3);
}

.cancel-button::before {
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

.cancel-button:hover::before {
	left: 100%;
}

.cancel-button:hover:not(:disabled) {
	background: linear-gradient(135deg, #9ca3af 0%, #6b7280 100%);
	transform: translateY(-3px);
	box-shadow: 0 12px 24px rgba(107, 114, 128, 0.5), 0 4px 8px rgba(107, 114, 128, 0.4);
}

.cancel-button:active {
	transform: translateY(0);
}

.submit-button {
	background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%);
	border: none;
	color: white;
	box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.submit-button::before {
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

.submit-button:hover::before {
	left: 100%;
}

.submit-button:hover:not(:disabled) {
	background: linear-gradient(135deg, #8b5cf6 0%, #3b82f6 100%);
	transform: translateY(-3px);
	box-shadow: 0 12px 24px rgba(139, 92, 246, 0.5), 0 4px 8px rgba(139, 92, 246, 0.4);
}

.submit-button:active {
	transform: translateY(0);
}

.cancel-button:disabled,
.submit-button:disabled {
	opacity: 0.6;
	cursor: not-allowed;
	transform: none;
}

/* 반응형 디자인 */
@media (max-width: 768px) {
	.requirements-list {
		grid-template-columns: 1fr;
	}
}

@media (max-width: 480px) {
	.password-change-header h3 {
		font-size: 1.3rem;
	}
}
```

### 2. 중요한 스타일 통합 수정사항

**ProfileEdit.jsx에서 반드시 해야 할 변경사항:**

1. **CSS 파일 import 변경:**

```jsx
import "./PasswordChange.css"; // 이 줄 추가
```

2. **클래스명 변경:**

```jsx
// 기존: className="input-group"
// 변경: className="form-group"
<div className="form-group">
```

3. **버튼 클래스명:**

```jsx
// 정보 변경 버튼들
<div className="button-group">
	{" "}
	// form-actions가 아님
	<button className="cancel-button">취소</button>
	<button className="save-button">저장</button> // submit-button이 아님
</div>
```

### 4. Toast.css z-index 수정

Toast 알림이 최상위에 표시되도록 z-index를 수정해야 합니다:

```css
/* frontend/src/components/common/UI/Toast.css */
.toast {
	position: fixed;
	top: 20px;
	right: 20px;
	z-index: 9999; /* 기존: 1000에서 9999로 변경 */
	min-width: 300px;
	max-width: 500px;
	/* ... 나머지 스타일은 동일 */
}
```

### 5. 데이터베이스 마이그레이션 명령어

기존 username 데이터가 있다면 다음 명령어로 정리하세요:

```javascript
// MongoDB에서 실행 (백엔드 디렉터리에서)
node -e "
require('dotenv').config();
const mongoose = require('mongoose');
const User = require('./models/User');
mongoose.connect(process.env.MONGO_URI).then(async () => {
	console.log('DB 연결 성공');
	const result = await User.deleteMany({});
	console.log('삭제된 사용자 수:', result.deletedCount);
	console.log('모든 사용자 데이터가 삭제되었습니다!');
	process.exit(0);
}).catch(err => {
	console.error('오류:', err);
	process.exit(1);
});
"
```

## 🚨 **포스팅을 따라할 때 주의사항**

### 1. 필수 순서

1. **백엔드 먼저**: User.js 모델 변경 → auth.js API 추가
2. **프론트엔드**: PasswordChange.css 생성 → 컴포넌트들 작성
3. **스타일 통합**: ProfileEdit.jsx에서 import와 클래스명 수정
4. **테스트**: 각 단계별로 기능 확인

### 2. 실수하기 쉬운 부분

❌ **잘못된 예시:**

```jsx
// ProfileEdit.jsx에서 이렇게 하면 안됨
<div className="input-group"> // ❌
import "./Profile.css"; // ❌
```

✅ **올바른 예시:**

```jsx
// ProfileEdit.jsx에서 반드시 이렇게
<div className="form-group"> // ✅
import "./PasswordChange.css"; // ✅
```

### 3. 디버깅 체크포인트

각 단계별로 확인해야 할 사항:

1. **백엔드 테스트**: Postman으로 `/auth/change-password` API 호출
2. **스타일 확인**: 정보변경과 비밀번호변경 입력박스가 동일한지 확인
3. **애니메이션 확인**: 모드 전환 시 부드러운 애니메이션 작동
4. **반응형 확인**: 모바일에서도 버튼이 가로배치되는지 확인

### 4. 파일 구조 변경

```
backend/
├── routes/auth.js (비밀번호 변경 API 추가)
└── models/User.js (username → name 변경)

frontend/src/components/
├── Profile/
│   ├── Profile.jsx (메인 프로필 컴포넌트)
│   ├── Profile.css (통합 스타일링)
│   ├── ProfileEdit.jsx (정보 변경)
│   ├── PasswordChange.jsx (비밀번호 변경)
│   └── PasswordChange.css (다크테마 스타일)
├── services/
│   └── api.js (changePassword 함수 추가)
└── common/UI/
    └── Toast.css (z-index 수정)
```

### 2. 상태 관리

React useState를 활용한 효율적인 상태 관리:

```jsx
const [formData, setFormData] = useState({
	currentPassword: "",
	newPassword: "",
	confirmPassword: "",
});
const [loading, setLoading] = useState(false);
const [passwordStrength, setPasswordStrength] = useState({
	level: "",
	color: "",
	requirements: [],
});
```

### 3. 에러 처리

포괄적인 에러 처리 시스템:

```jsx
try {
	await changePassword(passwordData);
	showToast("비밀번호가 성공적으로 변경되었습니다!", "success");
} catch (error) {
	let errorMessage = "비밀번호 변경에 실패했습니다.";

	if (error.response?.data?.message) {
		errorMessage = error.response.data.message;
	} else if (error.response?.status === 400) {
		errorMessage = "입력 정보를 확인해주세요.";
	}

	showToast(errorMessage, "error");
}
```

## 성능 최적화

### 1. 실시간 검증 최적화

비밀번호 입력 시에만 강도 검사를 실행하여 불필요한 연산을 방지했습니다:

```jsx
const handleChange = (e) => {
	const { name, value } = e.target;
	setFormData((prev) => ({ ...prev, [name]: value }));

	// 새 비밀번호 입력 시에만 강도 체크
	if (name === "newPassword") {
		setPasswordStrength(checkPasswordStrength(value));
	}
};
```

### 2. CSS 애니메이션 최적화

GPU 가속을 활용한 부드러운 애니메이션:

```css
.profile-title {
	transition: all 0.3s ease;
	animation: titleChange 0.3s ease-out;
}

.profile-content-transition {
	animation: contentSlideIn 0.4s cubic-bezier(0.25, 0.46, 0.45, 0.94);
}
```

## 사용자 경험 개선

### 1. 직관적인 네비게이션

-   **백 버튼**: 대시보드로 즉시 이동
-   **동적 제목**: 현재 모드에 따라 제목 변경
-   **부드러운 전환**: 모드 변경 시 애니메이션 효과

### 2. 시각적 피드백

-   **실시간 강도 표시**: 프로그레스 바와 색상으로 강도 표시
-   **요구사항 체크리스트**: 충족/미충족 상태를 시각적으로 표시
-   **버튼 상태**: 조건 충족 시에만 활성화

### 3. 반응형 디자인

-   **모바일 최적화**: 터치 친화적인 버튼 크기
-   **유연한 레이아웃**: 화면 크기에 따른 적응형 디자인
-   **일관된 스타일**: 모든 디바이스에서 동일한 사용자 경험

## 향후 개선 계획

### 1. 추가 보안 기능

-   [ ] 2단계 인증 (2FA) 통합
-   [ ] 비밀번호 변경 시 이메일 알림
-   [ ] 로그인 세션 강제 만료
-   [ ] 비밀번호 히스토리 관리

### 2. 사용자 경험 향상

-   [ ] 비밀번호 생성 도구
-   [ ] 키보드 단축키 지원
-   [ ] 다크/라이트 테마 토글
-   [ ] 접근성(A11y) 개선

### 3. 성능 최적화

-   [ ] 코드 스플리팅
-   [ ] 이미지 최적화
-   [ ] 캐싱 전략 개선
-   [ ] 번들 크기 최적화

---

## 🎯 마무리

이번 **Part 6-3**에서는 **안전한 비밀번호 변경 시스템**을 완전히 구현했습니다. **실시간 강도 검증**부터 **5단계 보안 요구사항**, **부드러운 애니메이션**까지 프로덕션 레벨의 보안 시스템을 만들었습니다.

### 🏆 주요 구현 성과

#### ✅ **백엔드 보안 강화**

-   JWT 인증 기반 비밀번호 변경 API
-   bcrypt를 활용한 안전한 해싱
-   현재 비밀번호 검증 및 중복 방지

#### ✅ **프론트엔드 UX 혁신**

-   실시간 비밀번호 강도 검증 시스템
-   5단계 보안 요구사항 (8자 이상, 대소문자, 숫자, 특수문자)
-   직관적인 시각적 피드백 (진행률 바, 색상 변화)

#### ✅ **완전한 UI/UX 통합**

-   Profile 시스템과 매끄러운 통합
-   다크테마 기반 일관된 디자인
-   부드러운 애니메이션과 트랜지션 효과

#### ✅ **개발자 친화적 구조**

-   모듈화된 컴포넌트 아키텍처
-   재사용 가능한 스타일 시스템
-   상세한 에러 처리 및 디버깅 지원

### 🚀 **다음 단계 계획**

**Part 7**에서는 더욱 고급화된 보안 기능들을 구현할 예정입니다:

-   **2단계 인증 (2FA)** 시스템
-   **비밀번호 변경 시 이메일 알림**
-   **로그인 세션 강제 만료** 기능
-   **비밀번호 히스토리 관리**

### 💡 **학습 포인트**

1. **보안**: bcrypt 해싱과 JWT 인증의 실전 활용
2. **UX**: 실시간 피드백으로 사용자 경험 향상
3. **아키텍처**: 재사용 가능한 컴포넌트 설계
4. **성능**: 최적화된 React 상태 관리

이제 **완전한 사용자 인증 및 관리 시스템**을 갖추게 되었습니다! 🎉

---

### 🔗 소스코드

전체 소스코드는 [GitHub 저장소](https://github.com/hoondongseo/SimpleAuthSystem)에서 확인하실 수 있습니다.

```bash
git clone https://github.com/hoondongseo/SimpleAuthSystem.git
cd SimpleAuthSystem

# 백엔드 실행
cd backend
npm install
npm start

# 프론트엔드 실행
cd ../frontend
npm install
npm start
```

**🌟 이 글이 도움이 되셨다면 GitHub Star를 눌러주시고, 궁금한 점은 언제든 댓글로 남겨주세요!**
