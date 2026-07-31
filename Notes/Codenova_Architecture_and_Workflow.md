# Codenova: Architecture & Workflow Guide

This document provides a comprehensive, step-by-step breakdown of the Codenova Cloud-Based Online Code Execution Platform.

---

## 1. System Architecture Overview
Codenova is a full-stack application built for safe, scalable code execution.

- **Frontend (`/frontend`)**: React.js, Monaco Editor, and Tailwind CSS. Provides the UI and communicates with the backend via REST API.
- **Backend (`/backend`)**: Node.js, Express.js, and Sequelize (PostgreSQL). The central brain handling authentication and code execution logic.
- **Infrastructure as Code (`/terraform`)**: Terraform scripts that provision AWS cloud resources (EKS, RDS, ECR).
- **Kubernetes Orchestration (`/k8s`)**: YAML manifests to deploy the platform onto the cluster, including monitoring (Prometheus/Grafana) and logging (ELK/EFK stack).
- **CI/CD Pipeline (`/jenkins` & `Jenkinsfile`)**: Automates testing, building Docker images, and deploying to Kubernetes.

---

## 2. Database Schema
The database uses PostgreSQL, managed via the Sequelize ORM.

1. **User**: Stores basic user account information (ID, username, email, hashed password).
2. **Language**: Stores the supported programming languages and their versions.
3. **Submission**: The central hub linking a user to their code run.
   - *Fields*: `id`, `code` (raw source code), `status` (queued, running, completed, failed).
   - *Relationships*: `userId` (User), `languageId` (Language).
4. **ExecutionResult**: Stores the final outcome of a code run. Strict 1-to-1 relationship with `Submission`.
   - *Fields*: `stdout` (standard output), `stderr` (errors), `executionTime` (ms), `memoryUsed`, `exitCode`.
   - *Relationship*: `submissionId` (Submission).

---

## 3. The Code Execution Engine (Docker SDK)

### How code is sent to the Docker Container
The heavy lifting is done in `backend/services/executionService.js` using the Docker SDK (`dockerode`).
1. **Base64 Encoding**: The raw code string is converted into a Base64 string.
2. **Command Construction**: A shell command is built to decode the Base64 string into a file and execute it.
   - *Example (Python)*: `echo <Base64> | base64 -d > script.py && python3 script.py`
3. **Container Creation**: The SDK requests the Docker daemon to create a container (`docker.createContainer()`) specifying the image (e.g., `python:3.11-slim`), the command, and strict security limits (Memory: 256MB, CPU: 0.5, NetworkMode: 'none').

### How the container destroys itself
The platform ensures no container is left behind to prevent memory leaks.
1. **The Execution Race**: Two timers start simultaneously via `Promise.race()`: a promise waiting for natural completion (`container.wait()`) and a strict 10-second timeout promise.
2. **Timeout Handling**: If the timeout wins, the container is forcefully killed (`container.kill()`).
3. **Guaranteed Cleanup**: Wrapped in a `try...catch...finally` block, the `finally` block executes `await container.remove({ force: true })`, aggressively destroying the container and temporary files regardless of the outcome.

---

## 4. End-to-End Workflow

### Step 1: User Signup
1. **Frontend (`frontend/src/pages/Register.js`)**: User submits their details. Sends a `POST` request to `/api/auth/register`.
2. **Backend (`backend/controllers/authController.js`)**: The `register` function hashes the password using `bcrypt.hash()` and saves the new user to the database (`User.create()`).

### Step 2: User Login
1. **Frontend (`frontend/src/pages/Login.js`)**: User submits credentials. Sends `POST` to `/api/auth/login`.
2. **Backend (`backend/controllers/authController.js`)**: The `login` function verifies the password (`bcrypt.compare()`). If valid, it generates a JSON Web Token (`jwt.sign()`) and sends it back to the client via an HTTP-only cookie.

### Step 3: Code Submission
1. **Frontend (`frontend/src/pages/Editor.js`)**: User clicks "Run". Sends a `POST` request to `/api/submissions` with `languageId` and `code`.
2. **Backend (`backend/controllers/submissionController.js`)**: The `createSubmission` function verifies the JWT, creates a `Submission` record with `status: 'queued'`, asynchronously triggers `executeCode(submission.id)`, and immediately returns the `submissionId` to the frontend (HTTP 201).

### Step 4: Frontend Polling
1. **Frontend (`frontend/src/pages/Editor.js`)**: The UI starts a `setInterval` loop (`pollResult(subId)`), sending periodic `GET` requests to `/api/submissions/${id}` to check if execution is finished.

### Step 5: The Execution Engine (Background)
1. **Preparing & Spawning (`backend/services/executionService.js`)**: 
   - Fetches the submission and updates status to `running`.
   - Spawns the Docker container: `await docker.createContainer(...)` and `await container.start()`.
2. **Running & Monitoring**: Enforces the 10-second timeout race.
3. **Capturing & Cleanup**: 
   - Reads output: `await container.logs({ stdout: true, stderr: true })`.
   - Saves to DB: `ExecutionResult.create(...)` and updates `Submission` status to `completed` or `failed`.
   - Destroys container: `await container.remove({ force: true })`.

### Step 6: Viewing the Results
1. **The Final Fetch (`backend/controllers/submissionController.js`)**: The frontend's polling hits `getSubmission`, sees the `completed` status, and returns the `ExecutionResult`.
2. **UI Update (`frontend/src/pages/Editor.js`)**: The polling loop stops, and `setOutput(res.data.result.stdout)` is called, displaying the final output to the user.
