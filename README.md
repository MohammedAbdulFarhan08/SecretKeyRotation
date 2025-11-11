EventBridge (Scheduler)
      │
      ▼
Lambda #1 (Scanner)
      │
      ▼
 Step Function ─────────────────────────────┐
      │                                     │
      ▼                                     │
 Lambda #2 (Rotation Engine)               │
      │ ├─ Delete old Vault secret          │
      │ ├─ Generate new password (Secrets Manager)
      │ ├─ Update Vault with new password
      │ ├─ Update all databases using new password
      │ └─ Store encrypted password metadata in PostgreSQL
      │
      ▼
 Lambda #3 (Verification) 
      │ ├─ Connect to all databases and verify new password works
      │
      ▼
   SNS Notifications (Success/Failure) ◄────┘
