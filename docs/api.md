---
layout: single
title: "API Reference"
permalink: /api/
toc: true
toc_label: "API Contents"
toc_icon: "code"
toc_sticky: true
classes: wide
---

Complete API documentation for developers with examples, authentication guides, and best practices.

## Overview

Our RESTful API provides programmatic access to all platform features. This reference includes endpoints, request/response formats, authentication, and examples.

## Base URL

All API requests should be made to:

```
https://api.example.com/v2
```

## Authentication

### API Keys

Include your API key in the Authorization header:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
     https://api.example.com/v2/resource
```

### OAuth 2.0

For user-specific operations, use OAuth 2.0:

```javascript
const token = await getOAuthToken();
fetch('https://api.example.com/v2/user', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

## Quick Reference

### Core Endpoints

| Method | Endpoint | Description |
|:-------|:---------|:------------|
| GET | `/users` | List all users |
| GET | `/users/{id}` | Get specific user |
| POST | `/users` | Create new user |
| PUT | `/users/{id}` | Update user |
| DELETE | `/users/{id}` | Delete user |
| GET | `/projects` | List projects |
| POST | `/projects` | Create project |
| GET | `/resources` | List resources |

## Response Format

All responses follow this structure:

### Success Response

```json
{
  "success": true,
  "data": {
    // Response data here
  },
  "meta": {
    "timestamp": "2025-01-15T10:00:00Z",
    "version": "2.0"
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested resource was not found",
    "details": {
      "resource_id": "12345"
    }
  }
}
```

## Rate Limiting

API rate limits:

| Plan | Requests/Hour | Burst |
|:-----|:--------------|:------|
| Free | 1,000 | 10/sec |
| Pro | 10,000 | 100/sec |
| Enterprise | Unlimited | Unlimited |

Rate limit headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642248000
```

## Common Operations

### List Resources

```bash
GET /resources?page=1&limit=20&sort=created_at
```

Parameters:
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 20, max: 100)
- `sort` - Sort field (default: created_at)
- `order` - Sort order: asc/desc (default: desc)

### Create Resource

```bash
POST /resources
Content-Type: application/json

{
  "name": "My Resource",
  "type": "document",
  "metadata": {
    "key": "value"
  }
}
```

### Update Resource

```bash
PUT /resources/{id}
Content-Type: application/json

{
  "name": "Updated Name",
  "metadata": {
    "updated": true
  }
}
```

### Delete Resource

```bash
DELETE /resources/{id}
```

## Pagination

Paginated responses include pagination metadata:

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 100,
    "total_pages": 5,
    "next_page": 2,
    "prev_page": null
  }
}
```

## Filtering

Use query parameters to filter results:

```bash
GET /resources?status=active&type=document&created_after=2025-01-01
```

Common filters:
- `status` - Resource status
- `type` - Resource type
- `created_after` - Created after date
- `created_before` - Created before date
- `updated_after` - Updated after date
- `search` - Full-text search

## Webhooks

Configure webhooks to receive real-time updates:

### Create Webhook

```bash
POST /webhooks
Content-Type: application/json

{
  "url": "https://your-app.com/webhook",
  "events": ["resource.created", "resource.updated"],
  "secret": "your-webhook-secret"
}
```

### Webhook Payload

```json
{
  "event": "resource.created",
  "timestamp": "2025-01-15T10:00:00Z",
  "data": {
    "resource_id": "12345",
    "name": "New Resource"
  },
  "signature": "sha256=..."
}
```

### Verify Webhook Signature

```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hash = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return `sha256=${hash}` === signature;
}
```

## SDKs

Official SDKs available:

### JavaScript/Node.js

```bash
npm install @example/sdk
```

```javascript
const SDK = require('@example/sdk');
const client = new SDK({
  apiKey: 'YOUR_API_KEY'
});

const users = await client.users.list();
```

### Python

```bash
pip install example-sdk
```

```python
from example_sdk import Client

client = Client(api_key='YOUR_API_KEY')
users = client.users.list()
```

### Ruby

```bash
gem install example-sdk
```

```ruby
require 'example_sdk'

client = ExampleSDK::Client.new(api_key: 'YOUR_API_KEY')
users = client.users.list
```

## Error Codes

| Code | Description | Action |
|:-----|:------------|:-------|
| 400 | Bad Request | Check request format |
| 401 | Unauthorized | Verify API key |
| 403 | Forbidden | Check permissions |
| 404 | Not Found | Verify resource exists |
| 429 | Too Many Requests | Respect rate limits |
| 500 | Internal Server Error | Contact support |

## Testing

### Test Environment

Use the test environment for development:

```
https://api-test.example.com/v2
```

Test API keys start with `test_`.

### Postman Collection

Download our [Postman Collection](https://www.getpostman.com/collections/example) for easy testing.

### cURL Examples

```bash
# List resources
curl -X GET https://api.example.com/v2/resources \
  -H "Authorization: Bearer YOUR_API_KEY"

# Create resource
curl -X POST https://api.example.com/v2/resources \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Resource"}'

# Update resource
curl -X PUT https://api.example.com/v2/resources/123 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Resource"}'

# Delete resource
curl -X DELETE https://api.example.com/v2/resources/123 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

## Changelog

### Version 2.0 (Current)
- Added pagination support
- Improved error messages
- New webhook events
- Performance improvements

### Version 1.0
- Initial release
- Basic CRUD operations
- Authentication support

## Support

For API support:

- 📧 Email: [api@example.com](mailto:api@example.com)
- 📚 Documentation: You're here!
- 🐛 Issues: [GitHub](https://github.com/yourusername/yourrepository/issues)
- 💬 Discord: [Developer Community](https://discord.gg/example)

---

<small>API documentation last updated: {{ site.time | date: "%B %d, %Y" }}</small>