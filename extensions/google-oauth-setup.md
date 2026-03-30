---
title: "Google OAuth Setup"
description: "One-time setup for any Google extension in IronClaw"
---

All Google extensions share the same OAuth 2.0 setup. Complete these steps once — you can reuse the same Google Cloud project and credentials for every Google extension you install.

---

<Steps>

<Step title="Create a Google Cloud Project">

Go to [Google Cloud Console](https://console.cloud.google.com) and create a new project (or select an existing one).

1. Click **Select a project** → **New Project**
2. Give it a name (e.g. `ironclaw`) and click **Create**

</Step>

<Step title="Create OAuth 2.0 Credentials">

Go to **Google Auth Platform → Clients** and create a new client:

1. Click **Create client**
2. Set **Application type** to **Web application**
3. Give it a name (e.g. `ironclaw`)
4. Under **Authorized redirect URIs**, click **+ Add URI** and enter:

   ```
   http://127.0.0.1:9876/callback
   ```

5. Click **Create** and copy the **Client ID** and **Client Secret** shown

</Step>

<Step title="Add Test Users">

Since the app is in **Testing** mode, only explicitly added users can authorize it. Go to **APIs & Services → OAuth consent screen**, scroll down to **Test users**, and click **+ Add users**.

Add the Google account(s) that will use the extension. The app supports up to 100 test users before requiring verification.

<Info>
Only test users can complete the OAuth flow while the app is in Testing mode. If you get an "access blocked" error, make sure your account is listed here.
</Info>

</Step>

<Step title="Open the SSH Tunnel">

The Google OAuth callback runs on the remote server at port `9876`. Open an SSH tunnel so the callback reaches the server:

```bash
ssh -p 15222 -L 9876:127.0.0.1:9876 solid-wolf@agent4.near.ai
```

Keep this terminal session open while completing the OAuth flow.

<Info>
The `-L 9876:127.0.0.1:9876` flag forwards your local port to the server. Without it, the OAuth callback will fail because port 9876 is only accessible from within the server.
</Info>

</Step>

<Step title="Set Environment Variables">

Export your OAuth credentials on the server:

```bash
export GOOGLE_OAUTH_CLIENT_ID=<your-client-id>
export GOOGLE_OAUTH_CLIENT_SECRET=<your-client-secret>
```

</Step>

</Steps>

You're ready to install any Google extension. Return to the extension page to complete the remaining steps.
