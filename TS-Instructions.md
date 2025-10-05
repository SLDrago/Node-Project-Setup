# Follow this to Create a Node TS project

## Initializing project structure

```bash
mkdir -p node-backend-ts/src/{config,controllers,middlewares,models,routes,services,utils}
cd node-backend-ts
touch index.ts .env .gitignore tsconfig.json
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
cd node-backend-ts
npm init -y

# Install runtime dependencies
npm install express mongoose dotenv bcryptjs jsonwebtoken

# Install dev dependencies for TypeScript
npm install -D typescript ts-node @types/node @types/express @types/bcryptjs @types/jsonwebtoken nodemon

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
    await mongoose.connect(process.env.MONGO_URI || "", {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
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
import mongoose, { Document, Schema } from "mongoose";
import bcrypt from "bcryptjs";

export interface IUser extends Document {
  name: string;
  email: string;
  password: string;
  matchPassword(enteredPassword: string): Promise<boolean>;
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
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

userSchema.methods.matchPassword = async function (enteredPassword: string) {
  return await bcrypt.compare(enteredPassword, this.password);
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

### `src/index.ts`

```ts
import express, { Application, Request, Response } from "express";
import dotenv from "dotenv";
import connectDB from "./config/db";
import authRoutes from "./routes/authRoutes";

dotenv.config();
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
