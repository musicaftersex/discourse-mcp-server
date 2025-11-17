# Discourse MCP Server - n8n Integration Guide

## Overview

This guide explains how to integrate the Discourse MCP server into n8n AI agent workflows using HTTP transport. The MCP server exposes Discourse forum capabilities as tools that can be called via HTTP requests from n8n.

## Table of Contents

1. [Quick Start](#quick-start)
2. [Authentication Setup](#authentication-setup)
3. [Server Configuration](#server-configuration)
4. [n8n Workflow Integration](#n8n-workflow-integration)
5. [Available Tools](#available-tools)
6. [Example Requests](#example-requests)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting](#troubleshooting)

---

## Quick Start

### 1. Start the Server with HTTP Transport

For **read-only access**:
```bash
npx @discourse/mcp@latest \
  --transport http \
  --port 3000 \
  --site https://your-discourse-site.com
```

For **write access with User API Key**:
```bash
npx @discourse/mcp@latest \
  --transport http \
  --port 3000 \
  --profile profile.json \
  --allow_writes \
  --read_only=false
```

### 2. Verify the Server is Running

**Health check:**
```bash
curl http://localhost:3000/health
```

**MCP endpoint:**
```
POST http://localhost:3000/mcp
```

---

## Authentication Setup

### User API Keys (Granular Access - Recommended)

User API Keys provide granular, scope-based access without requiring admin permissions. They're ideal for n8n workflows where you want fine-grained control.

#### Step 1: Generate a User API Key

```bash
npx @discourse/mcp@latest generate-user-api-key \
  --site https://your-discourse-site.com \
  --save-to profile.json \
  --scopes "read,write,notifications"
```

This will:
1. Generate an RSA key pair
2. Display an authorization URL
3. Prompt you to visit the URL and approve the request
4. Ask you to paste the encrypted payload
5. Decrypt and save the key to `profile.json`

#### Step 2: Review the Generated Profile

```json
{
  "auth_pairs": [
    {
      "site": "https://your-discourse-site.com",
      "user_api_key": "your_decrypted_key_here",
      "user_api_client_id": "your_client_id_here"
    }
  ],
  "site": "https://your-discourse-site.com",
  "transport": "http",
  "port": 3000,
  "read_only": false,
  "allow_writes": true,
  "log_level": "info"
}
```

#### Available Scopes

User API Keys support the following scopes:
- `read` - Read public and private content
- `write` - Create and edit posts, topics
- `message_bus` - Subscribe to notifications
- `push` - Push notifications
- `notifications` - Read notifications
- `session_info` - Access session information
- `one_time_password` - Generate OTPs

For n8n workflows, typically you'll need: `read,write,notifications`

### Admin API Keys (Full Access)

If you need admin-level operations (creating users, categories), use Admin API Keys:

```json
{
  "auth_pairs": [
    {
      "site": "https://your-discourse-site.com",
      "api_key": "your_admin_api_key",
      "api_username": "system"
    }
  ]
}
```

Generate Admin API Keys via:
- Discourse Admin Panel → API → New API Key

---

## Server Configuration

### Profile-Based Configuration (Recommended)

Create a `profile.json` file:

```json
{
  "site": "https://your-discourse-site.com",
  "transport": "http",
  "port": 3000,
  "auth_pairs": [
    {
      "site": "https://your-discourse-site.com",
      "user_api_key": "YOUR_USER_API_KEY",
      "user_api_client_id": "YOUR_CLIENT_ID"
    }
  ],
  "read_only": false,
  "allow_writes": true,
  "timeout_ms": 15000,
  "concurrency": 4,
  "log_level": "info",
  "max_read_length": 50000,
  "default_search": "order:latest-post",
  "tools_mode": "auto"
}
```

Run with profile:
```bash
npx @discourse/mcp@latest --profile /absolute/path/to/profile.json
```

### Configuration Options

| Flag | Default | Description |
|------|---------|-------------|
| `--transport` | `stdio` | Transport type: `stdio` or `http` |
| `--port` | `3000` | Port for HTTP transport |
| `--site` | - | Tether to a single Discourse site |
| `--auth_pairs` | `[]` | JSON array of site credentials |
| `--read_only` | `true` | Disable write operations |
| `--allow_writes` | `false` | Enable write tools (requires auth) |
| `--timeout_ms` | `15000` | Request timeout in milliseconds |
| `--concurrency` | `4` | Max concurrent requests |
| `--log_level` | `info` | Logging: `silent\|error\|info\|debug` |
| `--max_read_length` | `50000` | Max characters returned per post |
| `--default_search` | - | Default search query prefix |
| `--tools_mode` | `auto` | Tool discovery: `auto\|discourse_api_only\|tool_exec_api` |

### Command-Line Examples

**Read-only with debug logging:**
```bash
npx @discourse/mcp@latest \
  --transport http \
  --port 3000 \
  --site https://meta.discourse.org \
  --log_level debug
```

**Write-enabled with User API Key:**
```bash
npx @discourse/mcp@latest \
  --transport http \
  --port 3000 \
  --site https://your-site.com \
  --allow_writes \
  --read_only=false \
  --auth_pairs '[{"site":"https://your-site.com","user_api_key":"KEY"}]'
```

**With custom search defaults:**
```bash
npx @discourse/mcp@latest \
  --transport http \
  --port 3000 \
  --site https://your-site.com \
  --default_search "category:support tag:urgent order:latest-post"
```

---

## n8n Workflow Integration

### HTTP Request Node Configuration

Add an **HTTP Request** node in your n8n workflow with:

- **Method:** `POST`
- **URL:** `http://localhost:3000/mcp`
- **Authentication:** None (if running locally)
- **Headers:**
  - `Content-Type: application/json`

### MCP Protocol Format

All requests follow the JSON-RPC 2.0 format:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "tool_name",
    "arguments": {
      "arg1": "value1",
      "arg2": "value2"
    }
  }
}
```

### Response Format

Successful response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Human-readable result with optional JSON footer"
      }
    ]
  }
}
```

Error response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Error description"
  }
}
```

---

## Available Tools

### Read-Only Tools (Always Available)

#### `discourse_search`
Search for topics across the forum.

**Input:**
```json
{
  "query": "AI automation",
  "with_private": false,
  "max_results": 10
}
```

**Output:** Text summary + JSON footer with results array

---

#### `discourse_read_topic`
Read a topic with its posts.

**Input:**
```json
{
  "topic_id": 12345,
  "post_limit": 5,
  "start_post_number": 1
}
```

**Output:** Title, category, tags, and post content

---

#### `discourse_read_post`
Read a specific post by ID.

**Input:**
```json
{
  "post_id": 67890
}
```

**Output:** Author, timestamp, and post content

---

#### `discourse_list_categories`
List all forum categories.

**Input:**
```json
{}
```

**Output:** Category names with topic counts

---

#### `discourse_list_tags`
List all forum tags.

**Input:**
```json
{}
```

**Output:** Tags with usage counts

---

#### `discourse_get_user`
Get user profile information.

**Input:**
```json
{
  "username": "johndoe"
}
```

**Output:** Display name, trust level, bio, joined date

---

#### `discourse_filter_topics`
Advanced topic filtering with query language.

**Input:**
```json
{
  "filter": "category:support tag:bug status:open order:latest-post",
  "page": 1,
  "per_page": 20
}
```

**Query Language:**
- **Categories:** `category:support` or `categories:support,feature`
- **Tags:** `tag:bug` or `tags:bug,urgent` (AND: `tag:bug+urgent`)
- **Status:** `status:open|closed|archived|public`
- **Dates:** `created-after:2024-01-01` or `created-after:7` (days)
- **Numeric:** `likes-min:10`, `posts-max:50`, `views-min:100`
- **Ordering:** `order:activity|created|latest-post|likes|views`
- **Personal:** `in:bookmarked|watching|tracking`
- **Exclude:** `-category:meta`, `-tag:resolved`

**Output:** Paginated topic list + JSON footer

---

### Write Tools (Requires `--allow_writes` + Auth)

#### `discourse_create_post`
Create a reply in an existing topic.

**Input:**
```json
{
  "topic_id": 12345,
  "raw": "This is my reply to the topic.\n\nIt can be multi-line."
}
```

**Rate limit:** ~1 request/second

---

#### `discourse_create_topic`
Create a new topic.

**Input:**
```json
{
  "title": "New Discussion About AI",
  "raw": "This is the content of the first post...",
  "category_id": 5,
  "tags": ["ai", "automation", "discussion"]
}
```

**Rate limit:** ~1 request/second

---

#### `discourse_create_category`
Create a new category (requires admin API key).

**Input:**
```json
{
  "name": "AI Research",
  "color": "0088CC",
  "text_color": "FFFFFF",
  "parent_category_id": 1,
  "description": "Discussions about AI research and development"
}
```

**Rate limit:** ~1 request/second

---

#### `discourse_create_user`
Create a new user (requires admin API key).

**Input:**
```json
{
  "username": "newuser",
  "email": "newuser@example.com",
  "name": "New User",
  "password": "securepassword123",
  "active": true,
  "approved": true
}
```

**Rate limit:** ~1 request/second

---

## Example Requests

### Example 1: Search Topics

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "discourse_search",
    "arguments": {
      "query": "n8n automation workflows",
      "max_results": 5
    }
  }
}
```

### Example 2: Read a Topic

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "discourse_read_topic",
    "arguments": {
      "topic_id": 12345,
      "post_limit": 10
    }
  }
}
```

### Example 3: Filter Topics with Advanced Query

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "discourse_filter_topics",
    "arguments": {
      "filter": "category:support tag:bug+urgent status:open created-after:7 order:latest-post",
      "per_page": 20
    }
  }
}
```

### Example 4: Create a New Topic

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/call",
  "params": {
    "name": "discourse_create_topic",
    "arguments": {
      "title": "Weekly Automation Report",
      "raw": "Here's this week's automation report:\n\n- Tasks completed: 150\n- Errors: 2\n- Success rate: 98.7%",
      "category_id": 5,
      "tags": ["automation", "reports", "weekly"]
    }
  }
}
```

### Example 5: Reply to a Topic

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tools/call",
  "params": {
    "name": "discourse_create_post",
    "arguments": {
      "topic_id": 12345,
      "raw": "Thanks for sharing this! I've automated this workflow in n8n and it's working great."
    }
  }
}
```

### Example 6: Get User Information

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "method": "tools/call",
  "params": {
    "name": "discourse_get_user",
    "arguments": {
      "username": "system"
    }
  }
}
```

---

## Production Deployment

### Option 1: Using PM2

Install PM2:
```bash
npm install -g pm2
```

Create `ecosystem.config.js`:
```javascript
module.exports = {
  apps: [{
    name: 'discourse-mcp',
    script: 'npx',
    args: '@discourse/mcp@latest --profile /absolute/path/to/profile.json',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '500M',
    env: {
      NODE_ENV: 'production'
    }
  }]
};
```

Start the service:
```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

Monitor:
```bash
pm2 status
pm2 logs discourse-mcp
pm2 monit
```

### Option 2: Using systemd

Create `/etc/systemd/system/discourse-mcp.service`:
```ini
[Unit]
Description=Discourse MCP Server
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/discourse-mcp
ExecStart=/usr/bin/npx @discourse/mcp@latest --profile /opt/discourse-mcp/profile.json
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable discourse-mcp
sudo systemctl start discourse-mcp
sudo systemctl status discourse-mcp
```

View logs:
```bash
sudo journalctl -u discourse-mcp -f
```

### Option 3: Using Docker

Create `Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install the package globally
RUN npm install -g @discourse/mcp@latest

# Copy configuration
COPY profile.json /app/profile.json

EXPOSE 3000

CMD ["discourse-mcp", "--profile", "/app/profile.json"]
```

Build and run:
```bash
docker build -t discourse-mcp .
docker run -d \
  --name discourse-mcp \
  -p 3000:3000 \
  --restart unless-stopped \
  discourse-mcp
```

With docker-compose (`docker-compose.yml`):
```yaml
version: '3.8'

services:
  discourse-mcp:
    image: node:18-alpine
    container_name: discourse-mcp
    working_dir: /app
    volumes:
      - ./profile.json:/app/profile.json:ro
    ports:
      - "3000:3000"
    command: >
      sh -c "npm install -g @discourse/mcp@latest &&
             discourse-mcp --profile /app/profile.json"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

Start:
```bash
docker-compose up -d
docker-compose logs -f
```

### Reverse Proxy Configuration

#### nginx

```nginx
upstream discourse_mcp {
    server localhost:3000;
}

server {
    listen 80;
    server_name discourse-mcp.yourdomain.com;

    location / {
        proxy_pass http://discourse_mcp;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    location /health {
        proxy_pass http://discourse_mcp/health;
        access_log off;
    }
}
```

#### Caddy

```
discourse-mcp.yourdomain.com {
    reverse_proxy localhost:3000
}
```

### Security Best Practices

1. **Keep secrets in profile.json, not command-line arguments**
2. **Restrict file permissions:**
   ```bash
   chmod 600 profile.json
   chown www-data:www-data profile.json
   ```
3. **Use HTTPS in production** (via reverse proxy)
4. **Limit network access** (firewall rules, security groups)
5. **Enable rate limiting** at the reverse proxy level
6. **Monitor logs** for suspicious activity
7. **Rotate API keys** regularly
8. **Use User API Keys** instead of Admin keys when possible

---

## Troubleshooting

### Server Won't Start

**Check Node version:**
```bash
node --version  # Should be >= 18
```

**Check port availability:**
```bash
lsof -i :3000
netstat -tuln | grep 3000
```

**Run with debug logging:**
```bash
npx @discourse/mcp@latest --profile profile.json --log_level debug
```

### Authentication Errors

**"Unauthorized" or "403 Forbidden":**
- Verify your User API Key or Admin API Key is correct
- Check that the key hasn't expired (User API Keys expire after 180 days of inactivity)
- Ensure `allow_writes` and `read_only=false` are set for write operations
- Verify the site URL matches exactly in `auth_pairs`

**Generate a fresh User API Key:**
```bash
npx @discourse/mcp@latest generate-user-api-key \
  --site https://your-site.com \
  --save-to profile.json
```

### Connection Timeouts

**Increase timeout:**
```bash
npx @discourse/mcp@latest --profile profile.json --timeout_ms 30000
```

**Check Discourse site is accessible:**
```bash
curl -I https://your-discourse-site.com/about.json
```

### Rate Limiting

**Error: "Rate limit exceeded"**
- Write operations are limited to ~1 request/second
- Add delays between write operations in your n8n workflow
- Check Discourse site rate limits (Admin Panel → Settings)

### Tool Not Available

**"discourse_create_post not found":**
- Ensure `--allow_writes` is set
- Ensure `--read_only=false` is set
- Verify authentication is configured in `auth_pairs`
- Check logs with `--log_level debug`

### n8n Connection Issues

**"ECONNREFUSED" error:**
- Verify the MCP server is running: `curl http://localhost:3000/health`
- Check the port number matches
- If using Docker, ensure port mapping is correct

**"Timeout" errors:**
- Increase timeout in n8n HTTP Request node settings
- Check MCP server logs for slow responses
- Consider increasing `--timeout_ms`

### Viewing Logs

**Using PM2:**
```bash
pm2 logs discourse-mcp
pm2 logs discourse-mcp --lines 100
```

**Using systemd:**
```bash
sudo journalctl -u discourse-mcp -f
sudo journalctl -u discourse-mcp --since "1 hour ago"
```

**Using Docker:**
```bash
docker logs -f discourse-mcp
docker logs discourse-mcp --tail 100
```

### Testing Tools

**Test search:**
```bash
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "discourse_search",
      "arguments": {
        "query": "test",
        "max_results": 1
      }
    }
  }'
```

**Test health endpoint:**
```bash
curl http://localhost:3000/health
```

### Performance Tuning

**Increase concurrency for high-traffic:**
```bash
npx @discourse/mcp@latest --profile profile.json --concurrency 10
```

**Adjust read length limits:**
```bash
npx @discourse/mcp@latest --profile profile.json --max_read_length 100000
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `Site validation failed` | Invalid Discourse URL | Check the site URL is correct and accessible |
| `Tool discourse_create_post not found` | Write tools not enabled | Add `--allow_writes --read_only=false` and auth |
| `Unauthorized` | Missing or invalid credentials | Check `auth_pairs` configuration |
| `Rate limit exceeded` | Too many write requests | Add delays between write operations |
| `fetch failed` | Network/DNS issue | Check connectivity to Discourse site |
| `Timeout` | Request took too long | Increase `--timeout_ms` |

---

## Additional Resources

- **Main README:** `README.md`
- **Agent Guide:** `AGENTS.md`
- **Source Code:** `src/index.ts`
- **User API Keys Spec:** https://meta.discourse.org/t/user-api-keys-specification/48536
- **MCP Protocol:** https://modelcontextprotocol.io
- **Discourse API Docs:** https://docs.discourse.org

---

## Support

For issues or questions:
- GitHub Issues: https://github.com/discourse/discourse-mcp-server/issues
- Discourse Meta: https://meta.discourse.org
- n8n Community: https://community.n8n.io

---

## Version Information

- **Package:** `@discourse/mcp`
- **Current Version:** `0.1.9`
- **Node Requirement:** >= 18
- **MCP SDK:** `@modelcontextprotocol/sdk` ^1.0.0

Last updated: 2025-11-17
