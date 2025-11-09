# n8n Data Validation & HubSpot Sync Workflow

> Automated workflow for validating Google Sheets data, routing approvals via Slack, and syncing approved contacts to HubSpot CRM with comprehensive audit logging.

[![n8n](https://img.shields.io/badge/n8n-automation-EA4B71?logo=n8n)](https://n8n.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

<img width="1816" height="444" alt="image" src="https://github.com/user-attachments/assets/9d551f99-5270-4bee-b69f-f69ed165be48" />

**📹 [Watch Video Demo](https://drive.google.com/file/d/1gOCND-gfDg-_9vJfaTy2g20FtfprivQ2/view?usp=sharing)** - Full explanation and testing walkthrough

---

## 🎯 Overview

This n8n workflow automates a complete data validation and CRM synchronization pipeline that:

- ✅ Validates incoming Google Sheets data (email format, required fields, duplicate detection)
- 🔄 Handles Google Sheets synchronization lag with intelligent retry logic
- 📊 Processes multiple simultaneous row additions via batch processing
- 💬 Routes valid entries through Slack approval workflow
- 🔗 Syncs approved contacts to HubSpot (create or update)
- 📝 Maintains immutable audit logs with timestamps
- 📧 Sends detailed HTML email summaries for all outcomes

---

## 🏗️ Architecture

```
Google Sheets Trigger
         ↓
    Get All Rows
         ↓
   Pre-Loop Check (handles Transaction_ID lag)
         ↓
    Wait 10s (if needed) → Re-check data
         ↓
    New Data Filter
         ↓
   Loop Over Items ←──────────────┐
         ↓                          │
   Prepare Messages                │
         ↓                          │
   Valid or Invalid?               │
    ↙           ↘                  │
Valid          Invalid             │
  ↓               ↓                │
Ask Approval   Send Error         │
  ↓               ↓                │
Approve/Reject  Summary Email      │
  ↓                                │
Search HubSpot                     │
  ↓                                │
Create/Update Contact              │
  ↓                                │
Update Sheet Status                │
  ↓                                │
Log to Audit Sheet                 │
  ↓                                │
Summary Email ─────────────────────┘
```

---

## 🚀 Key Features

### 1. **Smart Lag Handling**
Solves the Google Sheets synchronization delay problem:
- Detects missing Transaction_ID on initial trigger
- Waits 10 seconds and re-checks data
- Prevents false validation errors from timing issues
- Marks as invalid only after confirmation

### 2. **Batch Processing**
Handles multiple simultaneous row additions:
- Filters all unprocessed/incomplete rows
- Loops through each row individually
- Each entry gets full validation → approval → sync cycle

### 3. **Comprehensive Validation**
- **Name**: Non-empty check
- **Email**: RFC 5322 format validation
- **Transaction_ID**: Non-empty + uniqueness check across all rows

### 4. **HubSpot Integration**
- Search-before-write pattern prevents duplicates
- Creates new contacts when email doesn't exist
- Updates existing contacts when found
- Syncs custom properties: `full_name`, `transaction_id`, `n8n_workflow`

### 5. **Audit Trail**
- Separate immutable log sheet
- Records: Transaction_ID, Name, Email, Status, ISO Timestamp
- Tracks all approval/rejection decisions

---

## 📋 Prerequisites

### Required Accounts
- n8n instance (cloud or self-hosted)
- Google Account (for Sheets)
- Slack workspace with bot permissions
- HubSpot account with API access
- Gmail account (for notifications)

### Google Sheets Structure

**Main Sheet** (e.g., "Data Entry"):
| Name | Email | Transaction_ID | Status | row_number |
|------|-------|----------------|--------|------------|
| John Doe | john@example.com | TXN001 | Approved | 2 |

**Audit Sheet** (e.g., "Audit Log"):
| Transaction_ID | Name | Email | Status | Timestamp | row_number |
|----------------|------|-------|--------|-----------|------------|
| TXN001 | John Doe | john@example.com | Approve | 2024-11-08T12:30:00Z | 2 |

### HubSpot Custom Properties
Create these in **Contacts** object:
```
- full_name (Single-line text)
- transaction_id (Single-line text)
- n8n_workflow (Single-line text)
```

---

## ⚙️ Setup Instructions

### 1. Import Workflow
```bash
# In n8n interface:
1. Click "Import from File" or "Import from URL"
2. Select workflow.json
3. Click "Import"
```

### 2. Configure Credentials

Create and connect the following credentials in n8n:

| Credential Type | Used By | Permissions Needed |
|----------------|---------|-------------------|
| Google Sheets Trigger OAuth2 | Trigger node | Read access to sheets |
| Google Sheets OAuth2 | Read/Write nodes | Read & Write access |
| Slack API | Messages & Approval | `chat:write`, `commands` |
| HTTP Header Auth | HubSpot nodes | Private App token with Contacts write scope |
| Gmail OAuth2 | Summary email | Send email permission |

**HubSpot Authentication Setup:**
1. Create Private App in HubSpot
2. Grant `crm.objects.contacts.write` and `crm.objects.contacts.read` scopes
3. Copy Access Token
4. In n8n, create "HTTP Header Auth" credential:
   - Name: `Authorization`
   - Value: `Bearer YOUR_ACCESS_TOKEN`

### 3. Update Workflow Configuration

Replace placeholder values in these nodes:

**Google Sheets (all nodes):**
- `documentId`: Your main Google Sheet ID
- `sheetName`: Your sheet name (usually "Sheet1")

**Audit Sheet (Log the status node):**
- `documentId`: Your audit log Google Sheet ID

**Slack (Invalid Data, Ask Approval):**
- `channelId`: Your Slack channel ID (find in channel settings)

**Email (Summary node):**
- `sendTo`: Your email address

### 4. Activate Workflow
1. Click the "Active" toggle in top-right
2. Workflow will poll Google Sheet every minute
3. Add test data to verify setup

---

## 🧪 Testing Scenarios

### Test 1: Valid New Contact
```
Name: John Doe
Email: john.doe@example.com
Transaction_ID: TXN001
```
**Expected Flow:**
1. ✅ Validation passes
2. 💬 Slack approval request sent
3. ✔️ Approve in Slack
4. 🔗 HubSpot contact created
5. 📝 Audit log updated
6. 📧 Success email received

### Test 2: Invalid Email Format
```
Name: Jane Smith
Email: not-an-email
Transaction_ID: TXN002
```
**Expected:** Validation fails → Slack error notification → Summary email (no approval)

### Test 3: Duplicate Transaction_ID
```
Name: Bob Johnson
Email: bob@example.com
Transaction_ID: TXN001 (already exists)
```
**Expected:** Validation fails → Error notification → No approval

### Test 4: Missing Transaction_ID (Lag Scenario)
```
Step 1: Add row without Transaction_ID
Step 2: Wait for trigger (workflow marks as "Waiting...")
Step 3: Add Transaction_ID within 10 seconds
```
**Expected:** Status changes from "Waiting..." to empty → Proceeds to validation

### Test 5: Bulk Addition
Add 3 rows simultaneously with different Transaction_IDs

**Expected:** All 3 rows processed individually with separate approval requests

### Test 6: Existing Contact Update
```
Email: existing@hubspot.com (already in HubSpot)
```
**Expected:** HubSpot contact updated instead of created

### Test 7: Rejection Workflow
Add valid row → Receive Slack approval → Select "Reject"

**Expected:** Status = "Rejected" → Logged → Summary email (no HubSpot sync)

---

## 🔧 Troubleshooting

| Issue | Solution |
|-------|----------|
| **Workflow doesn't trigger** | • Check Google Sheets OAuth credentials<br>• Verify polling is set to "Every minute"<br>• Ensure workflow is active |
| **False validation errors** | • Confirm "Wait 10 Seconds" node is enabled<br>• Check Pre-Loop Check logic |
| **Bulk rows not processing** | • Verify "New Data" filter includes all incomplete rows<br>• Check Loop Over Items configuration |
| **HubSpot 404 error** | • Ensure custom properties exist in HubSpot<br>• Verify Private App has correct scopes |
| **Approval timeout** | • Check Slack API token permissions<br>• Verify bot is in the channel |
| **Email not sending** | • Confirm Gmail OAuth is configured<br>• Check recipient email address |
| **Duplicate contacts created** | • Verify "Search for Contact" node is executing<br>• Check email field mapping |

---

## 📊 Workflow Metrics

- **Average execution time**: ~15-30 seconds per entry
- **Polling interval**: Every 60 seconds
- **Wait time for lag handling**: 10 seconds
- **Maximum batch size**: Unlimited (processes sequentially)

---

## 🔐 Security Considerations

- ✅ All credentials use OAuth2 or API tokens (no passwords in workflow)
- ✅ Sensitive data only transmitted over HTTPS
- ✅ HubSpot API uses Private App tokens (scoped permissions)
- ✅ Audit logs are append-only (immutable)
- ⚠️ Sheet IDs are visible in JSON (consider as public)
- ⚠️ Slack channel should be private for sensitive approvals

---

## 🛣️ Future Enhancements

- [ ] Add retry logic for HubSpot API failures
- [ ] Implement rate limiting for high-volume scenarios
- [ ] Add conditional approval routing based on transaction amount
- [ ] Create weekly digest emails with approval metrics
- [ ] Replace polling trigger with webhook for real-time processing
- [ ] Add support for bulk approval actions
- [ ] Implement approval expiration/timeout handling

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the [issues page](../../issues).

---

## 👤 Author

**Soso Pkhakadze**

- GitHub: [@sosopkhakadze](https://sosopkhakadze.github.io/my-portfolio/)
- LinkedIn: [soso-pkhakadze-linkedin](https://www.linkedin.com/in/soso-pkhakadze-733428274/)

---

## 🙏 Acknowledgments

- Built for Gamdom Automation Engineer technical assessment
- Powered by [n8n](https://n8n.io/) - Fair-code licensed workflow automation
- Integrations: Google Sheets, Slack, HubSpot, Gmail

---

## 📧 Support

If you have questions or need help setting up the workflow:
1. Check the [troubleshooting section](#-troubleshooting)
2. Watch the [video demo](https://drive.google.com/file/d/1gOCND-gfDg-_9vJfaTy2g20FtfprivQ2/view?usp=sharing)
3. Open an issue in this repository

---

**⭐ If this workflow helped you, please consider giving it a star!**
