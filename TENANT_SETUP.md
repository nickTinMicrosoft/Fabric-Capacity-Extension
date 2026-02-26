# Tenant Setup Guide — Fabric Capacity Extension

This guide walks an IT administrator through everything needed to deploy the Fabric Capacity Manager extension in a **new Azure AD tenant**. By the end you will have:

- An Azure AD App Registration configured for the extension
- The extension loaded in Microsoft Edge and authenticated against your tenant

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Azure AD role** | *Application Administrator* (or *Global Administrator*) to create the App Registration |
| **Azure subscription(s)** | One or more subscriptions containing Microsoft Fabric capacities |
| **Azure RBAC** | Users need **Reader** to view capacities; **Contributor** or **Fabric Administrator** to start/stop or change SKU |
| **Browser** | Microsoft Edge (Chromium-based) |
| **Extension files** | A copy of this extension folder (all files including `icon.png`) |

---

## Step 1 — Create the Azure AD App Registration

1. Sign in to the [Azure Portal](https://portal.azure.com).
2. Navigate to **Microsoft Entra ID** → **App registrations** → **+ New registration**.
3. Fill in the form:
   - **Name:** `Fabric Capacity Manager` (or any name you prefer)
   - **Supported account types:** choose one of the following:

     | Option | When to use | App Registration scope |
     |--------|-------------|------------------------|
     | **Accounts in this organizational directory only** *(Single tenant)* | Recommended for most organisations. Restricts sign-in to users in your tenant only. | Use your tenant GUID when registering the app in Azure AD. The extension will automatically detect the tenant from the signed-in user's token. |
     | **Accounts in any organizational directory** *(Multi-tenant)* | Use when users from multiple Azure AD tenants need access. | Select this option in the Azure AD app registration. The extension uses `common` as the initial authority and switches to the actual tenant after login. |

   - **Redirect URI:** leave blank for now (configured in Step 6).
4. Click **Register**.
5. On the **Overview** page, copy the **Application (client) ID** — you will need it in Step 4.

---

## Step 2 — Configure API Permissions

1. In the App Registration, go to **API permissions** → **+ Add a permission**.
2. Add the following **Delegated** permissions:

   | API | Permission | Purpose |
   |-----|-----------|---------|
   | **Azure Service Management** | `user_impersonation` | Manage Fabric capacities via the Azure Management API |
   | **Microsoft Graph** | `User.Read` | Read the signed-in user's display name and tenant info |

   > The scopes `offline_access`, `openid`, and `profile` are requested at runtime and do **not** need to be registered here.

3. If your tenant's Azure AD policies require it, click **Grant admin consent for \<your tenant\>**.

---

## Step 3 — Enable Public Client Flows

The extension uses the **PKCE authorization-code flow** (no client secret), which requires public-client support.

1. Go to **Authentication** → scroll to **Advanced settings**.
2. Set **Allow public client flows** to **Yes**.
3. Click **Save**.

---

## Step 4 — Configure the Extension Client ID at Runtime

You no longer need to edit source code. The extension provides a built-in **Settings panel** for entering the Client ID at runtime.

1. Load the extension in Edge (see Step 5).
2. Click the extension icon in the toolbar to open the popup.
3. Click the **⚙ (Settings)** button next to the refresh button in the header.
4. Paste the **Application (client) ID** you copied in Step 1 into the **App Registration Client ID** field.
5. Click **Save**. The extension will log out and apply the new Client ID on the next login.

> **Note:** The tenant ID is automatically detected from the signed-in user's token — you do not need to configure it manually. The extension uses `common` as the authority for the initial login, then switches to the user's actual tenant ID for subsequent token operations.

> **Tip:** The Client ID is stored in the extension's local storage, so it persists across popup sessions and browser restarts.

---

## Step 5 — Load the Extension in Edge

1. Open Microsoft Edge and navigate to `edge://extensions/`.
2. Enable **Developer mode** (toggle in the lower-left or upper-right corner).
3. Click **Load unpacked** and select the extension folder.
4. The extension card will appear. **Copy the Extension ID** — a long alphanumeric string shown on the card (e.g. `abcdefghijklmnopqrstuvwxyz012345`).
5. Pin the extension to the toolbar for easy access.

---

## Step 6 — Register the Redirect URI

1. Return to the Azure Portal → your App Registration → **Authentication**.
2. Under **Platform configurations**, click **+ Add a platform** → choose **Single-page application**.
3. Set the **Redirect URI** to:

   ```
   https://<extension-id>.chromiumapp.org/
   ```

   Replace `<extension-id>` with the Extension ID from Step 5. For example:

   ```
   https://abcdefghijklmnopqrstuvwxyz012345.chromiumapp.org/
   ```

   > **Important:** include the trailing `/`.

4. Click **Configure**, then **Save**.

---

## Step 7 — Verify

1. Click the extension icon in the Edge toolbar.
2. Click **Sign In** — an Azure AD login window will appear.
3. Authenticate with a user that has the required Azure RBAC roles.
4. After sign-in, your subscriptions and Fabric capacities should populate automatically.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| **AADSTS700054** — "response_type 'code' is not enabled for the application" or redirect URI mismatch | The redirect URI registered in Azure AD does not match the extension's actual ID. | Double-check the extension ID at `edge://extensions/` and update the redirect URI in the App Registration. Make sure it includes the trailing `/`. |
| **AADSTS65001** — "The user or administrator has not consented to use the application" | Admin consent is required by your tenant's policies. | Have a tenant admin go to **API permissions** and click **Grant admin consent**. |
| **AADSTS7000218** — "The request body must contain … 'client_assertion' or 'client_secret'" | Public client flows are not enabled. | Go to **Authentication → Advanced settings** and set **Allow public client flows** to **Yes** (see Step 3). |
| **No capacities shown after sign-in** | The signed-in user lacks RBAC permissions on the Azure subscriptions. | Assign at least **Reader** on the subscription or resource group containing the Fabric capacities. |
| **Token refresh fails repeatedly** | The refresh token has expired or been revoked (e.g. password change, conditional-access policy). | Click the extension title area **twice quickly** (double-click) to clear the cached tokens, then sign in again. |

---

## Distribution Notes

### Sharing as an unpacked extension
Zip the entire extension folder (including `icon.png`) and share it. The recipient follows Steps 4–7 above.

### Publishing to Edge Add-ons
If you publish the extension through the [Microsoft Edge Add-ons portal](https://partner.microsoft.com/en-us/dashboard/microsoftedge/overview), the Extension ID becomes **stable** and will not change across installations. You can register the redirect URI once and it will work for all users who install from the store.

### Per-tenant App Registration
Each tenant that uses the extension needs its **own** App Registration (Steps 1–3). The `clientId` and `tenantId` in `popup.js` must match that registration. If you need to support multiple tenants from a single build, keep `tenantId` set to `'common'` and use a multi-tenant App Registration.

---

## Required Azure RBAC Roles (Summary)

| Action | Minimum Role |
|--------|-------------|
| View capacities | **Reader** |
| Start / Stop capacities | **Contributor** |
| Change SKU | **Contributor** or **Fabric Administrator** |

---

## Reference: Extension Permissions

The extension's `manifest.json` requests the following:

| Permission | Purpose |
|-----------|---------|
| `identity` | OAuth2 authentication via `chrome.identity.launchWebAuthFlow()` |
| `storage` | Token caching and user preferences in `chrome.storage.local` |
| `activeTab` | Extension popup functionality |
| `scripting` | Extension operations |

### Host permissions

| Host | Purpose |
|------|---------|
| `https://management.azure.com/*` | Azure Management REST API |
| `https://login.microsoftonline.com/*` | Azure AD OAuth2 endpoints |
| `https://graph.microsoft.com/*` | Microsoft Graph API |

No data leaves the browser except to these three Microsoft-owned endpoints.
