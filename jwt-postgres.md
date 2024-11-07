To modify the authentication guide for a project that uses PostgreSQL and Typeform with TypeORM, I'll update the structure, controller, and other components to match your existing setup. Here’s how you can structure the JWT authentication setup with PostgreSQL:

### JWT Authentication in PostgreSQL and Serverless Microservices

#### Updated Project Structure for Authentication Setup
Modify your project to include the following structure:

```
|-- /entities
|   |-- User.ts                // Entity for User in PostgreSQL
|-- /user-service
|   |-- /src
|       |-- /controllers
|       |   |-- authController.ts  // Authentication Controller
|       |-- /routes
|       |   |-- userRoutes.ts
|       |   |-- authRoutes.ts      // Route for authentication
|       |-- /middleware
|       |   |-- authMiddleware.ts  // Middleware for authentication
|       |-- server.ts              // Main server file
|-- package.json
|-- tsconfig.json
```

#### Step 1: Install Required Packages
Run these commands to install dependencies:

```bash
npm install express typeorm jsonwebtoken bcryptjs dotenv pg
npm install --save-dev typescript @types/node @types/express @types/jsonwebtoken @types/bcryptjs
```

#### Step 2: Define Environment Variables
Update the `.env` file with your PostgreSQL and JWT configurations:

```
DATABASE_URL=your_postgres_database_url
JWT_SECRET=your_secret_key
```

#### Step 3: Update the User Entity
Update `User.ts` to include fields for authentication and integrate with TypeORM:

```typescript
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from "typeorm";

@Entity()
export class User {
    @PrimaryGeneratedColumn()
    user_id!: number;

    @Column()
    name!: string;

    @Column({ unique: true })
    email!: string;

    @Column()
    password!: string;  // Store hashed password

    @CreateDateColumn()
    createdAt!: Date;
}
```

#### Step 4: Create the Authentication Controller
Create `authController.ts` in the `/controllers` directory to handle user signup and signin:

```typescript
import { Request, Response } from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { AppDataSource } from '../database/ormconfig';
import { User } from '../entities/User';

const userRepo = AppDataSource.getRepository(User);
const JWT_SECRET = process.env.JWT_SECRET || 'default_secret';

export const signUp = async (req: Request, res: Response) => {
  const { name, email, password } = req.body;
  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = userRepo.create({ name, email, password: hashedPassword });
    await userRepo.save(newUser);
    res.status(201).json({ message: 'User created successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Error creating user', error });
  }
};

export const signIn = async (req: Request, res: Response) => {
  const { email, password } = req.body;
  try {
    const user = await userRepo.findOne({ where: { email } });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user.user_id }, JWT_SECRET, { expiresIn: '1h' });
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
Create `authMiddleware.ts` in `/middleware` to validate JWT tokens:

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
    req.userId = (decoded as any).id;
    next();
  });
};
```

#### Step 7: Protect Routes with Middleware
In your `userRoutes.ts`, use the `authenticate` middleware to protect certain routes:

```typescript
import { Router } from 'express';
import userController from './userController';
import { authenticate } from '../middleware/authMiddleware';

const router = Router();

router.get('/', authenticate, userController.get);
router.get('/:id', authenticate, userController.getByID);
router.post('/', authenticate, userController.create);
router.put('/:id', authenticate, userController.update);
router.delete('/:id', authenticate, userController.remove);

export default router;
```

#### Step 8: Integrate Authentication Routes into Main App File
Add the `authRoutes` to your main server file, `server.ts`:

```typescript
import express from 'express';
import dotenv from 'dotenv';
import { AppDataSource } from '../database/ormconfig';
import userRoutes from './routes/userRoutes';
import authRoutes from './routes/authRoutes';

dotenv.config();
const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3001;

app.use('/api/users', userRoutes);
app.use('/api/auth', authRoutes);

const connectToDB = async () => {
  try {
    await AppDataSource.initialize();
    console.log('Connected to PostgreSQL');
  } catch (error) {
    console.error('Error connecting to DB', error);
  }
};

connectToDB();

export { app };
```

#### Step 9: Testing the API with Authentication
To test your API, use a tool like Postman:

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
   - Use the token from the response as a Bearer token in headers for protected routes.