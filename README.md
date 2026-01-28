# Microsoft SMTP OAuth Configuration Guide

**Document Version:** 2.0
**JIRA Reference:** STOP-2672(https://spinifexit.atlassian.net/browse/STOP-2672) and STOP-2726(https://spinifexit.atlassian.net/browse/STOP-2726)
**Status:** Final Specification

---

## Overview

Microsoft has officially discontinued Basic Authentication for SMTP. To maintain secure email communication, the Strato system now utilizes **OAuth 2.0 (Modern Authentication)** via the **Client Credentials Flow**. This enables server-to-server email transmission (e.g., system notifications) without requiring a user to be logged in.

### Authentication Flow

The integration uses the **OAuth 2.0 Client Credentials Flow**, designed for server-to-server authentication where the application acts on its own behalf rather than on behalf of a specific user. This flow is ideal for automated email sending scenarios.

**Token Endpoint:**
```
https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
```

**Request Parameters:**
- `grant_type`: `client_credentials`
- `client_id`: Your Application (client) ID
- `client_secret`: Your client secret
- `scope`: `https://outlook.office365.com/.default`

**Token Lifetime:**
- Access tokens are typically valid for 60-90 minutes
- The system automatically refreshes tokens as needed

---

## Prerequisites

Before configuring Microsoft SMTP with OAuth 2.0, ensure you have the following credentials provisioned via Microsoft Azure Active Directory (Azure AD):

| Configuration Key | Description | Required For |
|------------------|-------------|--------------|
| **Client ID** | The Application (client) ID registered in Azure AD | Client Credentials Flow |
| **Tenant ID** | The Directory (tenant) ID for your Microsoft 365 organization | Client Credentials Flow |
| **Client Secret** | A secret key generated from the Azure portal for app authentication | Client Credentials Flow |
| **Hostname** | Microsoft SMTP server endpoint: `smtp.office365.com` | All |
| **Port** | SMTP port with STARTTLS: `587` | All |
| **Username** | Email address for SMTP authentication (e.g., `noreply@yourdomain.com`) | All |
| **Sender Email** | Microsoft 365 mailbox address (typically same as username) | All |

---

## Part 1: Azure AD & Exchange Online Setup

### Step 1: Register an Application in Azure AD

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Navigate to: **Azure Active Directory** > **App registrations** > **New registration**
3. Configure the application:
   - **Name:** `Strato SMTP Sender` (or your preferred name)
   - **Supported account types:** Select "Accounts in this organizational directory only (Single tenant)"
   - **Redirect URI:** Leave blank (not required for Client Credentials Flow)
4. Click **Register**
5. **Important:** Copy and securely store the **Application (client) ID** and **Directory (tenant) ID**

### Step 2: Configure API Permissions

1. In your registered app, navigate to **API permissions**
2. Click **Add a permission**
3. Select **APIs my organization uses**
4. Search for and select **Office 365 Exchange Online**
5. Choose **Application permissions** (not Delegated permissions)
6. Add the following permission:
   - `SMTP.SendAsApp` - Allows the application to send emails as any user
7. Click **Add permissions**
8. **Critical:** Click **Grant admin consent for [Your Organization]** and confirm
   - This step requires Azure AD administrator privileges
   - Without admin consent, the system will return an Unauthorized error

### Step 3: Generate a Client Secret

1. Navigate to **Certificates & secrets** in your app registration
2. Click **New client secret**
3. Provide a description (e.g., "Strato SMTP Secret")
4. Select an expiration period (recommended: 12-24 months or custom)
5. Click **Add**
6. **Important:** Immediately copy the secret **Value** (not the Secret ID)
   - This value is only shown once and cannot be retrieved later
   - Store it securely in a password manager or secrets vault
   - The Secret ID is not used by the code

### Step 4: Grant "Send As" Permission in Exchange Online

This step configures which mailbox the application can send emails from.

#### 4.1: Get the Service Principal Object ID

1. Navigate to **Azure Active Directory** > **Enterprise applications**
2. Search for your application name (e.g., "Strato SMTP Sender")
3. Click on the application
4. In the **Overview** section, copy the **Object ID**
   - **Important:** This is the Service Principal Object ID (found under Enterprise Applications)
   - This is different from the Object ID in App registrations
   - This is required for Exchange permissions

#### 4.2: Install Exchange Online PowerShell Module

Open PowerShell as Administrator and run:

```powershell
# Set execution policy to allow module installation
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

# Install the Exchange Online Management module
Install-Module -Name ExchangeOnlineManagement -Force
```

#### 4.3: Connect to Exchange Online

```powershell
# Connect to Exchange Online (you'll be prompted to sign in)
Connect-ExchangeOnline

# Sign in with an account that has Exchange Administrator privileges
```

#### 4.4: Grant Send As Permission

Run the following command, replacing the placeholders:

```powershell
Add-RecipientPermission -Identity "<mailbox@yourdomain.com>" -Trustee "<Service-Principal-Object-ID>" -AccessRights SendAs -Confirm:$false
```

**Parameters:**
- `Identity`: The email address of the mailbox that will be used to send emails (e.g., `noreply@yourdomain.com`)
- `Trustee`: The Service Principal Object ID from step 4.1
- `AccessRights`: Set to `SendAs` to allow sending emails as this mailbox

**Example:**
```powershell
Add-RecipientPermission -Identity "no.reply@stratohcm.com" -Trustee "a1b2c3d4-e5f6-7890-abcd-ef1234567890" -AccessRights SendAs -Confirm:$false
```

#### 4.5: Verify the Permission

```powershell
# Verify the permission was granted
Get-RecipientPermission -Identity "<mailbox@yourdomain.com>" | Where-Object {$_.Trustee -like "*<Service-Principal-Object-ID>*"}
```

---

## Part 2: Strato System Integration

### Backend Implementation (STOP-2672)

The following components have been updated to support OAuth 2.0:

- **Utility Class:** `MicrosoftOAuthProvider.java` - Uses MSAL (Microsoft Authentication Library) to fetch tokens
- **Mail Logic:** `MailBuilderBean.java` - Updated to handle `sendNonSecure` and `sendSecure` methods with dynamic OAuth token injection
- **Data Model:** `EmailConfig` entity updated with `clientID`, `clientSecret`, `isOauth`, and `tenantId` fields
- **Database Migration:** `prov_email_config` migration script adds OAuth columns

### Step 1: Enable OAuth for SMTP Services

1. Navigate to **Strato** > **User Settings** > **Upgrade Center**
2. Locate and install: **Security Enhancement - OAuth enablement for SMTP services**
3. Wait for the installation to complete
4. Ensure the `prov_email_config` migration script is run across all environments to reflect the new OAuth columns

### Step 2: Configure SMTP Settings in Strato Admin Tool

1. Open the **Strato Admin Tool**
2. Navigate to **SMTP Server Settings**
3. Enable SMTP configuration
4. Fill in the following details:

| Field | Value |
|-------|-------|
| **Authentication Type** | Bearer (OAuth 2.0) |
| **Email Provider** | Microsoft |
| **Hostname** | `smtp.office365.com` |
| **Port** | `587` |
| **Username** | Your Microsoft 365 mailbox address (e.g., `noreply@yourdomain.com`) |
| **Sender Email** | Same as username |
| **Client ID** | Application (client) ID from Azure AD |
| **Tenant ID** | Directory (tenant) ID from Azure AD |
| **Client Secret** | The secret value generated in Azure AD |

5. Click **Test Connection** to verify the configuration
   - This triggers the XOAUTH2 handshake
6. If successful, click **Save**

**Code Reference:** `system.view.js` (line 2105 in `strato-administration`) - UI configuration for SMTP settings

### Step 3: Verify Email Functionality

Test the configuration using one of the following methods:

#### Option 1: Email Template Test
1. Navigate to **Email Templates**
2. Select or create a test template
3. Click **Send Test Email**
4. Verify the email is received

#### Option 2: Workflow Test
1. Create or open a workflow with a **Send Email** step
2. Execute the workflow
3. Verify the email is received in the recipient's inbox

---

## Troubleshooting & Verification

### Token Verification

You can inspect the generated token by logging it (in a dev environment) and pasting it into [jwt.ms](https://jwt.ms):

- **aud (Audience):** Must be `https://outlook.office365.com`
- **roles:** Must include `SMTP.SendAsApp`

### Common Issues and Solutions

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| **535 5.7.3 Authentication unsuccessful** | Incorrect Secret or Tenant ID | Re-generate secret and verify Tenant ID matches the app registration |
| **554 5.2.252 SendAsDenied** | PowerShell step missed | Ensure the `Add-RecipientPermission` command was executed for the specific sender address |
| **Token Acquisition Failure** | Azure Regional Issues | Note: +63 (Philippines) country codes sometimes face SMS verification lag in Azure; use an Authenticator app instead |
| **Access Denied** | Wrong Service Principal Object ID used | Verify you used the Service Principal Object ID (from Enterprise applications), not the App registration Object ID |
| **Emails Not Sending** | Port 587 blocked by firewall | Verify network connectivity to `smtp.office365.com:587` |
| **Authentication Failed** | Admin consent not granted | Re-grant admin consent in Azure AD for API permissions |

---

## Security Best Practices

1. **Secret Management:**
   - Store client secrets in a secure secrets management system
   - Never commit secrets to source control
   - Set calendar reminders before secret expiration

2. **Secret Rotation:**
   - Schedule rotation every 12-24 months
   - Have a documented process for updating secrets in production

3. **Least Privilege:**
   - Do not grant `Mail.Send` (Graph API) if you only need SMTP
   - Use `SMTP.SendAsApp` permission only
   - Only grant SendAs permission to specific mailboxes that need it
   - Use dedicated service accounts for automated emails

4. **Monitoring:**
   - Monitor token acquisition failures
   - Set up alerts for authentication errors
   - Regularly audit Exchange Online permissions

5. **Environment Sync:**
   - Ensure the `prov_email_config` migration script is run across all environments to reflect the new OAuth columns

---

## Client Credentials Flow Diagram

```
┌─────────────┐                                  ┌──────────────────┐
│   Strato    │                                  │   Azure AD       │
│   System    │                                  │   (Microsoft)    │
└──────┬──────┘                                  └────────┬─────────┘
       │                                                  │
       │  1. Request Access Token                        │
       │  POST /oauth2/v2.0/token                        │
       │  - client_id                                    │
       │  - client_secret                                │
       │  - grant_type: client_credentials               │
       │  - scope: https://outlook.office365.com/.default│
       ├────────────────────────────────────────────────>│
       │                                                  │
       │  2. Return Access Token                         │
       │  { access_token, expires_in, ... }              │
       │<────────────────────────────────────────────────┤
       │                                                  │
       │                                                  │
┌──────▼──────┐                                  ┌────────▼─────────┐
│   Strato    │                                  │   Microsoft      │
│   System    │                                  │   SMTP Server    │
└──────┬──────┘                                  └────────┬─────────┘
       │                                                  │
       │  3. Connect to SMTP (STARTTLS)                  │
       ├────────────────────────────────────────────────>│
       │                                                  │
       │  4. AUTH XOAUTH2 <base64(access_token)>         │
       ├────────────────────────────────────────────────>│
       │                                                  │
       │  5. Authentication Success                      │
       │<────────────────────────────────────────────────┤
       │                                                  │
       │  6. Send Email                                  │
       ├────────────────────────────────────────────────>│
       │                                                  │
       │  7. Email Sent Confirmation                     │
       │<────────────────────────────────────────────────┤
       │                                                  │
```

---

## References

- [Azure Portal](https://portal.azure.com)
- [Microsoft Learn: Authenticate an IMAP, POP or SMTP connection using OAuth](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- [Microsoft Identity Platform: Client Credentials Flow](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)
- [MSAL for Java Overview](https://learn.microsoft.com/en-us/azure/active-directory/develop/msal-overview)
- [Exchange Online PowerShell Documentation](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [SpinifexIT Help Center - Security Enhancement](https://help.spinifexgroup.com)
- [SpinifexIT JIRA STOP-2672](https://spinifexgroup.atlassian.net/browse/STOP-2672)

---

**Next Review Date:** July 28, 2026
