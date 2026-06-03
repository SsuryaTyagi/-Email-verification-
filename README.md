# Email Verification Module

A plug-and-play email verification system for any Node.js + Express + MongoDB project.

---

## What's Included

| File | Purpose |
|------|---------|
| `utils/jwt.utils.js` | Generate & verify email verification token |
| `services/email.service.js` | Send verification email via Nodemailer |
| `controllers/auth.controller.js` | Register, Login, Logout, VerifyEmail logic |
| `models/user.js` | User schema with `verified` field |

---

## Prerequisites

```bash
npm install nodemailer jsonwebtoken mongoose bcrypt
```

---

## Step 1 — Environment Variables

Add these to your `.env` file:

```env
JWT_TOKEN_SECRET=your_jwt_secret_here
SMTP_USER=your_gmail@gmail.com
SMTP_PASS=your_gmail_app_password
CLIENT_URL=http://localhost:5173
```

> **Gmail App Password** — Go to Google Account → Security → 2-Step Verification → App Passwords → Generate one.

---

## Step 2 — User Model

Your `user.js` model must have a `verified` field:

```js
const userSchema = new mongoose.Schema({
  name:     { type: String, required: true },
  email:    { type: String, required: true, unique: true },
  password: { type: String, required: true },
  verified: { type: Boolean, default: false },  // ← required
});
```

---

## Step 3 — JWT Utils (`utils/jwt.utils.js`)

```js
const jwt = require("jsonwebtoken");

const generateVerificationToken = (email) => {
  return jwt.sign({ email }, process.env.JWT_TOKEN_SECRET, { expiresIn: "1d" });
};

const verifyVerificationToken = (token) => {
  return jwt.verify(token, process.env.JWT_TOKEN_SECRET);
};

module.exports = { generateVerificationToken, verifyVerificationToken };
```

---

## Step 4 — Email Service (`services/email.service.js`)

```js
const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  host: "smtp.gmail.com",
  port: 465,
  secure: true,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});

const sendVerificationEmail = async (email, name, token) => {
  const verifyURL = `${process.env.CLIENT_URL}/verify-email/${token}`;

  await transporter.sendMail({
    from: `"Your App" <${process.env.SMTP_USER}>`,
    to: email,
    subject: "Verify Your Email",
    html: `
      <div style="font-family:Arial; max-width:500px; margin:auto; padding:30px;">
        <h2>Hello ${name}!</h2>
        <p>Click the button below to verify your email. Link expires in 24 hours.</p>
        <a href="${verifyURL}"
           style="background:#7c3aed; color:white; padding:12px 28px;
                  border-radius:8px; text-decoration:none; font-weight:bold;">
          Verify Email
        </a>
        <p style="color:#999; margin-top:20px; font-size:13px;">
          If you didn't register, ignore this email.
        </p>
      </div>
    `,
  });
};

module.exports = { sendVerificationEmail };
```

> To change the app name or color — edit `"Your App"` and `background:#7c3aed`.

---

## Step 5 — Controller Logic

### Register
```js
const { generateVerificationToken } = require("../utils/jwt.utils");
const { sendVerificationEmail }     = require("../services/email.service");

const RegisterController = async (req, res) => {
  const { name, email, password } = req.body;

  const exists = await UserModel.findOne({ email });
  if (exists) return res.status(400).json({ message: "User already exists" });

  await UserModel.create({ name, email, password, verified: false });

  const token = generateVerificationToken(email);
  await sendVerificationEmail(email, name, token);

  return res.status(201).json({ message: "Check your email to verify your account." });
};
```

### Verify Email
```js
const { verifyVerificationToken } = require("../utils/jwt.utils");

const VerifyEmailController = async (req, res) => {
  try {
    const { token } = req.params;
    const decoded   = verifyVerificationToken(token);

    const user = await UserModel.findOne({ email: decoded.email });
    if (!user)         return res.status(400).json({ message: "Invalid link" });
    if (user.verified) return res.status(400).json({ message: "Already verified" });

    user.verified = true;
    await user.save();

    return res.status(200).json({ message: "Email verified successfully" });

  } catch (err) {
    const msg = err.name === "TokenExpiredError" ? "Link expired" : "Invalid link";
    return res.status(400).json({ message: msg });
  }
};
```

### Login — block unverified users
```js
if (!user.verified) {
  return res.status(403).json({ message: "Please verify your email before logging in." });
}
```

---

## Step 6 — Routes (`routes/auth.routes.js`)

```js
const express = require("express");
const router  = express.Router();
const { RegisterController, VerifyEmailController, LoginController } = require("../controllers/auth.controller");

router.post("/register",              RegisterController);
router.get("/verify-email/:token",    VerifyEmailController);
router.post("/login",                 LoginController);

module.exports = router;
```

---

## Step 7 — Frontend (React)

Create a page at `/verify-email/:token`:

```jsx
import { useEffect, useState } from "react";
import { useParams, useNavigate } from "react-router-dom";
import axios from "axios";

export default function VerifyEmailPage() {
  const { token }   = useParams();
  const navigate    = useNavigate();
  const [status, setStatus] = useState("verifying");

  useEffect(() => {
    axios.get(`/verify-email/${token}`)
      .then(() => { setStatus("success"); setTimeout(() => navigate("/login"), 2000); })
      .catch((err) => setStatus(err.response?.data?.message || "Invalid link"));
  }, [token]);

  return (
    <div style={{ textAlign: "center", marginTop: "20vh" }}>
      {status === "verifying" && <p>Verifying your email...</p>}
      {status === "success"   && <p>✅ Email verified! Redirecting to login...</p>}
      {status !== "verifying" && status !== "success" && <p>❌ {status}</p>}
    </div>
  );
}
```

Add route in `app.routes.js`:
```jsx
{ path: "/verify-email/:token", element: <VerifyEmailPage /> }
```

---

## Flow Summary

```
User registers
      ↓
Account created (verified: false)
      ↓
Verification email sent
      ↓
User clicks link in email
      ↓
GET /verify-email/:token
      ↓
verified: true saved in DB
      ↓
User can now login
```

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid login` on Gmail | Wrong SMTP password | Use App Password, not account password |
| `Link expired` | Token older than 24h | Re-register or add resend endpoint |
| `Already verified` | Link clicked twice | Safe to ignore |
| Email not received | Wrong `SMTP_USER` in `.env` | Double-check `.env` values |

---

## Reusing in a New Project

1. Copy `utils/jwt.utils.js`, `services/email.service.js`
2. Add `verified: false` to your user schema
3. Add the 3 controller functions
4. Add routes
5. Add frontend `/verify-email/:token` page
6. Set `.env` variables

That's it.
