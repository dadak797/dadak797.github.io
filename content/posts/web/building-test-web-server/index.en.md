---
title: Building a Test Web Server with Node.js and Express
date: 2026-09-10
draft: true
description: Build a simple backend with Node.js and Express that runs server-side logic for each request, then verify that each API behaves as expected.
categories:
  - Web
tags:
  - web-server
  - node-js
  - express
  - backend
  - cors
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> VS Code's Live Server extension and Python's `http.server` are development preview tools that only serve static files. They cannot implement dynamic features such as authentication, data storage, or generated API responses. This post builds a simple backend with Node.js and Express that runs server-side logic for each request, then verifies that each API behaves as expected.

## Installing Node.js

```bash
node -v  # Check the Node.js installation
npm -v   # Check the npm installation
```

- https://nodejs.org/en/download
- If both commands print a version, the installation was successful.

## Initializing the Project

```bash
npm init
```

- Enter a value for each prompt to create `package.json`. To accept all the defaults, use `npm init -y` instead.

```jsonc
// package.json
{
  "name": "ex-15",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "commonjs"
}
```

- name: The project name
- version: The project version
- description: A short description of the project
- main: The package entry point. When another module loads this package with a call such as `require("ex-15")`, Node.js loads `index.js`. A server application that is run directly does not import this package, so this field can be removed.
- scripts: Frequently used commands. Running `npm run test` executes `echo \"Error: no test specified\" && exit 1`.
- keywords: Keywords describing the package
- author: The author's name
- license: The type of license
- type: The module system to use. `commonjs` uses the traditional `require()` and `module.exports` syntax, while `module` enables ES module syntax with `import` and `export`.

## Installing Server Packages

```bash
npm install express
npm install cors
npm install multer
npm install --save-dev nodemon
```

- `express`: Creates the web server and handles routes.
- `cors`: Sets Cross-Origin Resource Sharing (CORS) response headers. It is needed when a browser must read an API response from a different origin.
- `multer`: Processes `multipart/form-data` file uploads.
- `nodemon`: Automatically restarts the server process when its source code changes.
- `--save-dev`: Installs a package that is only needed during development.
- Installing these packages adds them to `package.json` and creates the `node_modules` directory and `package-lock.json` file.

```jsonc
// package.json
{
  // ...
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  // ...
  "type": "module",
  "dependencies": {
    "cors": "^2.8.6",
    "express": "^5.2.1",
    "multer": "^2.3.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
```

- scripts: Remove the default script and add two scripts, `start` and `dev`.
- dependencies: Packages required in production. A package installed with `npm install package-name` is added here.
- devDependencies: Packages needed only during development. A package installed with `npm install package-name --save-dev` is added here.
- `node_modules`
  - Contains the installed package files.
  - This directory can be very large and can be recreated from `package.json` and `package-lock.json`, so add it to `.gitignore` instead of committing it to the repository.

> [!NOTE]
> `package.json` records the version ranges of packages that may be installed, while `package-lock.json` records the exact versions in the complete dependency tree that was installed. To reproduce the same dependency environment, commit both files and use `npm ci`. This command fails when the two files do not match and does not modify either `package.json` or `package-lock.json` during installation.

## Basic Server Configuration

- Before starting the server, create an `uploads` directory in the project root for uploaded files. Also place the `Cube.obj` file used by the download test in a `models` directory.

```javascript
// server.js
import express from "express";
import cors from "cors";
import multer from "multer";
import path from "path";
import { fileURLToPath } from "url";

const app = express();
const port = 3000;
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Parse JSON POST requests.
// This middleware runs when the request Content-Type is application/json.
app.use(express.json());

// Configure CORS.
// Allow API calls from a client running on a different origin.
// Example: http://localhost:8080 → http://localhost:3000
app.use(
  cors({
    origin: "http://localhost:8080",
    allowedHeaders: ["Content-Type", "Authorization"],
  }),
);

// Configure where uploaded files are stored.
// Multer normally generates random filenames. Here, a timestamp is appended
// to the original filename to make it unique.
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    const ext = path.extname(file.originalname);
    const base = path.basename(file.originalname, ext);
    cb(null, `${base}-${Date.now()}${ext}`);
  },
});
const upload = multer({ storage });

// Define the API endpoints.
// 1. Basic GET request
app.get("/hello", (req, res) => {
  res.json({
    message: "Hello from Node.js server",
  });
});

// 2. GET request with an authorization header
app.get("/auth", (req, res) => {
  const authorization = req.get("Authorization");

  // Return an error when the Authorization header is missing.
  if (!authorization) {
    return res.status(401).json({
      error: "Authorization header is required",
    });
  }

  res.json({
    message: "Authorization header received",
    authorization: authorization,
  });
});

// 3. POST request with a JSON body
app.post("/echo", (req, res) => {
  console.log("POST body:");
  console.log(req.body);

  // Return the received JSON body unchanged.
  res.json({
    received: req.body,
  });
});

// 4. File upload
app.post("/upload", upload.single("uploadFile"), (req, res) => {
  if (!req.file) {
    return res.status(400).json({
      error: "No file uploaded",
    });
  }

  console.log(req.file);

  res.json({
    message: "File uploaded successfully",
    filename: req.file.originalname,
    size: req.file.size,
  });
});

// 5. File streaming and download
app.get("/models/:filename", (req, res) => {
  const filesDir = path.join(__dirname, "models");

  // The root option makes Express confine the file path to the models directory.
  res.sendFile(req.params.filename, { root: filesDir });
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

- Configure JSON POST request parsing and CORS.
- `cors`: Middleware that sets CORS response headers so a browser can read an API response from another origin. This example assumes that a client running at `http://localhost:8080` calls an API at `http://localhost:3000`.
- Configure file uploads with Multer.
- Create controllers for five basic API tests using GET and POST requests:
  1. `/hello`: Returns a basic JSON response.
  2. `/auth`: Checks for an authorization header. This is only a header-checking example and does not validate the token.
  3. `/echo`: Accepts a JSON body in a POST request and returns it unchanged.
  4. `/upload`: Uploads a file.
  5. `/models/:filename`: Streams or downloads a file. The value after `/models/` is passed as `req.params.filename`, and the `root` option prevents access to files outside the `models` directory.

## Testing the Server

![Web server test](images/web-server-test.png)

- Start the server with `npm run dev`, then run the following commands in order.

### Test Commands

```bash
# Start the server.
npm run dev

# --- In a separate terminal window ---

# GET request
curl http://localhost:3000/hello

# GET request with an authorization header
curl \
  -H "Authorization: Bearer test-token" \
  http://localhost:3000/auth

# GET request without an authorization header
curl http://localhost:3000/auth

# POST request
curl \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Emscripten","language":"C++"}' \
  http://localhost:3000/echo

# Download a file.
curl http://localhost:3000/models/Cube.obj \
  -o Cube.obj

# Check the downloaded file.
ls

# Upload a file.
curl \
  -F "uploadFile=@Cube.obj" \
  http://localhost:3000/upload

# Stream a file.
curl http://localhost:3000/models/Cube.obj
```

- After uploading the file, check the server's `uploads` directory. It should contain a file such as `Cube-1788845621235.obj`. The number after `Cube-` is a timestamp and will vary.

### Example Code and Node.js Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-15
- Node.js v26.5.0
- npm 11.17.0

## FAQ

{{< faq summary="What are the advantages of an Express-based Node.js server compared with Flask, Django, or Spring Boot?" >}}
- Node.js is a JavaScript runtime, while Express is a web framework that runs on it. Strictly speaking, an Express-based Node.js server should therefore be compared with Flask, Django, or Spring Boot.
- Using JavaScript or TypeScript for both the frontend and backend makes it easier to share a language and data models, and it provides access to the npm ecosystem. Node.js also handles asynchronous, I/O-heavy requests efficiently, making it a lightweight choice for API and real-time communication servers.
- Django or Spring Boot may be more convenient for large applications that need built-in features such as authentication, an ORM, and administrative tools. CPU-intensive work can block the Node.js event loop, so the best choice depends on the server's purpose and the team's technology stack.
{{< /faq >}}

{{< faq summary="Why is `type` set to `module` in `package.json`?" >}}
- The `server.js` file in this post uses ES module features such as `import` and `import.meta.url`. To interpret a `.js` file as an ES module, set `"type": "module"` in the nearest `package.json`.
- To keep `"type": "commonjs"`, replace `import` with `require()` and `module.exports`. Alternatively, use the `.mjs` extension to run the file as an ES module regardless of the `type` setting.
{{< /faq >}}

{{< faq summary="Why is CORS required even when both servers use `localhost`?" >}}
- An origin is defined by the combination of protocol, host, and port. The client at `http://localhost:8080` and the API server at `http://localhost:3000` therefore have different origins because their ports differ. The API server must send the appropriate CORS response headers for a browser to expose the response to the client.
- CORS is not an authentication mechanism that blocks the request at the server. It is a browser policy that controls whether client-side code can read the response. Tools such as `curl` and Postman, as well as other servers, do not enforce CORS, so a successful request from one of them does not prove that the browser CORS configuration is correct.
{{< /faq >}}

{{< faq summary="Why does a file upload fail with `ENOENT` or `Unexpected field`?" >}}
- When Multer's `destination` is provided as a function, as it is in this example, the `uploads` directory must already exist. If it does not, create it with `mkdir uploads` before starting the server.
- The argument passed to `upload.single("uploadFile")` is the file field name expected by the server. The client must use the same name, as in `curl -F "uploadFile=@Cube.obj"`. Using a different name can cause an `Unexpected field` error.
{{< /faq >}}

{{< faq summary="Does `res.sendFile()` load the entire file into memory before sending it?" >}}
- No. `res.sendFile()` streams the file, so it does not need to load the entire file into memory first as `fs.readFile()` does. This makes it suitable for APIs that download relatively large files.
- In this example, the `root` option is set to the `models` directory, preventing the requested path from escaping that directory.
{{< /faq >}}

{{< faq summary="Can this example server be used as-is in production?" >}}
- This server is a test example intended to demonstrate basic Express features. In particular, `/auth` only checks whether an `Authorization` header exists; it does not validate the token or check the user's permissions.
- A production service also needs authentication and authorization, request validation, upload size and file type limits, safe filename generation, HTTPS, rate limiting, logging, and centralized error handling.
{{< /faq >}}
