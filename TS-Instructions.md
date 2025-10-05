# Follow this to Create a Node TS project

## Initializing project structure

```bash
mkdir -p node-backend-ts/src/{config,controllers,middlewares,models,routes,services,utils}
cd node-backend-ts
touch .env .gitignore tsconfig.json
touch src/index.ts
touch src/config/db.ts
touch src/controllers/authController.ts
touch src/middlewares/authMiddleware.ts
touch src/models/userModel.ts
touch src/routes/authRoutes.ts
touch src/services/authService.ts
touch src/utils/generateToken.ts
```

## Initialize Project & Install Dependencies

```bash
npm init -y

# Install runtime dependencies
npm install express mongoose dotenv jsonwebtoken argon2

# Install dev dependencies for TypeScript
npm install -D typescript ts-node @types/node @types/express @types/jsonwebtoken nodemon

```

## Create `.env` file

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/mydb
JWT_SECRET=your_jwt_secret
```

## Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

## Scripts in `package.json`

```json
"scripts": {
  "start": "node dist/index.js",
  "dev": "nodemon --exec ts-node src/index.ts",
  "build": "tsc"
}
```

## Example TS Code Snippets

### `src/config/db.ts`

```ts
import mongoose from "mongoose";

const connectDB = async (): Promise<void> => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log("MongoDB connected");
  } catch (error) {
    console.error("MongoDB connection error:", (error as Error).message);
    process.exit(1);
  }
};

export default connectDB;
```

### `src/models/userModel.ts`

```ts
import mongoose, { Document, Schema, Types } from "mongoose";
import argon2 from "argon2";

export interface IUser extends Document {
  name: string;
  email: string;
  password: string;
  matchPassword(enteredPassword: string): Promise<boolean>;
  _id: Types.ObjectId;
}

const userSchema: Schema<IUser> = new Schema(
  {
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
  },
  { timestamps: true }
);

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  this.password = await argon2.hash(this.password, { type: argon2.argon2id });
  next();
});

userSchema.methods.matchPassword = async function (password: string) {
  return await argon2.verify(this.password, password);
};

export default mongoose.model<IUser>("User", userSchema);
```

### `src/utils/generateToken.ts`

```ts
import jwt from "jsonwebtoken";

const generateToken = (id: string): string => {
  return jwt.sign({ id }, process.env.JWT_SECRET || "", { expiresIn: "1d" });
};

export default generateToken;
```

### `src/controllers/authController.ts`

```ts
import { Request, Response } from "express";
import { registerUser, loginUser } from "../services/authService";
import generateToken from "../utils/generateToken";

// Register Controller
export const register = async (req: Request, res: Response): Promise<void> => {
  const { name, email, password } = req.body;
  try {
    const user = await registerUser(name, email, password);
    res.status(201).json({
      _id: user._id,
      name: user.name,
      email: user.email,
      token: generateToken(user._id.toHexString()),
    });
  } catch (error) {
    res.status(400).json({ message: (error as Error).message });
  }
};

// Login Controller
export const login = async (req: Request, res: Response): Promise<void> => {
  const { email, password } = req.body;
  try {
    const user = await loginUser(email, password);
    res.json({
      _id: user._id,
      name: user.name,
      email: user.email,
      token: generateToken(user._id.toString()),
    });
  } catch (error) {
    res.status(401).json({ message: (error as Error).message });
  }
};

```

### `src/middlewares/authMiddleware.ts`

```ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import User, { IUser } from "../models/userModel";

interface JwtPayload {
  id: string;
}

export const protect = async (
  req: Request & { user?: IUser },
  res: Response,
  next: NextFunction
) => {
  let token;

  if (req.headers.authorization?.startsWith("Bearer")) {
    try {
      token = req.headers.authorization.split(" ")[1];
      const decoded = jwt.verify(
        token,
        process.env.JWT_SECRET || ""
      ) as JwtPayload;
      req.user = await User.findById(decoded.id).select("-password")!;
      next();
    } catch (error) {
      res.status(401).json({ message: "Not authorized, token failed" });
    }
  }

  if (!token) {
    res.status(401).json({ message: "No token, not authorized" });
  }
};

```

### `src/services/authService.ts`

```ts
import User, { IUser } from "../models/userModel";

// Register a new user
export const registerUser = async (
  name: string,
  email: string,
  password: string
): Promise<IUser> => {
  const userExists = await User.findOne({ email });
  if (userExists) throw new Error("User already exists");

  const user = await User.create({ name, email, password });
  return user;
};

// Login user
export const loginUser = async (
  email: string,
  password: string
): Promise<IUser> => {
  const user = await User.findOne({ email });
  if (user && (await user.matchPassword(password))) return user;
  throw new Error("Invalid email or password");
};

```

### `src/routes/authRoutes.ts`

```ts
import express from "express";
import { register, login } from "../controllers/authController";

const router = express.Router();

// Register a new user
router.post("/register", register);

// Login a user
router.post("/login", login);

export default router;

```

### `src/index.ts`

```ts
import express, { Application, Request, Response } from "express";
import dotenv from "dotenv";
import connectDB from "./config/db";
import authRoutes from "./routes/authRoutes";

dotenv.config({ debug: true });
connectDB();

const app: Application = express();
const PORT: string | number = process.env.PORT || 5000;

app.use(express.json());
app.use("/api/auth", authRoutes);

app.get("/", (req: Request, res: Response) => {
  res.send("Node TS Backend Starter is running!");
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

```
