# Essential Code Snippets RESTful API
## Folder Structure

```
Project
	models
	
	—— Mongoose Schema
	
	routes
	
	—— Routes
	
	.env
	
	server.js
	
	validation.js
	
	package.json

```

## Server JS
```
const express = require(“express”);
const mongoose = require("mongoose");
const dotenv = require("dotenv");

dotenv.config();

//ExpressJS
const app = express();
const authRoute = require("./routes/auth");
const postsRoute = require("./routes/posts");
//MongoDB Connect
mongoose.connect(
  process.env.DB_CONNECT,
  {
    useNewUrlParser: true,
    useUnifiedTopology: true
  },
  () => console.log("Connected to DB!")
);

app.use(express.json());

app.use("/api/user", authRoute);
app.use("/api/posts", postsRoute);

app.listen(3000, () => console.log(“Server Up and Running at port"));
```

## .env File
```
DB_CONNECT = mongodb://localhost:27017/yesh
TOKEN_SECRET = jksdjhjkdhfdk
```


## Field Validation
```
const Joi = require(“@hapi/joi”);

//validation
const registerValidation = async data => {
  const schema = Joi.object({
    name: Joi.string()
      .min(6)
      .required(),
    email: Joi.string().email({
      minDomainSegments: 2,
      tlds: { allow: ["com", "net"] }
    }),
    password: Joi.string().pattern(/^[a-zA-Z0-9]{3,30}$/)
  });
  return await schema.validateAsync(data);
};

const loginValidation = async data => {
  const schema = Joi.object({
    email: Joi.string().email({
      minDomainSegments: 2,
      tlds: { allow: ["com", "net"] }
    }),
    password: Joi.string().pattern(/^[a-zA-Z0-9]{3,30}$/)
  });
  return await schema.validateAsync(data);
};

module.exports.registerValidation = registerValidation;
module.exports.loginValidation = loginValidation;
```

## Mongoose Schema (models)
```
const mongoose = require(“mongoose”);

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    min: 6,
    max: 255
  },
  email: {
    type: String,
    required: true,
    min: 6
  },
  password: {
    type: String,
    required: true,
    min: 6,
    max: 50
  },
  date: {
    type: Date,
    default: Date.now
  }
},
{ versionKey: false }
);

module.exports = mongoose.model("User", userSchema);
```

## Routes
### Authentication Route
```
const router = require(“express”).Router();
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const { registerValidation, loginValidation } = require("../validation");

router.post("/register", async (req, res) => {
  //Validating the entered data
  try {
    const value = await registerValidation(req.body);
  } catch (err) {
    return res.status(400).send(err.details[0].message);
  }

  // Checking if user already exits
  const emailExists = await User.findOne({ email: req.body.email });
  if (emailExists) return res.status(400).send(“Email already exists");

  //Hashing the passwords
  const salt = await bcrypt.genSalt();
  const hashPassword = await bcrypt.hash(req.body.password, salt);
  //CREATING NEW USER
  const user = new User({
    name: req.body.name,
    email: req.body.email,
    password: hashPassword
  });
  try {
    const savedUser = await user.save();
    res.send({ user: user._id });
  } catch (err) {
    res.status(400).send(err);
  }
});

//Login

router.post("/login", async (req, res) => {
  //Validating the entered data
  try {
    const value = await loginValidation(req.body);
  } catch (err) {
    return res.status(400).send(err.details[0].message);
  }

  // Checking if user already exits
  const user = await User.findOne({ email: req.body.email });
  if (!user) return res.status(400).send("Email does not exists");
  const validPass = await bcrypt.compare(req.body.password, user.password);
  if (!validPass) return res.status(400).send("Password does not exists");

  //Assigning a JWT
  const token = jwt.sign({ _id: user.id }, process.env.TOKEN_SECRET);
  res.header("auth-token", token).send(token);
});

module.exports = router;
```

### Verify JWT Token (verifyToken)
```
const jwt = require(“jsonwebtoken”);

module.exports = function(req, res, next) {
  const token = req.header("auth-token");
  if (!token) return res.status(401).send("Access Denied");

  try {
    const verified = jwt.verify(token, process.env.TOKEN_SECRET);
    req.user = verified;
    next();
  } catch (error) {
    res.status(400).send("Invalid Token");
  }
};
```

### Protected Route
```
const router = require(“express”).Router();
const verify = require("./verifyToken");

router.get("/", verify, (req, res) => {
  res.json({
    posts: {
      title: "First Post",
      description: "You shouldnt access"
    }
  });
});

module.exports = router;
```

### Mongooses + Express
```
const router = require(“express”).Router();
const Student = require("../models/Student");
const { registerValidation } = require("../validation");

router.post(“/register”, async (req, res) => {
  //Validating the entered data
  try {
    const value = await registerValidation(req.body);
  } catch (err) {
    return res.status(400).send(err);
  }

  // Checking if student already exits
  const idExists = await Student.findOne({ _id: req.body._id });
  if (idExists) return res.status(400).send("ID already exists");

  //CREATING NEW Student
  const student = new Student({
    _id: req.body._id,
    name: req.body.name
  });
  try {
    const savedStudent = await student.save();
    res.send({ ID: student._id });
  } catch (err) {
    res.status(400).send(err);
  }
});

router.get(“/list”, async (req, res) => {
  try {
    const students = await Student.find({});
    res.send(students);
  } catch (err) {
    return res.status(400).send(err);
  }
});

router.get("/:Studentid", async (req, res) => {
  try {
    const student = await Student.findById(req.params.Studentid);
    if (student != null) res.send({ Students: student });
    else res.status(404).send({ Error: err, Message: "Student ID not found" });
  } catch (err) {
    return res
      .status(404)
      .send({ Error: err, Message: "Student ID not found" });
  }
});

router.patch("/update", async (req, res) => {
  try {
    const idExists = await Student.findByIdAndUpdate(
      { _id: req.body._id },
      { name: req.body.name }
    );
    return res.send({ Result: idExists._id });
  } catch (err) {
    return res.status(400).send(err);
  }
});

router.delete("/delete", async (req, res) => {
  try {
    const idDelete = await Student.findByIdAndRemove({ _id: req.body._id });
    return res.send({ Result: idDelete._id });
  } catch (err) {
    return res.status(404).send({ Result: "ID not found" });
  }
});

module.exports = router;
```
