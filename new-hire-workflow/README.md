# New Hire Provisioning Automation

An n8n workflow that automatically provisions accounts across GitHub, Slack, Jira, and email the moment a new employee is added to your HR system. No manual steps, no forgotten accounts, no new starters sitting around on day one waiting for access.

---

## What It Does

When your HR system creates a new employee record, this workflow fires instantly. It reads the employee's name, email, role, and department, then makes intelligent decisions about which tools they need access to. Developers get GitHub, Jira, and Slack. Non-technical staff get Slack and email. Everyone gets a welcome email. The hiring manager gets a confirmation. Your admin team gets notified in Slack. The whole thing runs in seconds without anyone touching it.

---

## How It Works

The workflow runs eleven nodes. After the initial data extraction, several branches run in parallel so provisioning across all tools happens at the same time rather than one after another.

**Node 1: HR System Webhook Trigger**
Your HR system sends a POST request to n8n whenever a new employee record is created. This is the entry point for the entire workflow. If your HR system does not support outgoing webhooks, the alternative approach is to use a Schedule Trigger that polls a Google Sheet or Airtable table where HR staff add new starters manually.

**Node 2: Extract and Validate Hire Data**
This is the brain of the workflow. It reads the incoming payload, normalises all field values, and makes several decisions before anything else runs.

First, it validates the record. If a first name, last name, or email is missing, the workflow throws an error and stops immediately rather than partially provisioning accounts in an inconsistent state.

Second, it determines which tools the employee needs based on their department and role. The logic works by checking whether the department or role contains certain keywords. For example, anyone in engineering, development, or devops is treated as a developer and gets GitHub and Jira. You can edit these keyword lists to match your company's terminology.

Third, it builds the list of Slack channels to add the new hire to. Everyone gets a base set of channels like general and announcements. Developers also get added to engineering-specific channels. The channel IDs are set as constants in this node's code and can be updated there.

**Node 3: Check - Needs GitHub and Username Provided**
A filter that only proceeds with the GitHub branch if two things are true: the employee's role qualifies them for GitHub access, and a GitHub username was actually included in the HR record. Both conditions must pass. If either fails, the GitHub branch is skipped silently.

**Node 4: GitHub - Invite to Organisation**
Sends an invitation to the new hire's existing GitHub account to join your organisation. The invitation arrives by email and the user must accept it before gaining access to your repositories.

**Node 5: Slack - Invite User**
Sends a Slack workspace invitation to the new hire's work email address. The channel list built in Node 2 is passed through here, so the user is automatically added to the right channels as soon as they accept.

**Node 6: Check - Needs Jira**
A filter that checks whether the employee's role qualifies them for Jira access. Developers, designers, and project managers pass through. Other departments are skipped.

**Node 7: Jira - Create User Account**
Creates a new Atlassian account for the employee and adds them to your Jira instance. Jira sends the user an activation email. The account will show as pending until they accept.

**Node 8: Jira - Add User to Project (Optional)**
An optional follow-up node that adds the newly created Jira user to a specific project with a designated role. This uses a direct Jira REST API call since n8n's built-in Jira node does not expose project role membership. The node can be disconnected without affecting anything else.

**Node 9: Send Welcome Email to New Hire**
Sends a personalised HTML email to the new hire's work address. The email includes their start date, a summary of which tool invitations to expect and where they will come from, and their manager's contact details for any pre-start questions.

**Node 10: Send Manager Confirmation Email**
Sends a summary email to the hiring manager listing every account that was provisioned and its status. This gives managers a record of what happened without needing to log into n8n.

**Node 11: Notify Admin Slack Channel**
Posts a real-time provisioning summary to an internal admin or IT Slack channel. This gives your operations team a live log of all onboarding activity. This node is optional and can be removed without affecting the rest of the workflow.

---

## Provisioning Logic by Role

The workflow provisions tools selectively based on department and role keywords. The defaults are listed below. All of this logic lives in Node 2 and can be edited.

| Tool | Who Gets It |
|---|---|
| GitHub | Developers, engineers, designers |
| Slack | Everyone |
| Jira | Developers, engineers, designers, product and project managers |
| Welcome email | Everyone |
| Manager confirmation email | Always sent to the manager on file |

---

## Setup

### Prerequisites

You will need the following before setting up this workflow:

An n8n instance, either cloud or self-hosted. An HR system that can send outgoing webhooks, or a Google Sheet or Airtable table as an alternative trigger. A GitHub organisation with owner-level access. A Slack workspace. A Jira project on Atlassian. An SMTP-capable email account for sending the welcome and confirmation emails.

### Step 1: Import the Workflow

Open n8n and create a new workflow. Click the menu in the top right and select Import from file. Upload the JSON file and the full workflow will load.

### Step 2: Add Your Credentials

You need to connect four services inside n8n.

**GitHub**
Go to Credentials, create a new GitHub API credential, and add a Personal Access Token from github.com/settings/tokens. The token must belong to an organisation owner and needs the admin:org scope to send invitations.

**Slack**
Go to Credentials, create a new Slack API credential. Create a Slack app at api.slack.com/apps and add the following OAuth scopes: users:write, channels:manage, groups:write. Copy the Bot OAuth Token into n8n.

**Jira**
Go to Credentials, create a new Jira Software Cloud API credential. Enter your Atlassian subdomain, your admin account email, and an API token generated at id.atlassian.com/manage-profile/security/api-tokens.

**SMTP**
Go to Credentials, create a new SMTP credential. Common settings are listed below.

| Email Provider | Host | Port |
|---|---|---|
| Google Workspace | smtp.gmail.com | 587 |
| Microsoft 365 | smtp.office365.com | 587 |
| SendGrid | smtp.sendgrid.net | 587 |

For Google Workspace, use an App Password rather than your account password. Generate one at myaccount.google.com/apppasswords.

### Step 3: Update the Placeholders

Open each node in n8n and replace the following values.

**In the Extract and Validate Hire Data node:**
Replace the Slack channel ID placeholders (CHANNEL_ID_GENERAL, CHANNEL_ID_ANNOUNCEMENTS, etc.) with your real channel IDs. Find these in Slack by right-clicking a channel, selecting View channel details, and copying the ID at the bottom. It will look like C04ABC12345.

**In the GitHub node:**
Replace YOUR_GITHUB_ORG_NAME with your organisation's GitHub handle. If your org is at github.com/acme-corp, enter acme-corp.

**In the Jira - Add User to Project node:**
Replace YOUR_JIRA_SUBDOMAIN, YOUR_JIRA_PROJECT_KEY, and YOUR_ROLE_ID. To find your project's role IDs, visit https://YOUR_SUBDOMAIN.atlassian.net/rest/api/3/project/YOUR_PROJECT_KEY/role in a browser while logged in as an admin. This returns a list of roles and their numeric IDs.

**In both email nodes:**
Replace onboarding@YOUR_COMPANY_DOMAIN.com with a real sending address from your SMTP account.

**In the Notify Admin Slack Channel node:**
Replace YOUR_ADMIN_SLACK_CHANNEL_ID with the ID of your internal admin or IT channel.

### Step 4: Match Your HR System's Field Names

Open the Extract and Validate Hire Data node and look at the field extraction block at the top. The code tries a few common variations for each field (for example, both first_name and firstName), but your HR system may use different names entirely. Update the field keys to match whatever your system sends.

The expected payload fields and their defaults are:

| Field | Expected Key | Notes |
|---|---|---|
| First name | first_name or firstName | Required |
| Last name | last_name or lastName | Required |
| Work email | email | Required |
| Role or job title | role or job_title | Used for provisioning logic |
| Department | department | Used for provisioning logic |
| Start date | start_date or startDate | Shown in welcome email |
| GitHub username | github_username or githubUsername | Required for GitHub branch |
| Manager email | manager_email or managerEmail | Used for confirmation email |

### Step 5: Set Up the HR Webhook

Activate the workflow in n8n and copy the webhook URL from the HR System Webhook Trigger node. In your HR system, navigate to the webhook or integration settings and create a new webhook pointing to that URL. Set it to fire when a new employee record is created. The exact steps vary by HR system. Common locations are listed in the notes on Node 1 inside n8n.

### Step 6: Test It

Trigger the webhook manually using a tool like Postman or send a test payload from your HR system. Use a real email address you have access to so you can confirm the welcome email arrives. Check your GitHub organisation's pending invitations, your Jira admin panel, and your Slack workspace to verify all accounts were created correctly.

---

## Customising the Welcome Email

The welcome email body is plain HTML and lives inside Node 9. You can edit it directly in n8n to match your company tone, add links to an onboarding wiki, include a first-day checklist, or any other content relevant to new starters.

---

## Disabling Individual Tools

Each provisioning branch can be disabled independently.

To disable GitHub provisioning: delete the Check node and the GitHub invite node, or simply disconnect them.
To disable Jira provisioning: delete the Check node and both Jira nodes, or disconnect them.
To disable the Slack admin notification: delete or disconnect Node 11.
To disable the optional Jira project assignment: delete or disconnect Node 8.

None of these changes will affect the other branches.

---

## License

MIT
