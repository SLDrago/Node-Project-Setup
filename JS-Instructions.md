# Follow this to Create a Node JS project

## Initializing project structure

```bash
mkdir -p node-backend-starter/src/{config,controllers,middlewares,models,routes,services,utils}
cd node-backend-starter
touch index.js .env .gitignore
touch src/config/db.js
touch src/controllers/authController.js
touch src/middlewares/authMiddleware.js
touch src/models/userModel.js
touch src/routes/authRoutes.js
touch src/services/authService.js
touch src/utils/generateToken.js
```

## Initialize Project & Install Dependencies

```bash
cd node-backend-starter
npm init -y

# Install dependencies
npm install express mongoose dotenv bcryptjs jsonwebtoken
npm install -D nodemon
```

## Create `.env` file

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/mydb
JWT_SECRET=your_jwt_secret
```

## Example TS Code Snippets

## `src/config/db.js`

```js
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log("MongoDB connected");
  } catch (error) {
    console.error("MongoDB connection error:", error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### `src/models/userModel.js`

```js
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

const userSchema = new mongoose.Schema(
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

userSchema.methods.matchPassword = async function (enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model("User", userSchema);
```

### `src/utils/generateToken.js`

```js
const jwt = require("jsonwebtoken");

const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: "1d" });
};

module.exports = generateToken;
```

### `src/services/authService.js`

```js
const User = require("../models/userModel");

exports.registerUser = async (name, email, password) => {
  const userExists = await User.findOne({ email });
  if (userExists) throw new Error("User already exists");

  const user = await User.create({ name, email, password });
  return user;
};

exports.loginUser = async (email, password) => {
  const user = await User.findOne({ email });
  if (user && (await user.matchPassword(password))) return user;
  throw new Error("Invalid email or password");
};
```

### `src/controllers/authController.js`

```js
const authService = require('../services/authService');
const generateToken = require('../utils/generateToken');

exports.register = async (req, res) => {
const { name, email, password } = req.body;
try {
const user = await authService.registerUser(name, email, password);
res.status(201).json({
\_id: user.\_id,
name: user.name,
email: user.email,
token: generateToken(user.\_id),
});
} catch (error) {
res.status(400).json({ message: error.message });
}
};

exports.login = async (req, res) => {
const { email, password } = req.body;
try {
const user = await authService.loginUser(email, password);
res.json({
\_id: user.\_id,
name: user.name,
email: user.email,
token: generateToken(user.\_id),
});
} catch (error) {
res.status(401).json({ message: error.message });
}
};
```

### `src/routes/authRoutes.js`

```js
const express = require("express");
const router = express.Router();
const authController = require("../controllers/authController");

router.post("/register", authController.register);
router.post("/login", authController.login);

module.exports = router;
```

### `src/middlewares/authMiddleware.js`

```js
const jwt = require("jsonwebtoken");
const User = require("../models/userModel");

const protect = async (req, res, next) => {
  let token;
  if (
    req.headers.authorization &&
    req.headers.authorization.startsWith("Bearer")
  ) {
    try {
      token = req.headers.authorization.split(" ")[1];
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = await User.findById(decoded.id).select("-password");
      next();
    } catch (error) {
      res.status(401).json({ message: "Not authorized" });
    }
  }
  if (!token) res.status(401).json({ message: "No token, not authorized" });
};

module.exports = protect;
```

### `index.js`

```js
require("dotenv").config();
const express = require("express");
const connectDB = require("./src/config/db");
const authRoutes = require("./src/routes/authRoutes");

const app = express();
const PORT = process.env.PORT || 5000;

// Connect to DB
connectDB();

// Middleware
app.use(express.json());

// Routes
app.use("/api/auth", authRoutes);

// Default route
app.get("/", (req, res) => {
  res.send("Node Backend Starter is running!");
});

// Start server
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### `package.json`

```json
"scripts": {
"start": "node index.js",
"dev": "nodemon index.js"
}
```
