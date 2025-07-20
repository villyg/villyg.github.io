---
layout: post
title: "Creating, Signing, and Validating JWTs with Public-Private Key Pairs Using JWKS in Node.js - Part 1"
date: 2025-07-17 17:37:16 -0400
categories: [nodejs, security, oauth]
permalink: /posts/:title
---

## Introduction

This blog post walks you through building a system to create, sign, and validate JSON Web Tokens (JWTs) using public-private key pairs and JSON Web Key Sets (JWKS) in Node.js. We'll use Node/Express for the server framework, Jose for JWT signing and validation, and Axios for HTTP requests. The system is composed of four microservices, orchestrated with Docker Compose for easy setup and execution.

The codebase includes:

1. Bootstrapper: Generates a public-private key pair and a JWKS.
2. Auth Server: Exposes routes to serve the JWKS and issue signed JWTs.
3. API Server: Protects a route, validating JWTs using the Auth server's JWKS.
4. Client: Requests a token from the Auth server and uses it to access the API's protected resource.

Below are step-by-step instructions, complete with sample code, to set up and run this system.

### Prerequisites

- Node.js (v18 or higher)
- Docker and Docker Compose
- Basic understanding of JWTs, public-private key cryptography, and Node.js.

## Step 1: Project Setup

Create a project directory with the following structure (files will be added in the steps below):

```
node-jwks-demo/
├── bootstrapper/
│   ├── index.js
│   └── package.json
├── auth/
│   ├── index.js
│   └── package.json
├── api/
│   ├── index.js
│   └── package.json
├── client/
│   ├── index.js
│   └── package.json
├── docker-compose.yml
└── keys/
    ├── private.pem
    ├── public.pem
    └── jwks.json

```

We'll use Docker Compose to run all services. The `keys/` directory will store the generated keys and JWKS.

## Step 2: Bootstrapper - Generating Keys and JWKS

The Bootstrapper generates an RSA key pair and a JWKS file using the `jose` library.

### Code for Bootstrapper

Create `bootstrapper/index.js`:

```javascript
import { writeFileSync } from "fs";
import { generateKeyPair, exportSPKI, exportPKCS8, exportJWK } from "jose";

// Define the path where keys will be saved
const path = "../keys/";

async function generateAndSaveKeys() {
  // Generate a key pair (RSA or EC — we'll use RSA here)
  const { publicKey, privateKey } = await generateKeyPair("RS256", {
    modulusLength: 2048, // recommended key size
    extractable: true,
  });

  // Export the public/private keys to PEM format
  const publicPem = await exportKeyToPEM(publicKey);
  const privatePem = await exportKeyToPEM(privateKey);

  // Export the public key to JWK format
  const jwk = await exportKeyToJWK(publicKey);

  // 4. Optionally add desired metadata
  jwk.use = "sig";
  jwk.alg = "RS256";
  jwk.kid = "my-key-id"; // optional, but recommended

  // Wrap in JWKS format
  const jwks = {
    keys: [jwk],
  };

  // Write keys to files
  writeFileSync(`${path}public_key.pem`, publicPem);
  writeFileSync(`${path}private_key.pem`, privatePem);
  writeFileSync(`${path}jwks.json`, JSON.stringify(jwks));

  console.log(
    `✅ Keys generated and saved to ${path} as public_key.pem and private_key.pem`,
  );
}

async function exportKeyToPEM(key) {
  const pem = await exportSPKI(key).catch(() => exportPKCS8(key));
  return pem;
}

async function exportKeyToJWK(key) {
  const jwk = await exportJWK(key);
  return jwk;
}

generateAndSaveKeys().catch(console.error);

```

Create `bootstrapper/package.json`:

```json
{
  "name": "auth",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^5.1.0",
    "jose": "^6.0.12"
  },
  "scripts": {
    "start": "node index.js"
  },
  "type": "module"
}
```

### Explanation

- The `jose` library generates an RSA key pair with the RS256 algorithm.
- The public and private keys are exported in PEM format and saved to `keys/`.
- The public key is converted to a JWK and wrapped in a JWKS object with a `kid` (key ID) for identification.

## Step 3: Auth Server - Serving JWKS and Signing JWTs

The Auth server provides two endpoints:

- `/jwks.json`: Returns the JWKS.
- `/token`: Signs and returns a JWT.

### Code for Auth Server

Create `auth/index.js`:

```javascript
import jwksjson from "../keys/jwks.json" with { type: "json" };
import express from "express";
import * as jose from "jose";
import fs from "fs";

const app = express();
const port = 3001;

const pathToPrivateKeyFile = new URL("../keys/private_key.pem", import.meta.url)
  .pathname;
const privateKeyFile = fs.readFileSync(pathToPrivateKeyFile, "utf8");

app.get("/token", async (req, res) => {
  try {
    const privateKey = await getPrivateKey();

    const token = await new jose.SignJWT({ user: "exampleUser" }) // payload
      .setProtectedHeader({ alg: "RS256" })
      .setIssuedAt()
      .setExpirationTime("2h")
      .sign(privateKey);

    res.json({ token: token });
  } catch (err) {
    console.error("Error generating token:", err);
    res.status(500).json({ error: "Token generation failed" });
  }
});

// Convert PEM to CryptoKey using jose
async function getPrivateKey() {
  return await jose.importPKCS8(privateKeyFile, "RS256");
}

/**
 * Endpoint to serve the JWKS (JSON Web Key Set) for public key retrieval.
 * This is typically used for verifying JWTs (JSON Web Tokens).
 */
app.get("/.well-known/jwks.json", (req, res) => {
  res.setHeader("Content-Type", "application/json");
  res.send(jwksjson);
});

app.listen(port, () => console.log(`Auth server running on port ${port}`));
```

Create `auth/package.json`:

```json
{
  "name": "auth",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^5.1.0",
    "jose": "^6.0.12"
  },
  "scripts": {
    "start": "node index.js"
  },
  "type": "module"
}
```

### Explanation

- The `/.well-known/jwks.json` endpoint serves the JWKS file generated by the Bootstrapper.
- The `/token` endpoint signs a JWT with the private key, including a subject (`sub`), scope, and expiration.
- The `kid` in the JWT header matches the JWKS for validation.

## Step 4: API Server - Protecting a Route with JWT Validation

The API server validates incoming JWTs using the Auth server's JWKS.

### Code for API Server

Create `api/index.js`:

```javascript
import express from "express";
import * as jose from "jose";

const app = express();
const port = 3002;
const JWKS_URL = "http://auth:3001/.well-known/jwks.json";
const JWKS = jose.createRemoteJWKSet(new URL(JWKS_URL));

app.get("/protected", async (req, res) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res
      .status(401)
      .json({ error: "Missing or invalid Authorization header" });
  }

  const token = authHeader.split(" ")[1];
  try {
    const { payload } = await jose.jwtVerify(token, JWKS, {
      algorithms: ["RS256"],
    });
    res.json({ message: "Protected resource accessed", user: payload.sub });
  } catch (error) {
    console.error("JWT verification error:", error);
    res.status(401).json({ error: "Invalid or expired token" });
  }
});

app.listen(port, () => console.log(`API server running on port ${port}`));
```

### Explanation

- The `/protected` endpoint checks for a Bearer token in the Authorization header.
- The `jose` library's `createRemoteJWKSet` fetches the JWKS from the Auth server to validate the token.
- If valid, the endpoint returns a success message with the token's subject.

## Step 5: Client - Fetching Token and Accessing Protected Resource

The Client requests a token from the Auth server and uses it to access the API's protected route.

### Code for Client

Create `client/index.js`:

```javascript
import axios from "axios";

async function runClient() {
  try {
    // Get token from Auth server
    const tokenResponse = await axios.get("http://auth:3001/token");
    const token = tokenResponse.data.token;

    // Use token to access protected API
    const apiResponse = await axios.get("http://api:3002/protected", {
      headers: { Authorization: `Bearer ${token}` },
    });

    console.log("API Response:", apiResponse.data);
  } catch (error) {
    console.error(
      "Client error:",
      error.response ? error.response.data : error.message,
    );
  }
}

runClient();
```

Create `client/package.json`:

```json
{
  "name": "client",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "axios": "^1.6.7"
  },
  "scripts": {
    "start": "node index.js"
  }
}
```

### Explanation

- The Client uses `axios` to fetch a JWT from the Auth server's `/token` endpoint.
- It then sends the token in the Authorization header to the API's `/protected` endpoint.

## Step 6: Docker Compose Setup

Create `docker-compose.yml` in the root directory to orchestrate the services:

```yaml
version: "3.8"
services:
  bootstrapper:
    build: ./bootstrapper
    volumes:
      - ./keys:/app/keys
    command: npm start
  auth:
    build: ./auth
    ports:
      - "3001:3001"
    volumes:
      - ./keys:/app/keys
    depends_on:
      - bootstrapper
    command: npm start
  api:
    build: ./api
    ports:
      - "3002:3002"
    volumes:
      - ./keys:/app/keys
    depends_on:
      - auth
    command: npm start
  client:
    build: ./client
    depends_on:
      - api
    command: npm start
```

### Explanation

- Each service is built from its respective directory.
- The `keys/` directory is shared as a volume to persist keys and JWKS.
- The `depends_on` ensures the Bootstrapper runs first, followed by the Auth server, API, and Client.

## Step 7: Running the System

- Ensure Docker is running.
- In the `jwt-demo/` directory, run:

```bash
docker-compose up --build
```

### Expected Output

- The Bootstrapper logs: `Keys and JWKS generated successfully`.
- The Auth server logs: `Auth server running on port 3001`
- The API server logs: `API server running on port 3002`
- The Client logs: `API Response: { message: 'Protected resource accessed', user: 'user123' }`

## Step 8: Testing Manually

You can test the system using `curl` or a tool like Postman:

1. Get the JWKS:

```bash
curl http://localhost:3001/jwks.json
```

2. Get a token:

```bash
curl http://localhost:3001/token
```

3. Access the protected resource:

```bash
curl -H "Authorization: Bearer <token>" http://localhost:3002/protected
```

## Conclusion

This system demonstrates how to use public-private key pairs and JWKS for secure JWT handling in a Node.js environment. The `jose` library simplifies key management and JWT operations, while Docker Compose ensures easy setup. You can extend this by adding more robust error handling, token refresh mechanisms, or additional protected routes.

For the full codebase, check out the [GitHub repository](https://github.com/villyg/node-jwks-demo).
