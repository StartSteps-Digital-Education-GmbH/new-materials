To create mock tests for your controller functions without making actual database connections, we’ll use Jest’s mocking capabilities to simulate database interactions, HTTP requests, and responses. This approach is particularly useful in a serverless environment like Vercel, where we want to avoid real connections and instead isolate the controller logic.

Here’s how we can set up and write the guide for testing each controller function with mocks.

### Steps to Mock Tests for Controller Functions

#### 1. **Set Up Required Mocks**

For testing, we need to mock:
- **Mongoose Models**: to simulate database operations without actual DB calls.
- **Express Request and Response Objects**: to simulate HTTP requests and responses.
- **Database Connection**: We’ll avoid calling `connectDB` by mocking it.

#### 2. **Install Required Libraries**

If you haven’t already, install `jest` and `ts-jest` (for TypeScript support):

```bash
npm install jest ts-jest @types/jest --save-dev
```

#### 3. **Configure the Test File Structure**

For each controller (e.g., `userController.ts`), create a corresponding test file in the `tests` directory, like `tests/userController.test.ts`.

#### 4. **Write the Test Cases with Mocks**

Here’s a sample guide for testing the controller functions in your code using Jest and Mongoose mock functions.

### Sample Guide for Mock Tests

#### Example Test File: `userController.test.ts`

```typescript
import { Request, Response } from 'express';
import mongoose from 'mongoose';
import User from '../src/models/userModel';
import userController from '../src/controllers/userController';

// Mock the database connection and model methods
jest.mock('mongoose', () => ({
  connect: jest.fn(),
  disconnect: jest.fn(),
}));

jest.mock('../src/models/userModel');

describe('User Controller Tests', () => {
  let req: Partial<Request>;
  let res: Partial<Response>;
  let statusMock: jest.Mock;
  let sendMock: jest.Mock;

  beforeEach(() => {
    // Reset mocks before each test
    req = {};
    statusMock = jest.fn().mockReturnThis();
    sendMock = jest.fn();
    res = { status: statusMock, send: sendMock };
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('GET /api/users', () => {
    it('should retrieve all users based on filters', async () => {
      req.query = { name: 'John', email: 'john@example.com' };

      // Mock the find method to return fake users
      (User.find as jest.Mock).mockResolvedValue([
        { _id: '123', name: 'John', email: 'john@example.com' },
      ]);

      await userController.get(req as Request, res as Response);

      expect(User.find).toHaveBeenCalledWith({ name: 'John', email: 'john@example.com' });
      expect(statusMock).toHaveBeenCalledWith(200);
      expect(sendMock).toHaveBeenCalledWith([
        { _id: '123', name: 'John', email: 'john@example.com' },
      ]);
    });
  });

  describe('GET /api/users/:id', () => {
    it('should return a user by ID', async () => {
      req.params = { id: '123' };

      // Mock the findById method to return a specific user
      (User.findById as jest.Mock).mockResolvedValue({ _id: '123', name: 'John' });

      await userController.getByID(req as Request, res as Response);

      expect(User.findById).toHaveBeenCalledWith('123');
      expect(statusMock).toHaveBeenCalledWith(200);
      expect(sendMock).toHaveBeenCalledWith({ _id: '123', name: 'John' });
    });

    it('should return 404 if user not found', async () => {
      req.params = { id: '123' };

      // Mock the findById method to return null
      (User.findById as jest.Mock).mockResolvedValue(null);

      await userController.getByID(req as Request, res as Response);

      expect(User.findById).toHaveBeenCalledWith('123');
      expect(statusMock).toHaveBeenCalledWith(404);
      expect(sendMock).toHaveBeenCalledWith({ message: 'User not found' });
    });
  });

  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      req.body = { name: 'Jane', email: 'jane@example.com' };

      // Mock the save method to simulate user creation
      (User.prototype.save as jest.Mock).mockResolvedValue({
        _id: '456',
        name: 'Jane',
        email: 'jane@example.com',
      });

      await userController.create(req as Request, res as Response);

      expect(statusMock).toHaveBeenCalledWith(201);
      expect(sendMock).toHaveBeenCalledWith({
        _id: '456',
        name: 'Jane',
        email: 'jane@example.com',
      });
    });
  });

  describe('PUT /api/users/:id', () => {
    it('should update a user by ID', async () => {
      req.params = { id: '123' };
      req.body = { name: 'John Updated', email: 'johnupdated@example.com' };

      // Mock the findByIdAndUpdate method to simulate user update
      (User.findByIdAndUpdate as jest.Mock).mockResolvedValue({
        _id: '123',
        name: 'John Updated',
        email: 'johnupdated@example.com',
      });

      await userController.update(req as Request, res as Response);

      expect(User.findByIdAndUpdate).toHaveBeenCalledWith(
        '123',
        { name: 'John Updated', email: 'johnupdated@example.com' },
        { new: true }
      );
      expect(statusMock).toHaveBeenCalledWith(200);
      expect(sendMock).toHaveBeenCalledWith({
        _id: '123',
        name: 'John Updated',
        email: 'johnupdated@example.com',
      });
    });

    it('should return 404 if user to update is not found', async () => {
      req.params = { id: '123' };
      req.body = { name: 'Nonexistent User', email: 'noexist@example.com' };

      // Mock the findByIdAndUpdate method to return null
      (User.findByIdAndUpdate as jest.Mock).mockResolvedValue(null);

      await userController.update(req as Request, res as Response);

      expect(statusMock).toHaveBeenCalledWith(404);
      expect(sendMock).toHaveBeenCalledWith({ message: 'User not found' });
    });
  });

  describe('DELETE /api/users/:id', () => {
    it('should delete a user by ID', async () => {
      req.params = { id: '123' };

      // Mock the findByIdAndDelete method to simulate user deletion
      (User.findByIdAndDelete as jest.Mock).mockResolvedValue({
        _id: '123',
        name: 'John',
        email: 'john@example.com',
      });

      await userController.remove(req as Request, res as Response);

      expect(User.findByIdAndDelete).toHaveBeenCalledWith('123');
      expect(statusMock).toHaveBeenCalledWith(204);
      expect(sendMock).toHaveBeenCalled();
    });

    it('should return 404 if user to delete is not found', async () => {
      req.params = { id: '123' };

      // Mock the findByIdAndDelete method to return null
      (User.findByIdAndDelete as jest.Mock).mockResolvedValue(null);

      await userController.remove(req as Request, res as Response);

      expect(statusMock).toHaveBeenCalledWith(404);
      expect(sendMock).toHaveBeenCalledWith({ message: 'User not found' });
    });
  });
});
```

### Explanation of Key Components

- **Mocking Mongoose Model Methods**: We mock methods like `find`, `findById`, `save`, `findByIdAndUpdate`, and `findByIdAndDelete` to simulate database interactions.
- **Express `req` and `res`**: Partial implementations of `Request` and `Response` objects are created, with `status` and `send` mocked using Jest.
- **Assertions**: Each test checks that the expected controller function was called with the correct parameters and that the response status and payload match expectations.

### Running the Tests

Run your tests with:

```bash
npm run test
```

This mock-based approach tests only the controller logic and response handling, bypassing actual database or server requests. This setup is especially efficient for serverless functions, as it enables isolated, fast, and reliable tests.