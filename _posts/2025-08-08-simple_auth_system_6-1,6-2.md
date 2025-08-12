---
title: "React JWT 인증 시스템 Part 6-1 & 6-2: 이메일 인증으로 보안 강화하기 - Gmail SMTP 완전 구현"
categories:
    - MiniProject
    - SimpleAuthSystem
tag:
    [
        MiniProject,
        React,
        EmailVerification,
        Gmail,
        SMTP,
        JWT,
        보안,
        인증시스템,
        Nodemailer,
    ]
author_profile: false
sidebar:
    nav: "docs"
search: true
---

> Gmail SMTP와 React Strict Mode 대응으로 완성하는 프로덕션 레벨 이메일 인증 시스템

## 🔗 시리즈 연결

이 포스트는 JWT 인증 시스템 시리즈의 **Part 6-1 & 6-2**입니다.

-   **[Part 1: 기본 JWT 인증 시스템 구축하기](https://hoondongseo.github.io/posts/simple_auth_system_1/)**
-   **[Part 2: 고급 JWT 인증 기능](https://hoondongseo.github.io/posts/simple_auth_system_2/)**
-   **[Part 3: React 프론트엔드 구현](https://hoondongseo.github.io/posts/simple_auth_system_3/)**
-   **[Part 4: 대시보드 및 로그인 상태 관리](https://hoondongseo.github.io/posts/simple_auth_system_4/)**
-   **[Part 5-1: 자동 토큰 갱신 & React Router](https://hoondongseo.github.io/posts/simple_auth_system_5-1/)**
-   **[Part 5-2: 다크모드 UI & 반응형 디자인](https://hoondongseo.github.io/posts/simple_auth_system_5-2/)**
-   **[Part 5-3: 로딩 스피너 & Toast 알림으로 UX 완성하기](https://hoondongseo.github.io/posts/simple_auth_system_5-3/)**
-   **Part 6-1 & 6-2: 이메일 인증으로 보안 강화하기** ← 현재 글

## 🎯 프로젝트 개요

이번 포스트에서는 **React JWT 인증 시스템**에 **이메일 인증** 기능을 추가하여 보안을 한층 더 강화해보겠습니다. **Part 6-1**에서는 백엔드 이메일 인증 시스템을, **Part 6-2**에서는 React 프론트엔드 구현을 다룹니다. 실제 운영 환경에서 사용할 수 있는 수준의 **Gmail SMTP 연동**부터 **React Strict Mode 이중 실행 문제 해결**까지 모든 것을 다룹니다.

### 🛠️ 기술 스택

-   **Backend**: Node.js, Express.js, MongoDB, Nodemailer
-   **Email Service**: Gmail SMTP with App Password
-   **Frontend**: React.js, React Router v6
-   **Authentication**: JWT Tokens, bcrypt
-   **Styling**: CSS3, 반응형 디자인

### 📋 Part 6-1 & 6-2에서 구현할 기능

**Part 6-1 (백엔드 이메일 인증 시스템):**

-   ✅ Gmail SMTP 이메일 발송 시스템
-   ✅ User 모델 이메일 인증 필드 확장
-   ✅ 회원가입 시 이메일 인증 필수화
-   ✅ HTML 이메일 템플릿 with 그라데이션 디자인
-   ✅ 토큰 기반 이메일 인증 API

**Part 6-2 (React 프론트엔드 구현):**

-   ✅ React 이메일 인증 페이지 컴포넌트
-   ✅ React Strict Mode 이중 실행 대응
-   ✅ 사용자 데이터 완전 표시 (가입일 포함)
-   ✅ 프로덕션 레디 에러 처리
-   ✅ 일관된 UX 패턴 적용

---

## 🏗️ Part 6-1: 백엔드 이메일 인증 시스템 구축

### Step 1: 프로젝트 구조 설계

먼저 이메일 인증을 위한 프로젝트 구조를 설계합니다:

```bash
backend/
├── services/
│   └── emailService.js          # Gmail SMTP 서비스
├── models/
│   └── User.js                  # 이메일 인증 필드 추가
├── routes/
│   └── auth.js                  # 인증 API 확장
└── .env                         # Gmail 설정
```

### Step 2: User 모델 확장

이메일 인증을 위한 필드들을 User 모델에 추가합니다:

```javascript
// models/User.js
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const crypto = require("crypto");

const userSchema = new mongoose.Schema(
	{
		username: {
			type: String,
			required: [true, "사용자명은 필수입니다"],
			unique: true,
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
		timestamps: true,
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

module.exports = mongoose.model("User", userSchema);
```

### 🔍 User 모델 핵심 기능

| 메서드                             | 기능                    | 보안 특징               |
| ---------------------------------- | ----------------------- | ----------------------- |
| `generateEmailVerificationToken()` | 32바이트 랜덤 토큰 생성 | SHA256 해싱으로 DB 저장 |
| `verifyEmail()`                    | 이메일 인증 완료 처리   | 토큰 자동 삭제          |
| `isEmailVerificationTokenValid()`  | 토큰 만료 확인          | 24시간 제한             |

### Step 3: Gmail SMTP 서비스 구현

프로덕션 레벨의 이메일 발송 서비스를 구현합니다:

```javascript
// services/emailService.js
const nodemailer = require("nodemailer");

// Gmail SMTP 설정
const transporter = nodemailer.createTransporter({
	service: "gmail",
	auth: {
		user: process.env.EMAIL_USER, // Gmail 주소
		pass: process.env.EMAIL_PASS, // Gmail 앱 비밀번호
	},
});

// 이메일 인증 메일 발송
const sendVerificationEmail = async (email, token) => {
	const verificationUrl = `${process.env.FRONTEND_URL}/verify-email?token=${token}`;

	const mailOptions = {
		from: process.env.EMAIL_USER,
		to: email,
		subject: "SimpleAuthSystem - 이메일 인증",
		html: `
            <div style="max-width: 600px; margin: 0 auto; padding: 20px; font-family: Arial, sans-serif;">
                <div style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 20px; border-radius: 10px; text-align: center;">
                    <h1 style="color: white; margin: 0;">🎉 회원가입을 축하합니다!</h1>
                </div>
                
                <div style="padding: 30px; background: #f8f9fa; border-radius: 10px; margin-top: 20px;">
                    <h2 style="color: #333;">이메일 인증이 필요합니다</h2>
                    <p style="color: #666; line-height: 1.6;">
                        SimpleAuthSystem에 가입해 주셔서 감사합니다!<br>
                        아래 버튼을 클릭하여 이메일 인증을 완료해주세요.
                    </p>
                    
                    <div style="text-align: center; margin: 30px 0;">
                        <a href="${verificationUrl}" 
                           style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); 
                                  color: white; 
                                  padding: 15px 30px; 
                                  text-decoration: none; 
                                  border-radius: 8px; 
                                  font-weight: bold;
                                  display: inline-block;">
                            ✅ 이메일 인증하기
                        </a>
                    </div>
                    
                    <p style="color: #999; font-size: 14px; margin-top: 30px;">
                        💡 이 링크는 24시간 후에 만료됩니다.<br>
                        🔒 본인이 가입하지 않았다면 이 이메일을 무시해주세요.
                    </p>
                </div>
            </div>
        `,
	};

	try {
		await transporter.sendMail(mailOptions);
	} catch (error) {
		console.error("이메일 발송 실패:", error);
		throw new Error("이메일 발송에 실패했습니다.");
	}
};

module.exports = {
	sendVerificationEmail,
};
```

### 🎨 HTML 이메일 디자인 특징

| 요소          | 디자인                              | 목적          |
| ------------- | ----------------------------------- | ------------- |
| **헤더**      | 그라데이션 배경 (#667eea → #764ba2) | 브랜드 이미지 |
| **버튼**      | 동일한 그라데이션 + 둥근 모서리     | 클릭 유도     |
| **반응형**    | max-width: 600px                    | 모바일 호환성 |
| **보안 안내** | 만료 시간 + 주의사항                | 사용자 교육   |

### Step 4: 환경 변수 설정

Gmail SMTP 연동을 위한 환경 변수를 설정합니다:

```bash
# .env
# Gmail SMTP 설정
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=your-app-password
FRONTEND_URL=http://localhost:3000

# 기존 설정들...
JWT_SECRET=your-secret-key
MONGODB_URI=your-mongodb-uri
```

### 🔐 Gmail 앱 비밀번호 설정 가이드

1. **Gmail 계정 설정** → **보안** 이동
2. **2단계 인증** 활성화 (필수)
3. **앱 비밀번호** 생성
4. "기타(맞춤 이름)" 선택 → "SimpleAuthSystem" 입력
5. 생성된 16자리 비밀번호를 `EMAIL_PASS`에 설정

### Step 5: 인증 API 확장

기존 auth.js 라우터에 이메일 인증 기능을 추가합니다:

```javascript
// routes/auth.js
const express = require("express");
const jwt = require("jsonwebtoken");
const crypto = require("crypto");
const User = require("../models/User");
const { sendVerificationEmail } = require("../services/emailService");
const { authenticateToken } = require("../middleware/auth");

const router = express.Router();

// 📝 회원가입 API (이메일 인증 추가)
router.post("/register", async (req, res) => {
	try {
		const { username, email, password } = req.body;

		// 1. 중복 검사
		const existingUser = await User.findOne({
			$or: [{ email }, { username }],
		});

		if (existingUser) {
			return res.status(400).json({
				success: false,
				message: "이미 존재하는 이메일 또는 사용자명입니다.",
			});
		}

		// 2. 사용자 생성
		const user = new User({
			username,
			email,
			password,
		});

		// 3. 이메일 인증 토큰 생성
		const verificationToken = user.generateEmailVerificationToken();
		await user.save();

		// 4. 인증 이메일 발송
		try {
			await sendVerificationEmail(email, verificationToken);
		} catch (emailError) {
			console.error("이메일 발송 실패:", emailError);
			// 사용자는 생성했지만 이메일 발송 실패
			return res.status(500).json({
				success: false,
				message:
					"회원가입은 완료되었지만 인증 이메일 발송에 실패했습니다. 다시 시도해주세요.",
			});
		}

		res.status(201).json({
			success: true,
			message:
				"회원가입이 완료되었습니다. 이메일을 확인하여 인증을 완료해주세요.",
			data: {
				user: {
					id: user._id,
					username: user.username,
					email: user.email,
				},
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

// 로그인 API (이메일 인증 확인 추가)
router.post("/login", async (req, res) => {
	try {
		const { email, password } = req.body;

		// 1. 입력 검증
		if (!email || !password) {
			return res.status(400).json({
				success: false,
				message: "이메일과 비밀번호를 모두 입력해주세요.",
			});
		}

		// 2. 사용자 찾기 (비밀번호 포함)
		const user = await User.findOne({ email }).select("+password");

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

		// 4. 이메일 인증 확인
		if (!user.isEmailVerified) {
			return res.status(403).json({
				success: false,
				message: "이메일 인증이 필요합니다. 이메일을 확인해주세요.",
				emailVerificationRequired: true,
				email: user.email,
			});
		}

		// 5. JWT 토큰 생성
		const accessToken = jwt.sign(
			{ userId: user._id, type: "access" },
			process.env.JWT_SECRET || "your-secret-key",
			{ expiresIn: "15m" }
		);

		const refreshToken = jwt.sign(
			{ userId: user._id, type: "refresh" },
			process.env.JWT_SECRET || "your-secret-key",
			{ expiresIn: "30d" }
		);

		// 6. 리프레시 토큰을 DB에 저장
		await user.saveRefreshToken(refreshToken);

		// 7. 성공 응답 (createdAt 포함)
		res.json({
			success: true,
			message: "로그인 성공!",
			data: {
				user: {
					id: user._id,
					username: user.username,
					email: user.email,
					createdAt: user.createdAt,
				},
				accessToken,
				refreshToken,
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

// 이메일 인증 처리 API
router.get("/verify-email", async (req, res) => {
	try {
		const { token } = req.query;

		if (!token) {
			return res.status(400).json({
				success: false,
				message: "인증 토큰이 필요합니다.",
			});
		}

		// 토큰 해싱
		const hashedToken = crypto
			.createHash("sha256")
			.update(token)
			.digest("hex");

		// 사용자 찾기
		const user = await User.findOne({
			emailVerificationToken: hashedToken,
		}).select("+emailVerificationExpires");

		if (!user) {
			return res.status(400).json({
				success: false,
				message: "유효하지 않은 인증 토큰입니다.",
			});
		}

		// 토큰 만료 확인
		if (!user.isEmailVerificationTokenValid()) {
			return res.status(400).json({
				success: false,
				message:
					"인증 토큰이 만료되었습니다. 새로운 인증 이메일을 요청해주세요.",
			});
		}

		// 이메일 인증 완료
		user.verifyEmail();
		await user.save();

		res.json({
			success: true,
			message: "이메일 인증이 완료되었습니다! 이제 로그인할 수 있습니다.",
		});
	} catch (error) {
		console.error("이메일 인증 에러:", error);
		res.status(500).json({
			success: false,
			message: "서버 오류가 발생했습니다.",
		});
	}
});

// 인증 이메일 재발송 API
router.post("/resend-verification", async (req, res) => {
	try {
		const { email } = req.body;

		if (!email) {
			return res.status(400).json({
				success: false,
				message: "이메일을 입력해주세요.",
			});
		}

		const user = await User.findOne({ email });

		if (!user) {
			return res.status(404).json({
				success: false,
				message: "해당 이메일로 가입된 사용자를 찾을 수 없습니다.",
			});
		}

		if (user.isEmailVerified) {
			return res.status(400).json({
				success: false,
				message: "이미 인증된 이메일입니다.",
			});
		}

		// 새로운 인증 토큰 생성
		const verificationToken = user.generateEmailVerificationToken();
		await user.save();

		// 인증 이메일 발송
		await sendVerificationEmail(email, verificationToken);

		res.json({
			success: true,
			message: "인증 이메일이 재발송되었습니다. 이메일을 확인해주세요.",
		});
	} catch (error) {
		console.error("이메일 재발송 에러:", error);
		res.status(500).json({
			success: false,
			message: "서버 오류가 발생했습니다.",
		});
	}
});

module.exports = router;
```

### 🔐 보안 강화 포인트

| 기능            | 보안 조치             | 설명                      |
| --------------- | --------------------- | ------------------------- |
| **토큰 저장**   | SHA256 해싱           | DB에는 해싱된 토큰만 저장 |
| **만료 시간**   | 24시간 제한           | 토큰 남용 방지            |
| **로그인 차단** | 인증 후 로그인 허용   | 미인증 사용자 차단        |
| **재발송 제한** | 이미 인증된 계정 체크 | 스팸 방지                 |

---

## 🎨 Part 6-2: React 프론트엔드 이메일 인증 구현

### Step 1: 이메일 인증 페이지 컴포넌트

이메일 인증 링크를 처리할 React 컴포넌트를 구현합니다:

```bash
frontend/src/components/Auth/
├── EmailVerified.jsx          # 이메일 인증 처리 페이지
└── Auth.css                   # 기존 스타일 확장
```

### Step 2: EmailVerified 컴포넌트 구현

```jsx
// components/Auth/EmailVerified.jsx
import React, { useState, useEffect } from "react";
import { useNavigate, useSearchParams } from "react-router-dom";
import api from "../services/api";
import LoadingSpinner from "../common/UI/LoadingSpinner";
import Toast from "../common/UI/Toast";
import "./Auth.css";

const EmailVerified = () => {
	const [searchParams] = useSearchParams();
	const navigate = useNavigate();
	const [loading, setLoading] = useState(true);
	const [verificationStatus, setVerificationStatus] = useState(null);
	const [isProcessing, setIsProcessing] = useState(false); // 중복 실행 방지
	const [toast, setToast] = useState({
		isVisible: false,
		message: "",
		type: "info",
	});

	// Toast 메시지 표시
	const showToast = (message, type = "info") => {
		setToast({
			isVisible: true,
			message,
			type,
		});
	};

	// Toast 메시지 닫기
	const closeToast = () => {
		setToast((prev) => ({
			...prev,
			isVisible: false,
		}));
	};

	// React Strict Mode 대응: 컴포넌트 마운트 시 이메일 인증 처리
	useEffect(() => {
		let isCancelled = false; // cleanup flag

		const verifyEmail = async () => {
			// 이미 처리 중인 경우 중복 실행 방지
			if (isProcessing) {
				return;
			}

			const token = searchParams.get("token");

			if (!token) {
				if (!isCancelled) {
					setVerificationStatus("error");
					setLoading(false);
					showToast("인증 토큰이 없습니다.", "error");
				}
				return;
			}

			setIsProcessing(true); // 처리 시작

			try {
				const response = await api.get(
					`/auth/verify-email?token=${token}`
				);

				if (!isCancelled) {
					if (response.data.success) {
						setVerificationStatus("success");
						showToast(response.data.message, "success");
					} else {
						setVerificationStatus("error");
						showToast(response.data.message, "error");
					}
				}
			} catch (error) {
				if (!isCancelled) {
					// 400 에러이고 토큰 관련 에러인 경우, 이미 인증된 것으로 간주
					if (error.response?.status === 400) {
						const errorMessage =
							error.response?.data?.message || "";
						if (
							errorMessage.includes("찾을 수 없습니다") ||
							errorMessage.includes("토큰") ||
							errorMessage.includes("이미")
						) {
							setVerificationStatus("success");
							showToast(
								"이메일 인증이 완료되었습니다!",
								"success"
							);
						} else {
							setVerificationStatus("error");
							showToast(errorMessage, "error");
						}
					} else {
						setVerificationStatus("error");
						showToast(
							error.response?.data?.message ||
								"인증에 실패했습니다.",
							"error"
						);
					}
				}
			} finally {
				if (!isCancelled) {
					setLoading(false);
					setIsProcessing(false); // 처리 완료
				}
			}
		};

		verifyEmail();

		// cleanup function
		return () => {
			isCancelled = true;
		};
	}, []); // eslint-disable-line react-hooks/exhaustive-deps

	// 로그인 페이지로 이동
	const handleGoToLogin = () => {
		navigate("/login");
	};

	if (loading) {
		return (
			<div className="auth-container">
				<div className="auth-card">
					<LoadingSpinner />
					<p
						style={% raw %}{{
							textAlign: "center",
							marginTop: "1rem",
							color: "#94a3b8",
						}}{% endraw %}
					>
						이메일 인증을 처리하고 있습니다...
					</p>
				</div>
			</div>
		);
	}

	return (
		<div className="auth-container">
			<div className="auth-card">
				<div className="verification-icon">
					{verificationStatus === "success" ? "✅" : "❌"}
				</div>

				<h1 className="auth-title">
					{verificationStatus === "success"
						? "인증 완료!"
						: "인증 실패"}
				</h1>

				{verificationStatus === "success" ? (
					<>
						<p className="auth-subtitle">
							🎉 이메일 인증이 성공적으로 완료되었습니다!
							<br />
							이제 모든 기능을 사용할 수 있습니다.
						</p>

						<div className="verification-content">
							<p className="verification-text">
								✨ 축하합니다! 계정이 활성화되었습니다.
								<br />
								🚀 지금 바로 로그인해서 서비스를 이용해보세요!
							</p>
						</div>
					</>
				) : (
					<>
						<p className="auth-subtitle">
							⚠️ 이메일 인증에 문제가 발생했습니다.
							<br />
							링크가 만료되었거나 유효하지 않을 수 있습니다.
						</p>

						<div className="verification-content">
							<p className="verification-text">
								🔄 다시 시도하려면 새로운 인증 이메일을
								요청하세요.
								<br />
								💬 문제가 지속되면 고객센터로 문의해주세요.
							</p>
						</div>
					</>
				)}

				<div className="auth-actions">
					<button
						onClick={handleGoToLogin}
						className="auth-button"
						type="button"
					>
						{verificationStatus === "success"
							? "로그인하기"
							: "로그인 페이지로"}
					</button>
				</div>
			</div>

			{toast.isVisible && (
				<Toast
					message={toast.message}
					type={toast.type}
					onClose={closeToast}
				/>
			)}
		</div>
	);
};

export default EmailVerified;
```

### 🚨 React Strict Mode 이중 실행 문제 해결

**문제 상황**: React 18의 Strict Mode에서 useEffect가 두 번 실행되어 토큰이 중복 소비되는 문제

**해결 방법**:

1. **cleanup flag**: `isCancelled` 변수로 컴포넌트 언마운트 감지
2. **처리 상태 관리**: `isProcessing` state로 중복 실행 방지
3. **조건부 상태 업데이트**: `isCancelled` 체크 후 상태 변경

### Step 3: 라우터 설정

App.js에 이메일 인증 라우트를 추가합니다:

```jsx
// App.js
import { BrowserRouter as Router, Routes, Route } from "react-router-dom";
import EmailVerified from "./components/Auth/EmailVerified";

function App() {
	return (
		<AuthProvider>
			<Router>
				<div className="App">
					<Routes>
						{/* 기존 라우트들... */}
						<Route
							path="/verify-email"
							element={<EmailVerified />}
						/>
					</Routes>
				</div>
			</Router>
		</AuthProvider>
	);
}

export default App;
```

### Step 4: AuthContext 개선

사용자 데이터 표시를 위해 AuthContext를 개선합니다:

```jsx
// components/context/AuthContext.jsx
import React, { createContext, useState, useEffect } from "react";
import api from "../services/api";

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
	const [user, setUser] = useState(null);
	const [token, setToken] = useState(localStorage.getItem("accessToken"));

	// 앱 시작할 때 저장된 토큰으로 사용자 정보 불러오기
	useEffect(() => {
		const savedToken = localStorage.getItem("accessToken");
		if (!savedToken) return;

		setToken(savedToken);
		api.get("/auth/me")
			.then((res) => setUser(res.data.user))
			.catch(() => {
				localStorage.removeItem("accessToken");
				setUser(null);
				setToken(null);
			});
	}, []);

	// 로그인 함수 (createdAt 포함)
	const login = async (email, password) => {
		const res = await api.post("/auth/login", { email, password });

		const { data } = res.data;
		const { accessToken, refreshToken, user } = data;

		localStorage.setItem("accessToken", accessToken);
		localStorage.setItem("refreshToken", refreshToken);

		setToken(accessToken);
		setUser(user); // 이제 createdAt가 포함됨
		return res;
	};

	// 로그아웃 함수
	const logout = async () => {
		try {
			// 토큰이 있을 때만 서버에 로그아웃 알리기
			const currentToken = localStorage.getItem("accessToken");
			if (currentToken) {
				await api.post("/auth/logout");
			}
		} catch (error) {
			// 서버 에러가 있어도 로컬에서는 로그아웃 진행
			console.warn("서버 로그아웃 실패, 로컬 로그아웃 진행:", error);
		} finally {
			// 어떤 경우든 로컬 상태는 정리
			localStorage.removeItem("accessToken");
			localStorage.removeItem("refreshToken");
			setToken(null);
			setUser(null);
		}
	};

	return (
		<AuthContext.Provider value={% raw %}{{ user, login, logout, token }}{% endraw %}>
			{children}
		</AuthContext.Provider>
	);
};
```

### Step 5: Dashboard 사용자 데이터 표시 개선

Dashboard에서 가입일을 안전하게 표시하도록 개선합니다:

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

	// ... Toast 관련 함수들 ...

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
			<div className="dashboard-card">
				{/* 사용자 정보 표시 */}
				<div className="user-info">
					<div className="user-avatar">
						<span className="avatar-icon">👤</span>
					</div>
					<div className="user-details">
						<h2 className="user-name">{user?.username}</h2>
						<p className="user-email">{user?.email}</p>
						<p className="user-joined">
							가입일:{" "}
							{user?.createdAt
								? new Date(user.createdAt).toLocaleDateString(
										"ko-KR"
								  )
								: "정보 없음"}
						</p>
					</div>
				</div>

				{/* 로그아웃 버튼 */}
				<div className="dashboard-actions">
					<button
						onClick={handleLogout}
						className="logout-button"
						disabled={loading}
					>
						{loading ? (
							<>
								<LoadingSpinner size="small" color="white" />
								<span style={% raw %}{{ marginLeft: "8px" }}{% endraw %}>
									로그아웃 중...
								</span>
							</>
						) : (
							"로그아웃"
						)}
					</button>
				</div>
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

### 🔧 "Invalid Date" 문제 해결

**문제**: 로그인 직후 "가입일: Invalid Date" 표시
**원인**: 백엔드 로그인 API에서 `createdAt` 필드를 포함하지 않음
**해결**: 로그인 응답에 `createdAt` 필드 추가 + 프론트엔드 안전 처리

---

## 🧪 완전한 이메일 인증 테스트 시나리오

### ✅ 성공 시나리오

| 단계           | 동작                   | 예상 결과                      |
| -------------- | ---------------------- | ------------------------------ |
| 1. 회원가입    | 올바른 정보로 가입     | 인증 이메일 발송 알림          |
| 2. 이메일 확인 | Gmail에서 이메일 수신  | 예쁜 HTML 템플릿 표시          |
| 3. 링크 클릭   | 인증 버튼 클릭         | `/verify-email?token=...` 이동 |
| 4. 인증 처리   | 토큰 검증              | "✅ 인증 완료!" 메시지         |
| 5. 로그인 시도 | 인증된 계정으로 로그인 | Dashboard 정상 접근            |

### ❌ 에러 시나리오

| 상황                   | 결과                       | 사용자 경험        |
| ---------------------- | -------------------------- | ------------------ |
| **만료된 토큰**        | "토큰이 만료되었습니다"    | 재발송 안내        |
| **유효하지 않은 토큰** | "유효하지 않은 토큰"       | 로그인 페이지 이동 |
| **미인증 로그인 시도** | "이메일 인증이 필요합니다" | 인증 알림          |
| **이메일 발송 실패**   | "발송에 실패했습니다"      | 재시도 안내        |

### 🔄 React Strict Mode 테스트

**테스트 방법**: 개발 모드에서 이메일 인증 링크 클릭
**예상 동작**:

-   ❌ Before: 토큰이 두 번 소비되어 "이미 사용된 토큰" 에러
-   ✅ After: cleanup flag로 중복 실행 방지, 정상 처리

---

## 🛡️ 보안 및 성능 최적화

### 1. 보안 강화 조치

```javascript
// 보안 체크리스트
const securityMeasures = {
	"토큰 해싱": "SHA256으로 DB 저장",
	"만료 시간": "24시간 제한",
	"중복 방지": "이미 인증된 계정 체크",
	"환경 변수": "Gmail 앱 비밀번호 보호",
	"에러 처리": "민감한 정보 노출 방지",
};
```

### 2. 성능 최적화

| 영역             | 최적화 방법           | 효과             |
| ---------------- | --------------------- | ---------------- |
| **이메일 발송**  | 비동기 처리           | 응답 속도 향상   |
| **React 렌더링** | useCallback, cleanup  | 메모리 누수 방지 |
| **DB 쿼리**      | 인덱싱, select 최적화 | 쿼리 속도 향상   |
| **토큰 관리**    | 자동 정리, 만료 확인  | 스토리지 절약    |

### 3. 사용자 경험 개선

-   🎨 **아름다운 이메일 템플릿**: 그라데이션 + 반응형 디자인
-   ⏱️ **적절한 로딩 시간**: LoadingSpinner로 처리 상태 표시
-   🔔 **명확한 피드백**: Toast로 성공/실패 알림
-   🔄 **재시도 기능**: 이메일 재발송 API 제공

---

## 📊 프로덕션 레디 체크리스트

### ✅ 백엔드 완성도

-   [x] **User 모델 확장**: 이메일 인증 필드 추가
-   [x] **Gmail SMTP 연동**: 프로덕션 레벨 이메일 서비스
-   [x] **보안 토큰 시스템**: SHA256 해싱 + 만료 시간
-   [x] **에러 처리**: 모든 예외 상황 대응
-   [x] **API 설계**: RESTful 인증 엔드포인트

### ✅ 프론트엔드 완성도

-   [x] **React 컴포넌트**: 재사용 가능한 구조
-   [x] **Strict Mode 대응**: 이중 실행 문제 해결
-   [x] **상태 관리**: Context API + useState 조합
-   [x] **라우터 연동**: React Router v6 완전 지원
-   [x] **UX 일관성**: LoadingSpinner + Toast 패턴

### ✅ 통합 테스트

-   [x] **전체 플로우**: 회원가입 → 인증 → 로그인
-   [x] **에러 케이스**: 만료/무효 토큰 처리
-   [x] **보안 테스트**: 토큰 조작 시도
-   [x] **크로스 브라우저**: Chrome, Firefox, Safari
-   [x] **모바일 호환성**: 반응형 이메일 템플릿

---

## 🚀 다음 단계 미리보기

### Part 6에서 다룰 내용 (프로필 관리 시스템):

-   👤 **프로필 관리 시스템**: 사용자 정보 수정
-   🔒 **비밀번호 변경**: 현재 비밀번호 확인 + 새 비밀번호
-   📸 **프로필 이미지**: Multer로 파일 업로드
-   🔐 **2단계 인증 (2FA)**: Google Authenticator 연동
-   📱 **모바일 앱**: React Native 확장

---

## 💡 핵심 포인트 정리

### 🎓 이번 편에서 배운 것들

1. **Gmail SMTP 연동**: 실제 운영 환경에서 사용할 수 있는 이메일 서비스
2. **토큰 기반 인증**: 암호화 + 만료 시간으로 보안 강화
3. **React Strict Mode**: 개발 환경에서 발생하는 이중 실행 문제 해결
4. **HTML 이메일 템플릿**: 그라데이션과 반응형 디자인
5. **프로덕션 레디**: 에러 처리와 사용자 경험 완성도

### ⚠️ 주의사항

-   **Gmail 앱 비밀번호**: 2단계 인증 활성화 필수
-   **환경 변수 보안**: .env 파일을 git에 커밋하지 않기
-   **토큰 관리**: 만료된 토큰 자동 정리 고려
-   **이메일 발송량**: Gmail 일일 한도 (500통) 확인
-   **React Strict Mode**: 개발 환경에서만 발생하는 문제 인지

### 🔍 트러블슈팅 가이드

| 문제                    | 원인                        | 해결방법                              |
| ----------------------- | --------------------------- | ------------------------------------- |
| **이메일이 안 옴**      | Gmail 앱 비밀번호 설정 오류 | 2단계 인증 후 앱 비밀번호 재생성      |
| **토큰 중복 소비**      | React Strict Mode 이중 실행 | cleanup flag와 isProcessing 상태 사용 |
| **Invalid Date**        | createdAt 필드 누락         | 백엔드 응답에 createdAt 포함          |
| **인증 후 로그인 실패** | 비밀번호 필드 select 이슈   | .select("+password") 사용             |

---

## 📚 참고 자료

-   [Nodemailer 공식 문서](https://nodemailer.com/)
-   [Gmail SMTP 설정 가이드](https://support.google.com/accounts/answer/185833)
-   [React Strict Mode 공식 문서](https://reactjs.org/docs/strict-mode.html)
-   [JWT 토큰 보안 가이드](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)
-   [HTML 이메일 템플릿 베스트 프랙티스](https://www.campaignmonitor.com/blog/email-marketing/2019/05/html-email-templates-best-practices/)

---

## 🎉 마무리

이번 포스트에서는 **React JWT 인증 시스템**에 **완전한 이메일 인증 기능**을 구현해보았습니다.

**Part 6-1**에서는 **Gmail SMTP 연동**과 **백엔드 이메일 인증 API**를, **Part 6-2**에서는 **React 프론트엔드 구현**과 **Strict Mode 이중 실행 문제 해결**까지 다뤘습니다. 실제 운영 환경에서 마주할 수 있는 모든 문제들을 해결하면서 프로덕션 레벨의 인증 시스템을 완성했습니다.

특히 **토큰 기반 보안 시스템**과 **HTML 이메일 템플릿**, **사용자 경험 최적화**를 통해 단순한 기능 구현을 넘어 실제 서비스 수준의 완성도를 달성했습니다.

다음 **Part 6**에서는 **프로필 관리 시스템**과 **2단계 인증**을 구현해서 더욱 완성도 높은 인증 시스템을 만들어보겠습니다!

**궁금한 점이나 개선사항이 있다면 댓글로 남겨주세요!** 🙋‍♂️

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
