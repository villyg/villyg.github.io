---
layout: post
title: "Implementing client credentials grant flow with public/private key pair, client assertion, and JWKS in Node.js"
date: 2025-07-17 17:37:16 -0400
tags: nodejs security oauth
permalink: /posts/:title
---

- [Introduction](#introduction)
- [Step 1: Project Setup](#step-1-project-setup)
- [Step 2: Bootstrapper - Generating Keys and JWKS](#step-2-bootstrapper---generating-keys-and-jwks)
- [Step 3: Auth Server - Serving JWKS and Signing JWTs](#step-3-auth-server---serving-jwks-and-signing-jwts)
- [Step 4: API Server - Protecting a Route with JWT Validation](#step-4-api-server---protecting-a-route-with-jwt-validation)
- [Step 5: Client - Fetching Token and Accessing Protected Resource](#step-5-client---fetching-token-and-accessing-protected-resource)
- [Step 6: Docker Compose Setup](#step-6-docker-compose-setup)
- [Step 7: Running the System](#step-7-running-the-system)
- [Step 8: Testing Manually](#step-8-testing-manually)
- [Conclusion](#conclusion)

## Introduction

This blog post will walk you through an implementation of the `client credentials` grant flow using a `client assertion`. We will build a system to create, sign, and validate JSON Web Tokens (JWTs) using public-private key pairs and JSON Web Key Sets (JWKS) in Node.js. We'll use Node/Express for the server framework, Jose for JWT signing and validation, and Axios for HTTP requests. The system is composed of a bootstrapper and three microservices, orchestrated with Docker Compose for easy setup and execution.

The workflow looks as follows:

<pre class="mermaid">
sequenceDiagram
        participant Client
        participant Auth
        participant Api
        activate Client
        Client->>Client: generate client_assertion
        Client->>Client: sign client_assertion with private key
        Client->>Auth: Exchange client assertion for access_token
        activate Auth        
        Auth->>Auth: Verify client_assertion with public key
        Auth->>Auth: Generate access_token
        Auth->>Auth: Sign access_token with private key
        Auth->>Client: return access_token
        deactivate Auth
        Client->>Api: Get protected resource with access_token
        activate Api
        Api->>Auth: Get public key from JWKS
        activate Auth
        Auth->>Api: return public key
        deactivate Auth
        Api->>Api: Verify access_token with public key
        Api->>Client: Return protected resource
        deactivate Api
        deactivate Client
</pre>
<script src="https://cdn.jsdelivr.net/npm/mermaid@10.9.1/dist/mermaid.min.js"></script>

The codebase includes:

1. Bootstrapper: Generates a public-private key pair and a JWKS.
2. Auth Server: Exposes routes to serve the JWKS and issue signed access tokens.
3. API Server: Protects a route, validating access tokens using the Auth server's JWKS.
4. Client: Signs a client assertion and exchanges it for an access token from the Auth server to access the API's protected resource.


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

Create `bootstrapper/package.json`:

```json
{
  "name": "bootstrapper",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "jose": "^6.0.12"
  },
  "scripts": {
    "start": "node index.js",
    "lint": "eslint ."
  },
  "type": "module",
  "devDependencies": {
    "@eslint/js": "^9.31.0",
    "eslint": "^9.31.0",
    "globals": "^16.3.0",
    "prettier": "3.6.2"
  }
}
```

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
  jwk.kid = "client-key"; // optional, but recommended

  // Wrap in JWKS format
  const jwks = {
    keys: [jwk],
  };

  // Write keys to files
  writeFileSync(`${path}public_key.pem`, publicPem);
  writeFileSync(`${path}private_key.pem`, privatePem);
  writeFileSync(`${path}jwks.json`, JSON.stringify(jwks));

  console.log(
    `✅ Keys generated and saved to ${path} as public_key.pem and private_key.pem`
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

### Explanation

- The `jose` library generates an RSA key pair with the RS256 algorithm.
- The public and private keys are exported in PEM format and saved to `keys/`.
- The public key is converted to a JWK and wrapped in a JWKS object with a `kid` (key ID) for identification.

## Step 3: Auth Server - Serving JWKS and Signing JWTs

The Auth server provides two endpoints:

- `/jwks.json`: Returns the JWKS.
- `/token`: Signs and returns a JWT.

### Code for Auth Server

Create `auth/package.json`:

```json
{
  "name": "auth",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "body-parser": "^2.2.0",
    "express": "^5.1.0",
    "jose": "^6.0.12"
  },
  "scripts": {
    "start": "node index.js"
  },
  "type": "module",
  "devDependencies": {
    "@eslint/js": "^9.31.0",
    "eslint": "^9.31.0",
    "globals": "^16.3.0"
  }
}
```

Create `auth/index.js`:

```javascript
import jwksjson from "../keys/jwks.json" with { type: "json" };
import express from "express";
import * as jose from "jose";
import fs from "fs";
import bodyParser from "body-parser";

const app = express();
const port = 3001;

// create application/x-www-form-urlencoded parser
const urlencodedParser = bodyParser.urlencoded();

const pathToPrivateKeyFile = new URL("../keys/private_key.pem", import.meta.url)
  .pathname;
const privateKeyFile = fs.readFileSync(pathToPrivateKeyFile, "utf8");

const privateKey = await jose.importPKCS8(privateKeyFile, "RS256");

const pathToPublicKeyFile = new URL("../keys/public_key.pem", import.meta.url)
  .pathname;

const publicKeyFile = fs.readFileSync(pathToPublicKeyFile, "utf8");

const publicKey = await jose.importSPKI(publicKeyFile, "RS256");

app.post("/token", urlencodedParser, async (req, res) => {
  console.info("Received request to /token endpoint");

  console.info("Validating request body...");

  // Check to see if the request body is present
  if (!req.body) {
    console.error("Request body is missing");
    return res.status(400).json({ error: "Request body is required" });
  }

  // Check to see if the client_id is present in the request body
  if (!req.body.client_id && req.body.client_id !== "client") {
    // ADD additional check for valid client_id here if needed
    console.error("Missing or invalid client_id in request body");
    return res.status(400).json({ error: "Missing or invalid client_id" });
  }

  // Check to see if the client_assertion_type is present in the request body
  if (
    !req.body.client_assertion_type &&
    req.body.client_assertion_type !==
      "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
  ) {
    console.error("Missing or invalid client_assertion_type in request body");
    return res
      .status(400)
      .json({ error: "Missing or invalid client_assertion_type" });
  }

  // Check to see if the client_assertion is present in the request body
  if (!req.body.client_assertion) {
    console.error("Missing client_assertion in request body");
    return res.status(400).json({ error: "Missing client_assertion" });
  }

  console.info("Request body validation successful");

  const client_assertion = req.body.client_assertion;

  console.log("client_assertion:", client_assertion);

  // Verify the client_assertion
  console.info("Verifying client_assertion...");
  try {
    const payload = await jose.jwtVerify(client_assertion, publicKey, {
      issuer: "client",
      audience: "auth",
    });

    console.info("client_assertion successfully verified");
    console.log("Verified client_assertion payload:", payload);

    console.info("Generating access token...");
    const access_token = await new jose.SignJWT({
      sub: payload.sub,
      user: "exampleUser",
    })
      .setProtectedHeader({ alg: "RS256", kid: "client-key" })
      .setIssuedAt()
      .setIssuer("auth")
      .setAudience("api")
      .setExpirationTime("2h")
      .sign(privateKey);

    console.info("Access token generated successfully");
    console.info("Returning access token to client...");
    res.json({ token: access_token });
  } catch (err) {
    console.error("Invalid client_assertion:", err.message);
    res.status(401).json({ error: "Invalid client_assertion" });
  }
});

/**
 * Endpoint to serve the JWKS (JSON Web Key Set) for public key retrieval.
 * This is typically used for verifying JWTs (JSON Web Tokens).
 */
app.get("/.well-known/jwks.json", (req, res) => {
  res.setHeader("Content-Type", "application/json");
  res.send(jwksjson);
});

app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});

app.listen(port, () => console.log(`Auth server running on port ${port}`));
```


### Explanation

- The `/.well-known/jwks.json` endpoint serves the JWKS file generated by the Bootstrapper.
- The `/token` endpoint verifies a `client assertion` and then signs and returns an `access token` with the private key, including a subject (`sub`), scope, and expiration.
- The `kid` in the JWT header matches the JWKS for validation.

## Step 4: API Server - Protecting a Route with JWT Validation

The API server validates incoming JWTs using the Auth server's JWKS.

### Code for API Server

Create `api/package.json`

```json
{
  "name": "api",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^5.1.0",
    "jose": "^6.0.12"
  },
  "scripts": {
    "start": "node index.js"
  },
  "type": "module",
  "devDependencies": {
    "@eslint/js": "^9.31.0",
    "eslint": "^9.31.0",
    "globals": "^16.3.0"
  }
}

```
Create `api/index.js`:

```javascript
import express from "express";
import * as jose from "jose";

const app = express();
const port = 3002;
const JWKS_URL = "http://auth:3001/.well-known/jwks.json";

console.info("Retrieving JWKS...");
const clientJWKSet = jose.createRemoteJWKSet(new URL(JWKS_URL));
console.info("JWKS successfully retrieved");


app.get("/protected", async (req, res) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res
      .status(401)
      .json({ error: "Missing or invalid Authorization header" });
  }

  const token = authHeader.split(" ")[1];
  try {
    const { payload } = await jose.jwtVerify(token, clientJWKSet, {
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

The Client exchanges a signed `client assertion` for an `access token` from the Auth server and uses it to access the API's protected route.

### Code for Client

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

Create `client/index.js`:

```javascript
import axios from "axios";
import * as jose from "jose";
import fs from "fs";

async function runClient() {
  console.info("Generating client assertion...");

  const pathToPrivateKeyFile = new URL(
    "../keys/private_key.pem",
    import.meta.url
  ).pathname;
  const privateKeyFile = fs.readFileSync(pathToPrivateKeyFile, "utf8");

  const privateKey = await jose.importPKCS8(privateKeyFile, "RS256");

  const now = Math.floor(Date.now() / 1000);

  const clientAssertion = await new jose.SignJWT({ sub: "client" })
    .setProtectedHeader({ alg: "RS256" })
    .setIssuedAt(now)
    .setIssuer("client")
    .setAudience("auth")
    .setExpirationTime("5m")
    .sign(privateKey);

  console.info("Client Assertion generated successfully");
  console.log("Client Assertion:", clientAssertion);

  try {
    console.info(
      "Exchanging client assertion for access_token from Auth server..."
    );

    const tokenResponse = await axios.post(
      "http://auth:3001/token",
      {
        client_assertion: clientAssertion,
        client_assertion_type:
          "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        client_id: "client",
      },
      {
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
      }
    );

    const token = tokenResponse.data.token;

    console.info("Access_token received successfully:");
    console.log("Token:", token);

    console.info("Accessing protected API...");

    const apiResponse = await axios.get("http://api:3002/protected", {
      headers: { Authorization: `Bearer ${token}` },
    });

    console.info("Protected API accessed successfully");

    console.log("API Response received:", apiResponse.data);
  } catch (error) {
    console.error(
      "Client error:",
      error.response ? error.response.data : error.message
    );
  }
}

runClient();
```

### Explanation

- The Client uses `jose` to sign a `client assertion`.
- The Client uses `axios` to exchange the `client assertion` for an `access token` from the Auth server's `/token` endpoint.
- It then sends the `access token` in the Authorization header to the API's `/protected` endpoint.

## Step 6: Docker Compose Setup

Create `docker-compose.yml` in the root directory to orchestrate the services:

```yaml
services:
  bootstrapper:
    build: ./bootstrapper
    volumes:
      - ./keys:/keys
    command: npm start
    profiles:
      - bootstrap
  auth:
    build: ./auth
    ports:
      - "3001:3001"
    volumes:
      - ./keys:/keys
    command: npm start
    profiles:
      - stack
  api:
    build: ./api
    ports:
      - "3002:3002"
    volumes:
      - ./keys:/keys
    depends_on:
      - auth
    command: npm start
    profiles:
      - stack
  client:
    build: ./client
    depends_on:
      - api
    volumes:
      - ./keys:/keys      
    command: npm start
    profiles:
      - stack
```

### Explanation

- Each service is built from its respective directory.
- The `keys/` directory is shared as a volume to persist keys and JWKS.
- The `bootstrap` and `stack` profiles are used to organize the components.

## Step 7: Running the System

- Ensure Docker is running.
- In the `jwt-demo/` directory, run the following command to generate the keys:

```bash
docker-compose --profile bootstrap up --build --force-recreate
```
- In the `jwt-demo/` directory, run the following command to spin up the stack:

```bash
docker-compose --profile stack up --build --force-recreate
```

### Expected Output

- The Bootstrapper logs: `Keys and JWKS generated successfully`.
- The Auth server logs: `Auth server running on port 3001`
- The API server logs: `API server running on port 3002`
- The Client logs: `API Response: { message: 'Protected resource accessed' }`

## Step 8: Testing Manually

You can test the system using `curl` or a tool like Postman:

Get the JWKS:

```bash
curl http://localhost:3001/jwks.json
```

Get a token:

```bash
curl http://localhost:3001/token
```

Access the protected resource:

```bash
curl -H "Authorization: Bearer <token>" http://localhost:3002/protected
```

## Conclusion

This system demonstrates how to use public-private key pairs, JWKS and client assertion for secure JWT handling in a Node.js environment. The `jose` library simplifies key management and JWT operations, while Docker Compose ensures easy setup. You can extend this by adding more robust error handling, token refresh mechanisms, or additional protected routes.

For the full codebase, check out the [GitHub repository](https://github.com/villyg/node-jwks-demo).
