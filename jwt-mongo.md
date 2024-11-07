### JWT Authentication in MongoDB database and Serverless Microservices

#### Project Structure for Authentication Setup
Ensure you have the following structure in your project:

```
|-- /models
|   |-- userModel.ts
/user-service
|-- /src
|   |-- /controllers
|   |   |-- authController.ts
|   |-- /routes
|   |   |-- userRoutes.ts
|   |   |-- authRoutes.ts   <-- new route for authentication
|   |-- /middleware
|   |   |-- authMiddleware.ts <-- new middleware for authentication
|   |-- index.ts (or server.ts)
|-- package.json
|-- tsconfig.json
```

#### Step 1: Install Required Packages
Run the following commands in the root of your project:

```bash
npm install express mongoose jsonwebtoken bcryptjs dotenv
npm install --save-dev typescript @types/node @types/express @types/mongoose @types/jsonwebtoken @types/bcryptjs
```

#### Step 2: Define Environment Variables
In your `.env` file, add:

```
MONGODB_URI=your_mongo_database_uri
JWT_SECRET=your_secret_key
```

#### Step 3: Update User Model
Update `userModel.ts` to include `password` for authentication:

```typescript
import mongoose, { Schema, Document } from "mongoose";

interface IUser extends Document {
    name: string;
    email: string;
    password: string;  // Include password for authentication
}

const userSchema = new Schema({
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },  // Password field
    createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model<IUser>('User', userSchema);
export default User;
export { IUser };
```

#### Step 4: Create Authentication Controller
Create `authController.ts` in the `/controllers` folder for handling signup and signin:

```typescript
import { Request, Response } from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import User from '../models/userModel';

const JWT_SECRET = process.env.JWT_SECRET || 'default_secret';

export const signUp = async (req: Request, res: Response) => {
  const { name, email, password } = req.body;

  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = new User({ name, email, password: hashedPassword });
    await newUser.save();
    res.status(201).json({ message: 'User created successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Error creating user', error });
  }
};

export const signIn = async (req: Request, res: Response) => {
  const { email, password } = req.body;

  try {
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign({ id: user._id }, JWT_SECRET, { expiresIn: '1h' });
    res.json({ token });
  } catch (error) {
    res.status(500).json({ message: 'Error signing in', error });
  }
};
```

#### Step 5: Define Authentication Routes
Create `authRoutes.ts` in `/routes`:

```typescript
import { Router } from 'express';
import { signUp, signIn } from '../controllers/authController';

const router = Router();

router.post('/signup', signUp);
router.post('/signin', signIn);

export default router;
```

#### Step 6: Add Middleware for Authorization
Create `authMiddleware.ts` in `/middleware` to validate JWT:

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'default_secret';

export const authenticate = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers['authorization']?.split(' ')[1];
  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }

  jwt.verify(token, JWT_SECRET, (err, decoded) => {
    if (err) {
      return res.status(401).json({ message: 'Unauthorized' });
    }
    req.userId = (decoded as any).id;  // Attach user ID to request
    next();
  });
};
```

#### Step 7: Protect Routes with Middleware
In your `userRoutes.ts`, use `authenticate` to protect specific routes:

```typescript
import { Router } from "express";
import userController from "./userController";
import { authenticate } from "../middleware/authMiddleware";

const router = Router();

router.get('/', authenticate, userController.get);
router.get('/:id', authenticate, userController.getByID);
router.post('/', authenticate, userController.create);
router.put('/:id', authenticate, userController.update);
router.delete('/:id', authenticate, userController.remove);

export default router;
```

#### Step 8: Integrate Authentication Routes into Main App File
In your `index.ts` or `server.ts`, add the `authRoutes`:

```typescript
import express from 'express';
import dotenv from 'dotenv';
import mongoose from 'mongoose';
import userRoutes from './routes/userRoutes';
import authRoutes from './routes/authRoutes';

dotenv.config();

const app = express();
app.use(express.json());

const PORT = process.env.USER_SERVICES_PATH || 3001;

app.use('/api/users', userRoutes);  // Protected routes
app.use('/api/auth', authRoutes);    // Auth routes for signup/signin

const connectDB = async () => {
  try {
    if (process.env.MONGODB_URI) {
      await mongoose.connect(process.env.MONGODB_URI);
      console.log("Connected to the DB");
    } else {
      console.log("Database Server URL not found in .env file");
    }
  } catch (error) {
    console.log("Error in connecting to the DB", error);
  }
};

connectDB();

export { app };
```

#### Step 9: Testing the API with Authentication
To test, use tools like Postman or cURL:

1. **Sign Up**: 
   - **POST** `http://localhost:3001/api/auth/signup`
   - **Body**:
     ```json
     {
       "name": "Test User",
       "email": "test@example.com",
       "password": "testpassword"
     }
     ```

2. **Sign In**:
   - **POST** `http://localhost:3001/api/auth/signin`
   - **Body**:
     ```json
     {
       "email": "test@example.com",
       "password": "testpassword"
     }
     ```
   - Response includes a JWT token that you should use as a Bearer token in headers for protected routes.

---

This implementation is adjusted for your serverless microservice setup, allowing each service to handle JWT-based authentication independently while protecting user-specific routes.