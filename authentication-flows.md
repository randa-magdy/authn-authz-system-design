# **5. Authentication & Password Reset Flows**

The platform supports multiple authentication methods and password management.
Below are the detailed **server-side flow steps** and **sequence diagrams** for each flow.

---

## **5.1 Email + Password Authentication (Admin Login)**

### **Flow Steps**

#### 🔹 Request Validation

* Validate email format and enforce password policy (min length, complexity).
* Sanitize input to prevent SQL injection or XSS.
* Apply rate limiting per IP and per email.

#### 🔹 Credential Verification

* Query database for admin by email.
* Compare provided password with stored hash (bcrypt/argon2).
* Check if account is active and not locked.
* If login fails → increment `failedAttempts`.
* Lock account if attempts exceed maximum threshold.

#### 🔹 Token Generation

* On success → reset `failedAttempts`.
* Generate **JWT access token** (short-lived: 15–30 min).
* Generate **JWT refresh token** (long-lived: 7–30 days).
* Sign tokens with server’s **private key**.
* Set tokens as **Secure, HttpOnly, SameSite=strict cookies**.

#### 🔹 Response Handling

* **Success** → return user profile + set cookies.
* **Failure** → log failed attempt, return generic error (`401 Unauthorized` or `423 Locked` if account locked).

### **Sequence Diagram**

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser/App
    participant S as Server
    participant DB as Database

    U->>B: Enter email & password
    B->>S: POST /auth/login {email, password}
    
    S->>S: Validate input & apply rate limiting
    S->>DB: SELECT user WHERE email = ?
    DB->>S: Return user record (with password hash & failedAttempts)
    
    S->>S: Compare password with hash
    
    alt Authentication Success
        S->>S: Reset failedAttempts = 0
        S->>S: Generate JWT access token (15–30 min)
        S->>S: Generate JWT refresh token (7–30 days)
        S->>S: Sign tokens with private key
        S->>B: 200 OK + Set-Cookie {AccessToken, RefreshToken}
        Note right of B: Cookies are Secure, HttpOnly, SameSite=strict
        B->>B: Store tokens in client side (cookies / memory)
        S->>B: Return user profile info
        B->>U: Login successful
    else Authentication Failure
        S->>S: Increment failedAttempts
        alt failedAttempts < Max (e.g., 5)
            S->>B: 401 Unauthorized
            B->>U: Invalid credentials
        else failedAttempts >= Max
            S->>DB: Update user status = "Locked"
            S->>B: 423 Locked (Account Locked)
            B->>U: Account locked, contact support
        end
    end
```

---

## **5.2 Email + Verification Code Authentication**

### **Flow Steps**

#### 🔹 Email Request Phase

* Validate email format.
* Check if email exists in the system; if not, create a temporary/pending user record.
* Generate a **6-digit one-time code** with short expiry (5–10 minutes).
* Store code in **cache/database** with: `{email, code, expiry, attempts=0}`.
* Send code via email service (asynchronous, non-blocking).
* Return success response: *“Code sent to email”*.

#### 🔹 Code Verification Phase

* Validate code format and apply rate limiting (per IP/email).
* Retrieve stored code from cache/database.
* Check if code exists, is not expired, and attempts < max (e.g., 5).
* Compare provided code with stored code.
* If match → mark code as used (delete/expire).
* If mismatch or expired → increment attempts, block after max failures.

#### 🔹 Authentication Success

* Create or get user record in the database.
* Generate **JWT access token** (15–30 minutes).
* Generate **JWT refresh token** (7–30 days).
* Sign tokens with the server’s **private key**.
* Set tokens as **Secure, HttpOnly, SameSite=strict cookies**.
* Clean up verification code (remove from cache/DB).
* Return user profile + authentication tokens.

### **Sequence Diagram**

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser/App
    participant S as Server
    participant DB as Database
    participant ES as Email Service
    participant Cache as Redis Cache

    Note over U,Cache: Phase 1: Request Verification Code
    U->>B: Enter email address
    B->>S: POST /auth/send-code {email}
    
    S->>S: Validate email format
    S->>S: Generate 6-digit code + expiry
    S->>Cache: Store {email, code, expiry, attempts=0}
    Cache->>S: stored
    
    S->>ES: Send verification email (async)
    ES->>U: Email with verification code
    
    S->>B: 200 OK "Code sent to email"
    B->>U: "Check your email for code"

    Note over U,Cache: Phase 2: Verify Code
    U->>B: Enter verification code
    B->>S: POST /auth/verify-code {email, code}
    
    S->>S: Validate input & rate limit
    S->>Cache: Get stored code for email
    Cache->>S: {code, expiry, attempts}
    
    S->>S: Compare codes & check expiry
    
    alt Verification Success
        S->>Cache: Mark code as used/delete
        S->>DB: Create or get user record
        DB->>S: User record saved
        S->>S: Generate JWT access token (15–30 min)
        S->>S: Generate JWT refresh token (7–30 days)
        S->>S: Sign tokens with private key
        S->>B: 200 OK + Set-Cookie {AccessToken, RefreshToken}
        Note right of B: Cookies = Secure, HttpOnly, SameSite=strict
        B->>B: Store tokens in client side (cookies / memory)
        B->>U: Login successful
    else Invalid Code or Expired
        S->>Cache: Increment failed attempts
        alt Attempts < Max
            S->>B: 401 Unauthorized
            B->>U: Invalid or expired code
        else Too Many Attempts
            S->>Cache: Invalidate code / block
            S->>B: 423 Locked
            B->>U: Too many failed attempts
        end
    end
```

---

## **5.3 Social Login (OAuth 2.0) Authentication**

*(Google / Facebook, OpenID Connect compatible)*

### **Flow Steps**

1. **Client Side – Start OAuth**

   * User clicks **“Login with Google”**.
   * Frontend calls **GET /auth/oauth/google**.
   * Backend generates authorization URL & redirects browser.

2. **User → Identity Provider → Back to Server**

   * User logs in & approves consent.
   * Provider redirects browser to backend callback URL with `code` & `state`.

3. **Server Side – Exchange Code for Tokens**

   * Validate `state`.
   * POST to provider token endpoint with code & client credentials.
   * Receive `access_token`, `refresh_token`, `id_token`.

4. **Server Side – Get User Profile**

   * Use `access_token` to fetch user info.

5. **Server Side – Create/Find User**

   * Check DB for existing user by `sub` or `email`.
   * Create new user if not exists.

6. **Server Side – Generate Internal Tokens**

   * Generate JWT access + refresh tokens.
   * Set as **Secure, HttpOnly cookies**.
   * Redirect user to frontend.

### **Sequence Diagram**

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser/App
    participant S as Server
    participant IDP as Identity Provider
    participant DB as Database
    participant Cache as Redis Cache

    Note over U,Cache: Phase 1: OAuth Initiation
    U->>B: Click "Sign in with Google/Facebook"
    B->>S: GET /auth/oauth/google
    
    S->>S: Generate state parameter
    S->>Cache: Store state for CSRF protection
    S->>S: Build authorization URL
    S->>B: Redirect to IDP authorization URL
    
    B->>IDP: GET /oauth/authorize?client_id=...&state=...
    IDP->>U: Show consent/login page
    U->>IDP: Grant permissions
    
    Note over U,Cache: Phase 2: Authorization Code Exchange
    IDP->>B: Redirect to callback URL + code & state
    B->>S: GET /auth/oauth/callback?code=...&state=...
    
    S->>Cache: Validate state parameter
    Cache->>S: State validation result
    
    S->>IDP: POST /oauth/token {code, client_secret}
    IDP->>S: {access_token, id_token, refresh_token}
    
    S->>S: Validate ID token signature & claims
    
    Note over U,Cache: Phase 3: User Profile & Account
    S->>IDP: GET /userinfo (with access_token)
    IDP->>S: User profile data
    
    S->>DB: SELECT user WHERE provider_id = ${sub} OR email = ${email}
    
    alt User Exists
        DB->>S: Existing user record
        S->>DB: UPDATE user SET last_login = NOW()
    else New User
        DB->>S: No user found
        S->>DB: INSERT new user with provider data
        DB->>S: New user record
    end
    
    Note over S,B: Phase 4: Token Issuance
    S->>S: Generate JWT access token (15–30 min)
    S->>S: Generate JWT refresh token (7–30 days)
    S->>S: Sign tokens with private key
    S->>B: Redirect to app + Set-Cookie {AccessToken, RefreshToken}
    Note right of B: Cookies = Secure, HttpOnly, SameSite=strict
    B->>U: Login successful, redirect to dashboard
```

---

## **5.4 Password Reset Flow (with Email Verification)**

### **Flow Steps**

#### 🔹 Request Reset Link

1. Validate submitted email.
2. Query DB; if no user exists → return generic success.
3. If exists → generate secure random reset token + expiry.
4. Store hashed token + expiry in DB/cache.
5. Send reset link to user:

```
https://my-app.com/reset-password?token=<raw_token>&email=<email>
```

#### 🔹 Verify Reset Token

1. User clicks link → frontend shows reset form.
2. Backend receives `{email, token}`.
3. Validate token format & lookup stored token.
4. Check expiry & one-time-use flag.

#### 🔹 Set New Password

1. User submits new password (validate strength).
2. Compare token (hash match).
3. If valid → hash new password, update DB, invalidate token, invalidate old sessions.
4. Response: success → `"Password updated successfully"`, failure → `"Invalid or expired reset link"`.

### **Sequence Diagram**

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser/App
    participant S as Server
    participant DB as Database
    participant ES as Email Service
    participant Cache as Redis Cache

    Note over U,Cache: Phase 1: Request Reset Link
    U->>B: Enter email in "Forgot Password"
    B->>S: POST /auth/request-reset {email}
    
    S->>S: Validate email & rate limit
    S->>DB: SELECT user WHERE email=?
    DB->>S: User found / not found
    
    alt User Exists
        S->>S: Generate secure reset token + expiry
        S->>Cache: Store {hashed_token, expiry, email}
        Cache->>S: Token stored
        S->>ES: Send reset email with link
        ES->>U: "Reset Password" email
    end
    S->>B: 200 OK "If account exists, reset link sent"
    B->>U: Show generic success message

    Note over U,Cache: Phase 2: Verify Reset Token
    U->>B: Click reset link
    B->>S: GET /auth/reset-password?email=...&token=...
    S->>Cache: Validate token & expiry
    Cache->>S: Token valid/invalid

    Note over U,Cache: Phase 3: Set New Password
    U->>B: Submit new password
    B->>S: POST /auth/reset-password {email, token, newPassword}
    
    S->>Cache: Fetch stored token for email
    Cache->>S: {hashed_token, expiry}
    S->>S: Compare token & expiry
    
    alt Valid Token
        S->>S: Hash new password
        S->>DB: UPDATE user SET password_hash=newHash
        DB->>S: Success
        S->>Cache: Delete reset token
        S->>B: 200 OK "Password updated"
        B->>U: Show success
    else Invalid Token
        S->>B: 400 Bad Request "Invalid or expired link"
        B->>U: Show error
    end
```

