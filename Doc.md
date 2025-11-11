Perfect — you’re asking for exactly what a real **internal design document / architecture spec** would contain 👏

Below is a **complete documentation draft** for your **Automated Password Rotation System** — written in a professional structure (you can use this for internal proposals or implementation notes).

---

# 🔐 **Automated Password Rotation System – Architecture & Design Document**

---

## 📘 **1. Project Overview**

This project aims to automate **password rotation** across the organization’s AWS and database ecosystem, ensuring compliance with internal security policies and reducing manual operational overhead.
Whenever a service (like DynamoDB, EC2, or application database) is created, credentials are stored securely in **Vault (running inside EKS)** and synchronized with **AWS Secrets Manager**.

The goal of this system is to **automatically detect expiring passwords**, **rotate them securely**, **update all dependent systems**, and **notify relevant users** — using a combination of AWS Lambda, AWS Step Functions, EventBridge, SNS, and KMS.

---

## 🎯 **2. Scope**

The automation covers:

* Detection of secrets nearing expiration (Vault → Scanner Lambda).
* Automatic rotation of expired secrets (Secrets Manager → Rotation Lambda).
* Update of new passwords across all target systems (databases, apps, etc.).
* Verification of rotated credentials (Verification Lambda).
* Centralized orchestration and logging using Step Functions.
* Notifications and audit via SNS and PostgreSQL.
* Secure storage and encryption of credentials using KMS and Vault.

Out of scope:

* Vault deployment or configuration in EKS (already set up in the organization).
* Manual credential usage flows — those remain unchanged.

---

## 🧩 **3. High-Level Architecture**

```
          ┌──────────────────────────────┐
          │       EventBridge Rule       │
          │  (Triggers daily/weekly scan)│
          └──────────────┬───────────────┘
                         │
                         ▼
                 ┌───────────────────┐
                 │ Lambda #1: Scanner│
                 │ - Scans Vault for │
                 │   expiring secrets│
                 │ - Sends alert or  │
                 │   triggers StepFn │
                 └──────────┬────────┘
                            │
                            ▼
                ┌──────────────────────────┐
                │ AWS Step Function         │
                │ - Orchestrates rotation   │
                │ - Calls Lambda #2 → #3    │
                └──────────┬────────────────┘
                           │
     ┌─────────────────────┴─────────────────────┐
     ▼                                           ▼
┌───────────────┐                         ┌────────────────┐
│Lambda #2:     │                         │Lambda #3:      │
│Rotation Engine│                         │Verification    │
│- Delete old    │                         │- Verify rotated│
│  Vault secret  │                         │  credentials   │
│- Generate new  │                         │  in all systems│
│  password (SM) │                         │- Return status │
│- Update Vault  │                         └──────┬─────────┘
│- Update DBs    │                                │
│- Store metadata│                                │
└──────┬─────────┘                                │
       ▼                                          ▼
        ┌──────────────────────────────────────┐
        │     SNS Notifications (Success/Fail)  │
        │     PostgreSQL Audit Logs             │
        └──────────────────────────────────────┘
```

---

## 🧠 **4. Component-Level Description**

### 🟩 **1. EventBridge**

EventBridge acts as the **scheduler and event trigger** for the entire rotation process.
It triggers **Lambda #1 (Scanner Lambda)** periodically — typically once every 24 hours or based on organizational policy.
It ensures no manual intervention is required for initiating the password checks.

**Responsibilities:**

* Schedule recurring runs.
* Trigger Lambda #1 based on time-based rules.
* Ensure reliable event delivery to Lambda.

---

### 🟦 **2. Lambda #1 — Scanner Lambda**

This Lambda is the **entry point of the workflow**.
It connects to the **Vault running in EKS** and scans all secrets stored within, checking two main conditions:

1. If the password is **about to expire** (e.g., 7 days before expiry).
   → Sends an **alert notification** via SNS to the secret owner.
2. If the password **has reached its expiry date**.
   → Triggers the **Step Function** to begin the automated rotation process.

**Responsibilities:**

* Authenticate to Vault using AppRole or service account.
* Retrieve secret metadata (name, path, creation/expiry date).
* Compare expiry dates.
* Send SNS alerts or trigger Step Function accordingly.
* Log scan results to CloudWatch.

---

### 🟨 **3. AWS Step Function**

Step Function acts as the **orchestration layer** for the password rotation workflow.
It defines and controls the sequence of Lambda executions and failure handling.

**Flow:**

1. Receive payload from Lambda #1 (secret details).
2. Invoke **Lambda #2 (Rotation Engine)**.
3. On success, invoke **Lambda #3 (Verification Lambda)**.
4. Based on verification output:

   * Send **success notification** (password rotation successful).
   * Send **failure notification** (rotation/verification failed).

**Responsibilities:**

* Manage the rotation workflow.
* Provide retry, timeout, and rollback logic.
* Maintain full execution logs and history.

---

### 🟧 **4. Lambda #2 — Rotation Engine**

This is the **core Lambda function** responsible for executing the rotation itself.
It handles the full lifecycle of the secret — from deletion of the old password to propagation of the new one across all systems.

**Responsibilities:**

1. **Delete old secret** from Vault (or mark inactive).
2. **Generate new password** using AWS Secrets Manager (`GenerateRandomPassword`).
3. **Update Vault** with the new password.
4. **Reset password in all connected databases** (PostgreSQL, MySQL, etc.) by connecting via admin accounts and issuing password reset queries.
5. **Encrypt and store password metadata** in PostgreSQL (encrypted with AWS KMS).
6. Return success/failure status to Step Function.

**Security Considerations:**

* Uses AWS KMS to encrypt credentials before storing.
* Uses least-privileged IAM roles for Vault and database access.
* Logs all rotation actions for auditing.

---

### 🟪 **5. Lambda #3 — Verification Lambda**

This Lambda ensures the new credentials are valid and functional.
It runs **immediately after Lambda #2** (via Step Function) and tests the updated credentials against all systems that were updated.

**Responsibilities:**

* Retrieve new password from Vault.
* Connect to each target database/system using new credentials.
* Perform a simple query or authentication test.
* Return consolidated result to Step Function (Success or Failure).
* Write verification logs to CloudWatch and PostgreSQL.

**If verification fails:**

* Step Function sends a “Rotation Failed” SNS alert.
* Optionally trigger a rollback or manual review.

---

### 🟫 **6. Vault (Running in EKS)**

Vault serves as the **central credential store** and is already integrated with organizational applications.
It holds credentials for all AWS and non-AWS services.

**Responsibilities:**

* Store service credentials securely.
* Provide versioning and metadata (creation, expiry).
* Allow read/write/delete operations via authenticated Lambdas.
* Act as the source of truth for all service passwords.

**Integration Notes:**

* Lambdas authenticate to Vault using AppRole or Kubernetes service accounts.
* Vault policies restrict access to only required secrets.

---

### 🟥 **7. Databases (PostgreSQL, MySQL, etc.)**

Each target database holds user credentials that are rotated as part of this automation.
These databases are updated by **Lambda #2** using an admin or rotation-specific role.

**Responsibilities:**

* Accept password reset commands from Lambda #2.
* Maintain audit logs of user credential changes.
* Reflect new credentials for continued service access.

---

### 🟧 **8. SNS (Simple Notification Service)**

SNS acts as the **notification and alerting mechanism** across the system.
It is used by Lambda #1, Lambda #2, and Step Function for various alerts.

**Notification Types:**

* **Warning Alert:** Secret about to expire (from Lambda #1).
* **Success Alert:** Password rotated and verified successfully (from Step Function).
* **Failure Alert:** Rotation or verification failed (from Step Function).

Notifications can be configured to send emails, Slack messages, or PagerDuty alerts to the relevant team or service owner.

---

### 🟨 **9. PostgreSQL (Audit Database)**

This database serves as the **audit and metadata store** for all password rotation activities.
It does **not** store raw passwords — only encrypted values and metadata.

**Responsibilities:**

* Store encrypted password blobs (via KMS).
* Store rotation history, timestamps, and version info.
* Enable reporting and compliance checks.

**Table Example:**

| Secret_Name | Rotation_Date | Expiry_Date | Status | Encrypted_Password | Rotated_By |
| ----------- | ------------- | ----------- | ------ | ------------------ | ---------- |

---

### 🟦 **10. AWS KMS (Key Management Service)**

KMS provides **encryption and decryption** services for sensitive data such as passwords.
It is used within Lambda #2 before writing credentials to PostgreSQL and optionally before updating Vault.

**Responsibilities:**

* Encrypt/decrypt sensitive data.
* Manage rotation keys for encryption.
* Enforce access control using KMS key policies.

---

## ⚙️ **5. Security Controls**

* **IAM Roles:** Each Lambda uses a unique, least-privileged role.
* **Vault AppRole:** Secure authentication mechanism for Vault.
* **KMS Encryption:** Used for encrypting passwords before storage.
* **CloudWatch Logs:** Every Lambda execution is logged for auditing.
* **SNS Alerts:** Immediate visibility for failures or anomalies.
* **Step Function History:** Centralized execution trace for compliance.

---

## 🧩 **6. Future Enhancements**

* Add rollback Lambda to revert to last known good password in case of verification failure.
* Integrate with Slack or ServiceNow for incident creation.
* Build dashboard for rotation status monitoring.
* Integrate AWS Config for compliance monitoring.

---

## ✅ **7. Summary**

| Component         | Description                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------- |
| **Lambda #1**     | Scans Vault, detects expiring secrets, sends alerts, triggers Step Function               |
| **Lambda #2**     | Deletes old passwords, generates new ones, updates Vault & all databases, stores metadata |
| **Lambda #3**     | Verifies new credentials in all systems                                                   |
| **Vault (EKS)**   | Central secret store                                                                      |
| **EventBridge**   | Scheduler and trigger for periodic scans                                                  |
| **Step Function** | Orchestrates rotation and verification workflow                                           |
| **SNS**           | Sends alerts and rotation status                                                          |
| **PostgreSQL**    | Stores audit metadata and encrypted password entries                                      |
| **KMS**           | Encrypts passwords and sensitive data                                                     |

---

Would you like me to turn this into a **formatted technical document (PDF or DOCX)** that you can share internally (with cover page, diagrams, and index)?
It’ll look like a real AWS architecture proposal document.
