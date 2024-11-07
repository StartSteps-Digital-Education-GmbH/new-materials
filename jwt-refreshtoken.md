To integrate refresh tokens with your existing authentication setup, here’s what we’ll add:

1. **Generate and Store Refresh Tokens:** When a user logs in, generate both access and refresh tokens, save the refresh token to the database, and return it to the client.
2. **Refresh Token Endpoint:** Add a `/refresh-token` route to generate a new access token using the refresh token.
3. **Logout Endpoint:** Implement a logout route that clears the refresh token from the database.

### Updated Code

We’ll modify your existing `signup` and `signin` functions and add `refreshToken` and `logout` functions.

---

#### 1. **Environment Variables**
Add the following in `.env`:

```plaintext
JWT_SECRET=your_jwt_secret
JWT_REFRESH_SECRET=your_refresh_token_secret
ACCESS_TOKEN_EXPIRATION=1h
REFRESH_TOKEN_EXPIRATION=7d
```

---

#### 2. **Controller Modifications (controllers/authController.ts)**

```typescript
import { Request, Response } from 'express';
import bcrypt from 'bcryptjs';
import User from '../models/userModel.js';
import jwt from 'jsonwebtoken';

// Helper functions to generate tokens
const generateAccessToken = (userId: string) => {
    return jwt.sign({ id: userId }, process.env.JWT_SECRET!, { expiresIn: process.env.ACCESS_TOKEN_EXPIRATION });
};

const generateRefreshToken = (userId: string) => {
    return jwt.sign({ id: userId }, process.env.JWT_REFRESH_SECRET!, { expiresIn: process.env.REFRESH_TOKEN_EXPIRATION });
};

const signup = async (req: Request, res: Response) => {
    const { name, email, password } = req.body;
    try {
        const hashedPassword = await bcrypt.hash(password, 10);
        const newUser = new User({ name, email, password: hashedPassword });
        await newUser.save();
        res.status(201).send({ message: "User created" });
    } catch (error) {
        res.status(500).send(error);
    }
};

const signin = async (req: Request, res: Response) => {
    const { email, password } = req.body;
    try {
        const user = await User.findOne({ email });
        if (!user) return res.status(404).send('User Not Found!');

        const isMatch = await bcrypt.compare(password, user.password);
        if (!isMatch) return res.status(401).send('Invalid Password');

        const accessToken = generateAccessToken(user._id);
        const refreshToken = generateRefreshToken(user._id);

        // Save refresh token in the database
        user.refreshToken = refreshToken;
        await user.save();

        res.status(200).send({ accessToken, refreshToken });
    } catch (error) {
        res.status(500).send(error);
    }
};

// Refresh Token endpoint
const refreshToken = async (req: Request, res: Response) => {
    const { refreshToken } = req.body;
    if (!refreshToken) return res.status(401).send('Refresh Token required');

    try {
        const user = await User.findOne({ refreshToken });
        if (!user) return res.status(403).send('Invalid Refresh Token');

        jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!, (err, decoded) => {
            if (err) return res.status(403).send('Invalid Refresh Token');

            const accessToken = generateAccessToken(user._id);
            res.status(200).send({ accessToken });
        });
    } catch (error) {
        res.status(500).send(error);
    }
};

// Logout endpoint
const logout = async (req: Request, res: Response) => {
    const { refreshToken } = req.body;
    if (!refreshToken) return res.status(400).send('Refresh Token required');

    try {
        const user = await User.findOneAndUpdate({ refreshToken }, { $unset: { refreshToken: "" } });
        if (!user) return res.status(403).send('Invalid Refresh Token');

        res.status(204).send(); // Success, no content
    } catch (error) {
        res.status(500).send(error);
    }
};

export default { signup, signin, refreshToken, logout };
```

---

#### 3. **Middleware Modifications (middlewares/auth.ts)**

Update the `auth` middleware to require only an access token. When it expires, the client should use the refresh token to get a new access token.

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const auth = (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers['authorization']?.split(' ')[1];
    if (!token) return res.status(401).send('Access Token required');

    try {
        jwt.verify(token, process.env.JWT_SECRET!, (err, decoded) => {
            if (err) return res.status(403).send('Invalid Access Token');
            req.user = decoded;
            next();
        });
    } catch (error) {
        res.status(500).send(error);
    }
};

export default auth;
```

---

#### 4. **User Model Update (models/userModel.js)**

Add a `refreshToken` field to store the refresh token for each user.

```typescript
import mongoose from 'mongoose';

const userSchema = new mongoose.Schema({
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    refreshToken: { type: String }, // Store refresh token
});

export default mongoose.model('User', userSchema);
```

---

#### 5. **Routes Update**

In your main routes file, add routes for `refresh-token` and `logout`.

```typescript
import express from 'express';
import authController from '../controllers/authController.js';

const router = express.Router();

router.post('/signup', authController.signup);
router.post('/signin', authController.signin);
router.post('/refresh-token', authController.refreshToken); // New route for refreshing access token
router.post('/logout', authController.logout); // New route for logout

export default router;
```

---

#### 6. **Client-side Handling of Tokens**

The client should store the `accessToken` and `refreshToken`, and when the `accessToken` expires, it can call the `/refresh-token` endpoint to get a new one.

```javascript
async function refreshAccessToken() {
    try {
        const response = await fetch('/refresh-token', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refreshToken: localStorage.getItem('refreshToken') })
        });
        const data = await response.json();
        if (data.accessToken) {
            localStorage.setItem('accessToken', data.accessToken);
        }
    } catch (error) {
        console.error('Failed to refresh access token:', error);
    }
}
```

---

### Summary
1. **`signup` and `signin`**: Generate both `accessToken` and `refreshToken` during login.
2. **`refreshToken`** endpoint: Uses the refresh token to generate a new access token.
3. **`logout`** endpoint: Removes the refresh token from the database.
4. **Middleware**: Auth middleware only checks for the access token, and expired tokens can be refreshed with `/refresh-token`. 

This setup enables a more secure token-based authentication with the ability to refresh tokens, maintaining the user’s session securely.