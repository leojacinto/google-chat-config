# Google Chat to ServiceNow Virtual Agent Integration

## What This Is
Google Chat bot connected to ServiceNow Virtual Agent using the **native `sn_va_google_chat` plugin** and the **Now Virtual Agent** Marketplace app.

---

## Prerequisites (done once, do not redo)

### 1. Google Workspace Domain
- Domain: `<YOUR_WORKSPACE_DOMAIN>` (Google Workspace, NOT tested on personal account)
- Admin: `<WORKSPACE_ADMIN_EMAIL>`
- **Why Workspace matters:** Personal Gmail GCP projects lock "Build as Workspace add-on" permanently ON and use a different service account format. A paid Workspace account is required for the self-configured bot path.

### 2. GCP Project
- Project: `<GCP_PROJECT_ID>` (project number `<GCP_PROJECT_NUMBER>`)
- Owner: `<WORKSPACE_ADMIN_EMAIL>`
- APIs enabled:
  - **Google Chat API**
  - **Google Workspace Add-ons API** (creates the inbound service account automatically)

### 3. Now Virtual Agent Marketplace App
- Install from: https://workspace.google.com/marketplace/app/now_virtual_agent/442468716736
- Requires Google Workspace admin access to install for the domain
- This is ServiceNow's official Google Chat bot. Once installed, it appears as "Now Virtual Agent" in Google Chat.
- The self-configured bot (below) is a separate bot that uses your own GCP project and service accounts. Both can coexist.

### 4. GCP Chat API Configuration (self-configured bot)
Navigate: `https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat?project=<GCP_PROJECT_ID>`

| Setting | Value |
|---------|-------|
| App status | LIVE |
| App name | `<YOUR_BOT_NAME>` |
| **Build as Workspace add-on** | **CHECKED** (mandatory, the native SN plugin expects this) |
| Connection | HTTP endpoint URL |
| HTTP endpoint URL | `https://<INSTANCE>.service-now.com/api/sn_va_google_chat/va_google_chat_inbound/events` |
| Triggers | Use a common HTTP endpoint URL for all triggers |
| Service account email | `service-<GCP_PROJECT_NUMBER>@gcp-sa-gsuiteaddons.iam.gserviceaccount.com` |

### 5. GCP Service Accounts
| Purpose | Email |
|---------|-------|
| Inbound (Google to SN) | `service-<GCP_PROJECT_NUMBER>@gcp-sa-gsuiteaddons.iam.gserviceaccount.com` |
| Outbound (SN to Google) | `<OUTBOUND_SA>@<GCP_PROJECT_ID>.iam.gserviceaccount.com` |

### 6. Outbound P12 Key
- File: `<YOUR_P12_FILE>.p12`
- Password: `<P12_PASSWORD>`
- Uploaded to SN as attachment, sys_id: `<P12_ATTACHMENT_SYS_ID>`
- Used for JWT-signed outbound calls to `chat.googleapis.com`

---

## ServiceNow Configuration

### Plugin
- `sn_va_google_chat` - Conversational Integration with Google Chat (pre-installed on instance)

### Bot Installation
Run once via the native API:
```
GET /api/sn_va_google_chat/multi_instance_google_chat_bot_ui/installGoogleChatCustomBot
  /{bot_name}
  /{inbound_service_account_email}
  /{outbound_service_account_email}
  /{private_key_password}
  /{p12_attachment_sys_id}
  /{isUpdate}
```

Example call:
```
/installGoogleChatCustomBot
  /<BOT_NAME>
  /service-<PROJECT_NUMBER>%40gcp-sa-gsuiteaddons.iam.gserviceaccount.com
  /<OUTBOUND_SA>%40<PROJECT_ID>.iam.gserviceaccount.com
  /<P12_PASSWORD>
  /<P12_ATTACHMENT_SYS_ID>
  /false
```

This API creates all required records automatically:
- `sys_cs_provider_application` (bot identity)
- `provider_auth` (inbound JWT validation)
- `oauth_oidc_entity` (OIDC token verification)
- `oidc_provider_configuration` (issuer: `https://accounts.google.com`)
- `oauth_entity` (outbound OAuth provider)
- `jwt_provider` + `jwt_keystore_aliases` (outbound JWT signing)
- `sys_certificate` (P12 keystore)

**If reinstalling:** delete ALL of these records first. The API returns conflicts like "bot already exists" or "failed to create outbound keystore alias" if leftovers exist.

### Key SN Record sys_ids
| Record | Table | sys_id |
|--------|-------|--------|
| VA Google Chat Provider | `sys_cs_provider` | *(query by name)* |
| Google Chat Channel | `sys_cs_channel` | *(query by name)* |
| P12 Attachment | `sys_attachment` | *(uploaded during setup)* |

### Topic Configuration
For a topic to work via Google Chat for unauthenticated (Guest) users:

1. **Roles must include `public`** - set on BOTH `sys_cs_topic.roles` AND `sys_cb_topic.roles`
2. **Republish the topic** from VA Designer after changing roles. The `published_definition` JSON blob is a snapshot regenerated only on publish. REST/GlideRecord writes to the topic record do NOT update it.
3. Topic must be `active=true`, `published=true`, `discoverable=true`
4. `model_type=nlu_keyword` enables both NLU and keyword matching

### Account Linking (Guest to SN User)
The native adapter creates Guest consumers by default. The `sys_cs_channel_user_profile` and `sys_cs_consumer` records need the `user` field set to link to an SN user.

**Critical:** REST Table API writes to `sn_cs` scoped tables are **silently stripped**. To write these fields, create a temporary `sys_ws_operation` under the `sn_va_google_chat` scoped API (find the `sys_ws_definition` for `va_google_chat_inbound`), run a GlideRecord update from that scope, then delete the temp operation.

---

## Debugging

### Do
- Use VA Designer **Test tool** for topic/NLU testing (channel-agnostic, instant)
- Check `open_nlu_predict_log` for NLU prediction results (model, confidence, utterance)
- Check `sys_cs_conversation` filtered by `device_type=Google Chat` for conversation state
- Use `va_gchat_debug_log` system property for custom debug logging

### Do NOT
- **NEVER** query `syslog` / `sys_log` - it freezes the session especially for large and busy instances
- Don't guess table names - verify with `sys_db_object` first
- Don't chase language issues (disproven: NLU runs with `language=en`)
- Don't try to write `published_definition` directly - it's computed on publish

### Common Issues
| Symptom | Cause | Fix |
|---------|-------|-----|
| "I didn't understand" despite NLU match | `published_definition.roles` doesn't include `public` | Republish topic from VA Designer |
| No NLU prediction logged | Context profile forces wrong `model_type` | Check `sys_cs_context_profile` order/conditions |
| REST writes silently fail | Scoped table protection | Use GlideRecord via scoped scripted REST op |
| `installGoogleChatCustomBot` errors | Leftover records from previous install | Delete all related oauth/oidc/jwt/provider records first |
| "not responding" in Google Chat | Add-on checkbox unchecked in GCP | Check "Build as Workspace add-on" in GCP Chat API config |

---

## Setup for a New Instance

Follow these steps to connect a different ServiceNow instance to Google Chat.

### Step 1: GCP Setup
1. Create a GCP project under a **Google Workspace** account (not personal Gmail)
2. Enable **Google Chat API** and **Google Workspace Add-ons API**
3. In the Chat API config, set:
   - HTTP endpoint URL to `https://<YOUR_INSTANCE>.service-now.com/api/sn_va_google_chat/va_google_chat_inbound/events`
   - Check **"Build this Chat app as a Workspace add-on"**
   - Note the **service account email** shown (format: `service-<NUMBER>@gcp-sa-gsuiteaddons.iam.gserviceaccount.com`)
4. Create a service account for outbound (SN to Google) calls
5. Generate a P12 key for that outbound service account

### Step 2: ServiceNow Setup
1. Confirm the `sn_va_google_chat` plugin is active on the instance
2. Upload the P12 key file as an attachment to any record (e.g. `sys_certificate`). Note the `sys_attachment` sys_id from the URL
3. Call the install API (browser or REST client):
```
https://<YOUR_INSTANCE>.service-now.com/api/sn_va_google_chat/multi_instance_google_chat_bot_ui/installGoogleChatCustomBot/<BOT_NAME>/<INBOUND_SA_EMAIL>/<OUTBOUND_SA_EMAIL>/<P12_PASSWORD>/<P12_ATTACHMENT_SYS_ID>/false
```
4. Verify the install created records: check `sys_cs_provider_application`, `oauth_oidc_entity`, `jwt_keystore_aliases` for new entries matching your bot name

### Step 3: Topic Setup
1. Open VA Designer, find or create a topic
2. Set roles to **public** (or appropriate role)
3. **Publish** the topic (this regenerates `published_definition`)
4. Test in VA Designer Test tool first, then Google Chat

### Step 4: Verify
1. Open Google Chat, find the bot by name
2. Type "hi" - you should get the VA greeting
3. Type a phrase matching your topic - it should trigger

### Switching an Existing Bot to a New Instance
If the bot was previously pointing to a different SN instance:
1. Update the HTTP endpoint URL in GCP Chat API config to the new instance
2. On the new SN instance, run `installGoogleChatCustomBot` with the same GCP service accounts
3. If the install fails with conflicts, delete any existing `sys_cs_provider_application`, `oauth_oidc_entity`, `jwt_keystore_aliases`, `oidc_provider_configuration`, and `provider_auth` records that reference the old bot name or service accounts

### .env File
Store instance-specific values locally (do not commit):
```
SN_INSTANCE=https://<YOUR_INSTANCE>.service-now.com
SN_USERNAME=<admin_user>
GCP_PROJECT_ID=<your-project>
GCP_PROJECT_NUMBER=<123456789>
GCP_OUTBOUND_SA=<sa-name>@<project>.iam.gserviceaccount.com
GCP_P12_FILE=<your-key>.p12
GCP_P12_PASSWORD=<password>
GCP_WORKSPACE_EMAIL=<admin@yourdomain.com>
```

---

## Result

![Google Chat VA Integration Working](screenshot-discovery-performance-metrics.png)

Google Chat successfully triggers the **Discovery Performance Metrics** VA topic. The bot presents the topic flow with interactive dropdowns and submit buttons rendered natively in Google Chat.
