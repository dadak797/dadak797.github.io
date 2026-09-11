---
title: 테스트용 웹 서버 구축하기 - Node.js & Express
date: 2026-09-10
draft: false
description: Node.js와 Express를 이용하여 요청마다 서버 로직을 실행할 수 있는 백엔드를 간단히 구축하고, 각 API가 요청한 동작을 잘 수행하는지 테스트한다.
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
> VS-Code의 확장 프로그램인 Live Server나 Python `http.server`는 정적 파일을 보여주기만 하는 개발용 미리보기 도구라서, 실제 서비스처럼 로그인·데이터 저장·API 응답 같은 동적 기능을 구현할 수는 없다. 이 글에서는 Node.js와 Express를 이용하여 요청마다 서버 로직을 실행할 수 있는 백엔드를 간단히 구축하고, 각 API가 요청한 동작을 잘 수행하는지 테스트한다.

## Node.js 설치

```bash
node -v  # Node.js 설치 확인
npm -v   # npm 설치 확인
```

- https://nodejs.org/ko/download
- 각 버전이 출력되면 설치가 잘 되었다는 뜻

## 프로젝트 초기화

```bash
npm init
```

- 각 항목을 입력하면 package.json이 생성됨. 귀찮으면 `npm init -y`로 하여 기본값을 사용

```jsonc
// package.json
{
  "name": "ex-15",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "commonjs",
}
```

- name: 프로젝트 이름
- version: 버전
- description: 프로젝트에 관한 간단한 설명
- main: 이 패키지의 진입점(entry point)이 되는 파일. 다른 코드에서 `require("ex-15")`처럼 이 패키지를 불러오면 `index.js`가 로드됨. 직접 실행하는 서버 애플리케이션에서는 이 패키지를 import 하지 않기 때문에 삭제해도 무방함
- scripts: 자주 쓰는 명령어를 등록할 수 있음. `npm run test`를 실행하면 `echo \"Error: no test specified\" && exit 1`가 실행됨
- keywords: 핵심어
- author: 작성자의 이름
- license: 라이선스의 종류
- type: 사용할 모듈 시스템을 지정함. `commonjs`는 전통적인 `require()`/`module.exports` 방식을 사용한다는 뜻이고, `module`로 바꾸면 ES module 방식(`import`/`export`)을 사용하겠다는 뜻

## 서버 구동을 위한 패키지 설치

```bash
npm install express
npm install cors
npm install multer
npm install --save-dev nodemon
```

- `express`: 웹 서버를 생성해 주는 패키지
- `cors`: 교차 출처 리소스 공유(Cross-Origin Resource Sharing; CORS)를 위한 응답 헤더를 설정하는 패키지. 브라우저에서 다른 origin의 API 응답을 읽어야 할 때 필요함
- `multer`: `multipart/form-data` 파일 업로드를 쉽게 처리하기 위해 사용하는 패키지
- `nodemon`: 서버의 소스코드가 수정되면 프로세스를 자동 재시작
- `--save-dev`: 개발 중에만 사용하는 패키지를 설치할 때 붙여줘야 함
- 패키지들을 설치하면 `package.json`에 아래와 같이 패키지 목록이 추가되고, `node_modules` 폴더와 `package-lock.json` 파일이 생성됨

```jsonc
// package.json
{
  // ...
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
  },
  // ...
  "type": "module",
  "dependencies": {
    "cors": "^2.8.6",
    "express": "^5.2.1",
    "multer": "^2.3.0",
  },
  "devDependencies": {
    "nodemon": "^3.1.14",
  },
}
```

- scripts: 기본 스크립트를 삭제하고, 두 가지 스크립트(`start`, `dev`)를 추가
- dependencies: 배포에 필요한 패키지 목록. `npm install 패키지명`으로 패키지를 설치했을 때, 그 이름이 여기에 추가됨
- devDependencies: 개발 중에만 필요한 패키지 목록. `npm install 패키지명 --save-dev`로 설치하면 여기에 추가됨
- `node_modules`
  - 설치한 패키지 파일들이 저장되는 폴더
  - 용량이 매우 크고 `package.json`과 `package-lock.json`을 통해 다시 생성할 수 있으므로, `.gitignore`에 추가하고 저장소에는 포함하지 않음

> [!NOTE]
> `package.json`은 설치 가능한 패키지의 버전 범위를 기록하고, `package-lock.json`은 실제로 설치된 전체 의존성 트리의 정확한 버전을 기록한다. 동일한 의존성 환경을 재현하려면 두 파일을 저장소에 함께 커밋하고 `npm ci`를 사용한다. `npm ci`는 두 파일의 내용이 일치하지 않으면 오류를 발생시키며, 설치 과정에서 `package.json`이나 `package-lock.json`을 수정하지 않는다.

## 서버 설정 파일과 프로젝트 구조

- 서버를 실행하기 전에 프로젝트 루트에 업로드 파일을 저장할 `uploads` 폴더를 만들고, 개발자 도구에서 API를 테스트할 때 사용할 `index.html`을 작성함

```JavaScript
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
const uploadDir = path.join(__dirname, "uploads");

// CORS 설정
// http://localhost:8080에서 실행되는 별도의 클라이언트가
// http://localhost:3000의 API를 호출할 수 있도록 허용
app.use(
  cors({
    origin: "http://localhost:8080",
    allowedHeaders: ["Content-Type", "Authorization"],
  }),
);

// JSON POST 요청 처리 설정
// 요청의 Content-Type이 application/json 일 때 동작함
app.use(express.json());

// 업로드된 파일 저장 위치 설정
// 기본적으로 파일명은 무작위로 생성되지만, 여기서는 파일이름에 타임스탬프를 붙여 고유하게 만듦
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, uploadDir);
  },
  filename: (req, file, cb) => {
    const ext = path.extname(file.originalname);
    const base = path.basename(file.originalname, ext);
    cb(null, `${base}-${Date.now()}${ext}`);
  },
});
const upload = multer({ storage });

// Home
app.get("/", (req, res) => {
  res.sendFile(path.join(__dirname, "index.html"));
});

// API 생성
// 1. GET 요청
app.get("/hello", (req, res) => {
  res.json({
    message: "Hello from Node.js server",
  });
});

// 2. 인증 헤더가 포함된 GET
app.get("/auth", (req, res) => {
  const authorization = req.get("Authorization");

  // 인증 헤더에 Authorization이 없으면 오류 발생
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

// 3. JSON body를 가지고 POST 요청
app.post("/echo", (req, res) => {
  console.log("POST body:");
  console.log(req.body);

  // 요청으로 받은 JSON body를 그대로 리턴
  res.json({
    received: req.body,
  });
});

// 4. 파일 업로드
app.post("/upload", upload.single("uploadFile"), (req, res) => {
  if (!req.file) {
    return res.status(400).json({
      error: "No file uploaded",
    });
  }

  res.json({
    message: "File uploaded successfully",
    originalName: req.file.originalname,
    filename: req.file.filename,
    size: req.file.size,
    downloadUrl: `/download/${encodeURIComponent(req.file.filename)}`,
  });
});

// 5. 파일 다운로드
app.get("/download/:filename", (req, res) => {
  // root 옵션을 지정하면 Express가 파일 경로를 uploads 폴더 내부로 제한함
  res.sendFile(req.params.filename, { root: uploadDir });
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

```html
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-15</title>
  </head>
  <body></body>
</html>
```

- JSON POST 요청과 CORS를 설정
- `cors`: 브라우저에서 다른 origin의 API 응답을 읽을 수 있도록 CORS 응답 헤더를 설정하는 미들웨어. 이 글의 개발자 도구 테스트는 같은 origin에서 실행되므로 CORS가 필요하지 않지만, `http://localhost:8080`에서 실행되는 별도의 클라이언트가 `http://localhost:3000`의 API를 호출하는 상황을 위해 설정함
- multer를 이용한 파일 업로드 설정
- GET, POST를 조합한 총 5가지의 API와 컨트롤러를 생성하여 기본적인 테스트를 수행
  1.  `/hello`: 기본적인 JSON 객체를 응답하는 API
  2.  `/auth`: 인증 헤더를 확인하는 API. 토큰을 인증하는 것은 아니고 테스트용 헤더 확인 예제
  3.  `/echo`: JSON body를 POST 요청으로 받고, 그 요청을 그대로 리턴하는 API
  4.  `/upload`: 파일을 `uploads` 폴더에 저장하고, 저장된 파일명과 다운로드 URL을 응답하는 API
  5.  `/download/:filename`: 파일 다운로드를 위한 API. `/download/` 뒤의 문자열은 `req.params.filename`으로 전달되며, `root` 옵션을 통해 `uploads` 폴더 외부의 파일에는 접근할 수 없도록 제한함

### 프로젝트 구조

```
ex-15/
├── node_modules/
├── uploads/
├── index.html
├── package-lock.json
├── package.json
└── server.js
```

## 서버 테스트 하기

### 서버 구동 시키기

```bash
npm run dev
```

- `http://localhost:3000`에 접속한 뒤 개발자 도구의 Console 탭에서 아래 스크립트를 순차적으로 실행

### GET 요청 테스트

```JavaScript
const response = await fetch('http://localhost:3000/hello', {
  method: 'GET',
});
const json = await response.json();
console.log(json);
```

![GET test](images/GET_test.png)
_그림 1. GET 요청 결과 - 서버로부터 정상적으로 응답을 받음_

### 인증 헤더가 포함된 GET 요청 테스트

```JavaScript
// 인증 헤더를 포함하여 요청
const response = await fetch('http://localhost:3000/auth', {
  method: 'GET',
  headers: {
    'Authorization': 'Bearer test-token',
  },
});
const json = await response.json();
console.log(json);

// 인증 헤더 없이 요청
const response_no_header = await fetch('http://localhost:3000/auth', {
  method: 'GET',
});
const json_no_header = await response_no_header.json();
console.log(json_no_header);
```

![get with auth](images/GET_with_auth.png)
_그림 2. 인증 헤더를 포함한 GET 요청 결과 - 인증 헤더가 포함되지 않은 경우에는 오류가 발생함_

### POST 요청 테스트

```JavaScript
const response = await fetch('http://localhost:3000/echo', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    'name': 'Emscripten',
    'language': 'C++',
  }),
});
const json = await response.json();
console.log(json);
```

![post test](images/POST_test.png)
_그림 3. POST 요청 결과 - 전달한 JSON body를 그대로 돌려 받음_

### 파일 업로드 테스트

```JavaScript
const textContent = "Hello from JavaScript";
const fileName = "test_file.txt";

const textBlob = new Blob([textContent], { type: 'text/plain' });
const mockFile = new File([textBlob], fileName, { type: 'text/plain' });

const formData = new FormData();
formData.append('uploadFile', mockFile);

const uploadResponse = await fetch('http://localhost:3000/upload', {
  method: 'POST',
  body: formData,
});
const uploadResult = await uploadResponse.json();
console.log(uploadResult);

// 다음 다운로드 테스트에서 사용
globalThis.downloadUrl = uploadResult.downloadUrl;
```

- 문자열로 `File` 객체를 만든 뒤 `FormData`에 추가하여 POST 요청으로 전달
- 서버는 파일을 `test_file-{timestamp}.txt` 형식으로 `uploads` 폴더에 저장하고, 저장된 파일명과 다운로드 URL을 응답함

![get file upload](images/POST_file_upload.png)
_그림 4. POST 파일 업로드 요청 결과 - `uploads` 폴더에 `test_file-{timestamp}.txt` 파일이 생성됨_

### 파일 다운로드 테스트

- 이전 테스트의 업로드 응답으로 받은 `downloadUrl`을 이용하여 `uploads` 폴더에 저장된 파일을 내려받음. 이전 예제 실행 후 브라우저를 갱신하지 않고 실행해야 함

```JavaScript
// 파일 다운로드
const downloadResponse = await fetch(globalThis.downloadUrl);
const blob = await downloadResponse.blob();

const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = 'test_file.txt';
document.body.appendChild(a);
a.click();

a.remove();
URL.revokeObjectURL(url);   // 메모리 정리

// 응답 본문을 문자열로 읽기
const textResponse = await fetch(globalThis.downloadUrl);
const text = await textResponse.text();
console.log(text);
```

![get file download](images/GET_file_download.png)
_그림 5. GET 파일 다운로드 요청 결과 - 첫 번째 요청에서는 `test_file.txt` 파일을 내려받고, 두 번째 요청에서는 응답 본문을 문자열로 읽어 파일 내용을 확인함_

### 예제 코드 및 Node.js 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-15
- Node.js v26.5.0
- npm 11.17.0

## FAQ

{{< faq summary="Express 기반 Node.js 서버를 Flask, Django나 Spring Boot와 비교하면 어떤 장점이 있나요?" >}}

- Node.js는 JavaScript 런타임이고 Express는 그 위에서 동작하는 웹 프레임워크입니다. 따라서 정확히는 Express 기반 Node.js 서버와 Flask, Django, Spring Boot를 비교해야 합니다.
- 프론트엔드와 백엔드를 모두 JavaScript 또는 TypeScript로 작성할 수 있어 언어와 데이터 모델을 공유하기 쉽고, npm 생태계의 패키지를 활용할 수 있다는 장점이 있습니다. 또한 비동기 I/O 중심의 요청을 효율적으로 처리하므로 API 서버나 실시간 통신 서버를 가볍게 구축하기 좋습니다.
- 반면 인증, ORM, 관리 도구처럼 다양한 기능이 기본으로 필요한 대규모 애플리케이션에서는 Django나 Spring Boot가 더 편리할 수 있습니다. CPU 연산이 많은 작업은 Node.js의 이벤트 루프를 막을 수 있으므로, 서버의 목적과 팀의 기술 스택에 따라 선택하는 것이 좋습니다.
  {{< /faq >}}

{{< faq summary="`package.json`의 `type`을 `module`로 설정한 이유는 무엇인가요?" >}}

- 이 글의 `server.js`는 `import`와 `import.meta.url`을 사용하는 ES module 방식으로 작성되어 있습니다. `.js` 파일을 ES module로 해석하려면 가장 가까운 `package.json`에 `"type": "module"`을 지정해야 합니다.
- `"type": "commonjs"`를 유지하려면 `import` 대신 `require()`와 `module.exports`를 사용해야 합니다. 또는 파일 확장자를 `.mjs`로 변경하면 `type` 설정과 관계없이 ES module로 실행할 수 있습니다.
  {{< /faq >}}

{{< faq summary="개발자 도구에서 테스트할 때도 CORS 설정이 필요한가요?" >}}

- 이 글처럼 `http://localhost:3000`에서 개발자 도구를 열어 같은 origin의 API를 호출할 때는 CORS 설정이 필요하지 않습니다.
- 현재 `cors` 미들웨어는 `http://localhost:8080`에서 실행되는 별도의 클라이언트가 `http://localhost:3000`의 API를 호출하는 경우를 위해 설정했습니다. Origin은 프로토콜, 호스트, 포트의 조합으로 구분되므로 두 주소는 포트가 달라 서로 다른 origin입니다. CORS는 인증 기능이 아니라 브라우저가 다른 origin의 응답을 읽을 수 있는지를 제어하는 정책입니다.
  {{< /faq >}}

{{< faq summary="파일 업로드 중 `ENOENT` 또는 `Unexpected field` 오류가 발생하는 이유는 무엇인가요?" >}}

- 이 예제처럼 Multer의 `destination`을 함수로 지정한 경우에는 `uploads` 폴더를 미리 생성해야 합니다. 폴더가 없다면 `mkdir uploads`로 생성한 뒤 서버를 실행합니다.
- `upload.single("uploadFile")`의 인자는 서버가 받을 파일 필드의 이름입니다. 클라이언트에서도 `formData.append("uploadFile", mockFile)`처럼 같은 이름을 사용해야 하며, 다른 이름을 사용하면 `Unexpected field` 오류가 발생할 수 있습니다.
  {{< /faq >}}

{{< faq summary="`res.sendFile()`은 파일 전체를 메모리에 올린 뒤 전송하나요?" >}}

- 아니요. `res.sendFile()`은 파일을 스트리밍 방식으로 전송하므로, `fs.readFile()`처럼 파일 전체를 먼저 메모리에 올릴 필요가 없습니다. 따라서 비교적 큰 파일을 다운로드하는 API에도 사용할 수 있습니다.
- 이 예제에서는 `root` 옵션을 `uploads` 폴더로 지정하여 요청한 경로가 해당 폴더 밖으로 벗어나지 못하도록 제한합니다. 다만 클라이언트의 `response.text()`는 응답 전체를 받은 뒤 문자열로 변환하므로, 해당 코드는 스트리밍 수신이 아니라 응답 본문의 내용을 확인하는 용도입니다.
  {{< /faq >}}

{{< faq summary="이 예제 서버를 실제 서비스에 그대로 사용해도 되나요?" >}}

- 이 서버는 Express의 기본 기능을 확인하기 위한 테스트용 예제입니다. 특히 `/auth`는 `Authorization` 헤더가 있는지만 확인할 뿐, 토큰의 유효성을 검증하거나 사용자의 권한을 확인하지 않습니다.
- 실제 서비스에서는 인증과 권한 검사, 요청 데이터 검증, 업로드 파일의 크기와 형식 제한, 안전한 파일명 생성, HTTPS, 요청 제한, 로깅 및 공통 오류 처리 등을 추가해야 합니다.
  {{< /faq >}}
