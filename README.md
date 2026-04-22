# n8n Workflow Setup Guide

## AI Product Idea Generator — Reddit → Product Hunt

This workflow extracts pain points from Reddit, searches for existing solutions on Product Hunt, and saves results to Notion.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and a lightweight runtime like [Colima](https://github.com/abetterinternet/colima) (macOS)
- A [TinyFish API key](https://agent.tinyfish.ai/signup) (free, 500 steps included)
- A [Notion API key](https://www.notion.so/my-integrations)

## Setup

### 1. Start n8n with Docker

```bash
# macOS without Docker Desktop — install Colima first
brew install colima
colima start --cpu 2 --memory 4

# Start n8n
mkdir -p ~/.n8n
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Open http://localhost:5678 and create your owner account.

### 2. Install the TinyFish Community Node

The n8n Docker image doesn't include build tools, so we build the node in a separate container using the exact same Node.js version.

```bash
# Check the Node version inside the n8n image
docker run --rm --entrypoint /bin/sh n8nio/n8n -c "node --version"
# Example output: v24.13.1

# Build the community node using that exact version
docker run --rm -v ~/.n8n/nodes:/out node:24.13.1-alpine sh -c "
  apk add python3 make g++ &&
  cd /out &&
  npm init -y 2>/dev/null &&
  npm install n8n-nodes-tinyfish --ignore-scripts &&
  rm -rf node_modules/isolated-vm &&
  echo 'Done'
"

# Restart n8n to pick up the new node
docker restart n8n
```

> **Why `--ignore-scripts` and removing `isolated-vm`?**
> The `isolated-vm` native module is used by n8n's Code node sandbox, not by TinyFish itself. It causes segfaults in the hardened Alpine Docker image. Removing it has zero impact on the TinyFish node — all other nodes continue to work normally.

---

### Required Credentials

You need to configure the following credentials in n8n before running this workflow:

---

## 1. TinyFish Web Agent API

**Used in nodes:**
- TinyFish: Reddit Pain Points
- TinyFish: Search Product Hunt

**Setup Instructions:**

1. In n8n, go to **Credentials** → **New**
2. Search for and select **TinyFish API**
3. Enter your TinyFish API credentials:
   - **API Key**: Your TinyFish API key
4. Click **Save**
5. Name it: `TinyFish Web Agent account`

**Where to get credentials:**
- Sign up at [TinyFish](https://tinyfish.io) or your TinyFish provider
- Generate an API key from your account dashboard

---

## 2. Notion API

**Used in nodes:**
- Append a block
- Notion Create a page

**Setup Instructions:**

1. In n8n, go to **Credentials** → **New**
2. Search for and select **Notion API**
3. Click **Connect my account** or enter credentials manually:
   - **Authentication**: OAuth2 (recommended) or Internal Integration Token
   - Follow the OAuth flow or paste your integration token
4. Click **Save**
5. Name it: `Notion account`

**Where to get credentials:**

### Option A: OAuth2 (Recommended)
- Follow n8n's OAuth connection flow
- Grant access to your Notion workspace

### Option B: Internal Integration Token
1. Go to [Notion Integrations](https://www.notion.so/my-integrations)
2. Click **+ New integration**
3. Give it a name (e.g., "n8n Integration")
4. Select the workspace
5. Copy the **Internal Integration Token**
6. In your Notion page, click **Share** → **Invite** → Select your integration

**Important:** Make sure the integration has access to:
- The parent database/page specified in the workflow
- Permission to create pages and append blocks

---

## Importing the Workflow

1. Open n8n
2. Click **Workflows** → **Import from File**
3. Select `AI Product Idea Generator — Reddit.json`
4. After import, you'll see warnings about missing credentials
5. Click on each node with a credential warning
6. Select the appropriate credential from the dropdown
7. Save the workflow

---

## Workflow Configuration

### Update Notion Page URL

In the **Notion Create a page** node, update the `pageId` parameter:

```json
"pageId": {
  "__rl": true,
  "value": "YOUR_NOTION_PAGE_URL",
  "mode": "url"
}
```

Replace `YOUR_NOTION_PAGE_URL` with your actual Notion database or page URL.

---

## Testing the Workflow

1. Click **Execute Workflow** or the **Click to Run** trigger
2. Monitor each node's execution
3. Check your Notion page for the results

---

## Troubleshooting

### TinyFish API Issues
- Verify your API key is valid and has sufficient credits
- Check rate limits on your TinyFish account

### Notion API Issues
- Ensure the integration has access to the target page
- Verify the page URL is correct
- Check that the integration has write permissions

### General Issues
- Check n8n logs for detailed error messages
- Verify all nodes are properly connected
- Ensure your n8n instance can access external APIs
