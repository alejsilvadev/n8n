# GitHub Issues to Jira Auto Sync

An n8n workflow that automatically creates a Jira ticket every time a new issue is opened on GitHub. It maps labels to ticket types and priorities, posts a comment back on the GitHub issue with a link to the Jira ticket, and optionally notifies a Slack channel.

---

## What It Does

When a developer or user opens a GitHub issue, this workflow fires instantly via a webhook and handles everything from there. A Jira ticket is created with the correct type and priority based on the labels applied to the issue. A comment is then posted back on the GitHub issue confirming the sync and linking directly to the new Jira ticket. No manual copying, no tickets falling through the cracks.

---

## How It Works

The workflow is made up of seven nodes that run in sequence.

**Node 1: GitHub Webhook Trigger**
GitHub sends a POST request to n8n every time an issue event occurs in your repository. This is the entry point for the entire workflow.

**Node 2: Filter for Opened Issues**
Not every issue event should trigger a Jira ticket. GitHub fires webhooks for edits, label changes, assignments, and closures as well. This filter checks that the action is specifically "opened" and stops the workflow for anything else.

**Node 3: Extract and Map Issue Data**
This is the core logic node. It reads the GitHub issue payload and extracts the title, body, reporter, labels, and repository. It then uses the labels to decide two things: what type of Jira ticket to create and what priority to assign it. It also checks for labels that should be skipped entirely.

Label to ticket type mapping:

| GitHub Label | Jira Ticket Type |
|---|---|
| bug, defect | Bug |
| feature, enhancement, story | Story |
| epic | Epic |
| anything else | Task |

Label to priority mapping:

| GitHub Label | Jira Priority |
|---|---|
| critical, urgent, p0 | Highest |
| high, p1 | High |
| low, p3 | Low |
| anything else | Medium |

Labels that will skip Jira ticket creation entirely: question, duplicate, wontfix, invalid, docs.

All of these mappings can be edited in the code of this node to match your team's labeling conventions.

**Node 4: Filter for Skip Labels**
A second filter that checks if the issue was flagged to be skipped. If it was, the workflow stops here without creating anything in Jira.

**Node 5: Create Jira Ticket**
Creates the ticket in Jira using the data prepared in Node 3. The ticket will include the original issue title, a formatted description containing the reporter's username and a link back to the GitHub issue, the issue type, and the priority. All tickets created by this workflow are automatically tagged with the label "github-sync" in Jira so they can be easily identified and filtered.

**Node 6: Post GitHub Comment with Jira Link**
Once the Jira ticket is created, this node posts a comment on the original GitHub issue. The comment includes the Jira ticket ID and a direct link to it. This keeps GitHub users informed and creates a permanent cross-reference between the two systems.

**Node 7: Notify Slack (Optional)**
Sends a message to a Slack channel of your choice summarising the new ticket. This node is entirely optional and can be removed without affecting the rest of the workflow.

---

## Setup

### Prerequisites

You will need accounts for the following services:

- n8n (cloud or self-hosted)
- GitHub (with access to the repository you want to monitor)
- Jira (Atlassian account, free tier supports up to 10 users)
- Slack (optional, only needed for the notification node)

### Step 1: Import the Workflow

Open n8n and create a new workflow. Click the menu in the top right corner and select "Import from file". Upload the JSON file from this repository and the full workflow will appear.

### Step 2: Add Your Credentials

You need to connect three services inside n8n before the workflow can run.

**GitHub**
Go to Credentials, create a new GitHub API credential, and paste in a Personal Access Token. You can generate one at github.com/settings/tokens. The token needs the "repo" scope.

**Jira**
Go to Credentials, create a new Jira Software Cloud API credential, and fill in your Atlassian subdomain, your account email, and an API token. You can generate an API token at id.atlassian.com/manage-profile/security/api-tokens.

**Slack (optional)**
Go to Credentials, create a new Slack API credential, and paste in a Bot OAuth Token. You can create a Slack app and get a token at api.slack.com/apps. The app needs the "chat:write" scope.

### Step 3: Update the Placeholders

Open each node and replace the following values:

**In the Create Jira Ticket node:**
Replace YOUR_JIRA_PROJECT_KEY with your actual Jira project key. You can find this by looking at any existing ticket in your project. If tickets are named DEV-123, your key is DEV.

**In the Post GitHub Comment node and Notify Slack node:**
Replace YOUR_JIRA_SUBDOMAIN with the subdomain of your Atlassian account. If your Jira is at acme.atlassian.net, the subdomain is acme.

**In the Notify Slack node (optional):**
Replace YOUR_SLACK_CHANNEL_ID with the ID of the channel you want to post to. You can find this by right-clicking a channel in Slack, selecting "View channel details", and copying the ID at the bottom. It will look something like C04ABC12345.

### Step 4: Set Up the GitHub Webhook

Activate the workflow in n8n and copy the webhook URL from the GitHub Webhook Trigger node. Then go to your GitHub repository, navigate to Settings, then Webhooks, and click "Add webhook". Paste the URL into the Payload URL field, set the content type to "application/json", and under "Which events would you like to trigger this webhook?" select "Issues" only. Save the webhook.

### Step 5: Test It

Open a new issue on your GitHub repository with a label like "bug". Within a few seconds you should see a new Jira ticket created, and a comment appear on the GitHub issue with a link to it.

---

## Customising Label Mappings

All label logic lives in the "Extract and Map Issue Data" node. Open that node in n8n and edit the JavaScript arrays to match whatever labels your team uses. The comments in the code explain exactly which section controls ticket type and which controls priority.

---

## Disabling the Slack Notification

If you do not use Slack, simply delete the "Notify Slack" node or disconnect it from the workflow. Everything else will continue to work normally.

---

## License

MIT
