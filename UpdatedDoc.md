Perfect 👍 — here’s a **concise version** with **1–2 line summaries** for every component used in your automated password rotation project:

---

## 🔹 **Component Summary**

**1. EventBridge**
Triggers the scanning Lambda on a fixed schedule (e.g., daily) to initiate the password rotation workflow automatically.

**2. Lambda #1 – Scanner Lambda**
Scans Vault for secrets nearing expiration, sends alerts via SNS, and triggers the Step Function if a password has expired.

**3. Step Function**
Acts as the orchestrator that sequentially runs the rotation and verification Lambdas and manages success or failure paths.

**4. Lambda #2 – Rotation Engine**
Deletes old passwords from Vault, generates new ones using AWS Secrets Manager, updates all databases with the new credentials, encrypts them via KMS, and stores metadata in PostgreSQL.

**5. Lambda #3 – Verification Lambda**
Verifies that the rotated credentials work correctly across all target systems and reports the result back to the Step Function.

**6. Vault (in EKS)**
Centralized secret management system where all service credentials are securely stored and version-controlled.

**7. AWS Secrets Manager**
Generates strong, random passwords for rotation and integrates seamlessly with Lambda for secure credential handling.

**8. Databases (PostgreSQL, MySQL, etc.)**
Store application credentials that are updated automatically during rotation by the Rotation Lambda.

**9. PostgreSQL (Audit DB)**
Maintains an encrypted record of password rotation history, metadata, and audit logs for compliance and tracking.

**10. AWS SNS**
Sends alerts and notifications to users or admins for events like upcoming expiry, rotation success, or verification failure.

**11. AWS KMS**
Encrypts and decrypts passwords before storage to ensure all sensitive data remains protected at rest and in transit.

**12. CloudWatch**
Captures logs and metrics from all Lambdas and Step Functions for monitoring, debugging, and audit visibility.

---

Would you like me to include this as a **“Component Summary” section** at the end of the full architecture document (like a one-page overview)?
