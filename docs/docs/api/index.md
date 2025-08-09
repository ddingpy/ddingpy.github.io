---
layout: default
title: API Reference
nav_order: 4
has_children: true
permalink: /docs/api/
---

# API Reference
{: .no_toc }

Complete reference documentation for our RESTful API.
{: .fs-6 .fw-300 }

---

## Overview

Our API provides programmatic access to all platform features. This reference includes:

- Authentication methods
- Available endpoints
- Request/response formats
- Code examples in multiple languages
- Rate limiting information
- Error handling

## Base URL

All API requests should use the following base URL:

```
https://api.example.com/v2
```

{: .note }
> **Version Notice:** We recommend using API v2 for all new integrations. API v1 is deprecated and will be sunset on December 31, 2025.

## Authentication

### API Key Authentication

Include your API key in the request header:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
     https://api.example.com/v2/users
```

### OAuth 2.0

For applications requiring user authorization:

```javascript
const auth = new OAuth2Client({
  clientId: 'YOUR_CLIENT_ID',
  clientSecret: 'YOUR_CLIENT_SECRET',
  redirectUri: 'https://yourapp.com/callback'
});
```

## Quick Examples

### JavaScript/Node.js

```javascript
const axios = require('axios');

const client = axios.create({
  baseURL: 'https://api.example.com/v2',
  headers: {
    'Authorization': `Bearer ${API_KEY}`,
    'Content-Type': 'application/json'
  }
});

// GET request
const response = await client.get('/users');
console.log(response.data);

// POST request
const newUser = await client.post('/users', {
  name: 'John Doe',
  email: 'john@example.com'
});
```

### Python

```python
import requests

headers = {
    'Authorization': f'Bearer {API_KEY}',
    'Content-Type': 'application/json'
}

# GET request
response = requests.get(
    'https://api.example.com/v2/users',
    headers=headers
)
print(response.json())

# POST request
new_user = requests.post(
    'https://api.example.com/v2/users',
    headers=headers,
    json={
        'name': 'John Doe',
        'email': 'john@example.com'
    }
)
```

### cURL

```bash
# GET request
curl -X GET https://api.example.com/v2/users \
  -H "Authorization: Bearer YOUR_API_KEY"

# POST request
curl -X POST https://api.example.com/v2/users \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}'
```

## Core Endpoints

### Users

- `GET /users` - List all users
- `GET /users/{id}` - Get user details
- `POST /users` - Create new user
- `PUT /users/{id}` - Update user
- `DELETE /users/{id}` - Delete user

### Projects

- `GET /projects` - List all projects
- `GET /projects/{id}` - Get project details
- `POST /projects` - Create new project
- `PUT /projects/{id}` - Update project
- `DELETE /projects/{id}` - Delete project

### Resources

- `GET /resources` - List all resources
- `GET /resources/{id}` - Get resource details
- `POST /resources` - Create new resource
- `PUT /resources/{id}` - Update resource
- `DELETE /resources/{id}` - Delete resource

## Response Format

All API responses follow this structure:

### Success Response

```json
{
  "success": true,
  "data": {
    "id": "123",
    "name": "Example",
    "created_at": "2025-01-15T10:30:00Z"
  },
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 100
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "field": "email",
      "issue": "Invalid email format"
    }
  }
}
```

## Rate Limiting

API rate limits vary by plan:

| Plan | Requests per Hour | Burst Rate |
|:-----|:------------------|:-----------|
| Free | 1,000 | 20/second |
| Pro | 10,000 | 100/second |
| Enterprise | Unlimited | Custom |

Rate limit information is included in response headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1614556800
```

## Error Codes

| Code | Description | Action |
|:-----|:------------|:-------|
| 400 | Bad Request | Check request syntax |
| 401 | Unauthorized | Verify API key |
| 403 | Forbidden | Check permissions |
| 404 | Not Found | Verify endpoint/resource |
| 429 | Too Many Requests | Respect rate limits |
| 500 | Internal Server Error | Contact support |

## Pagination

List endpoints support pagination:

```
GET /users?page=2&per_page=50
```

Parameters:
- `page` - Page number (default: 1)
- `per_page` - Items per page (default: 20, max: 100)

## Filtering & Sorting

### Filtering

```
GET /users?status=active&role=admin
```

### Sorting

```
GET /users?sort=created_at&order=desc
```

## Webhooks

Configure webhooks to receive real-time notifications:

```json
POST /webhooks
{
  "url": "https://yourapp.com/webhook",
  "events": ["user.created", "user.updated"],
  "secret": "your_webhook_secret"
}
```

## SDKs & Libraries

Official SDKs available:

- [JavaScript/TypeScript](https://github.com/example/js-sdk)
- [Python](https://github.com/example/python-sdk)
- [Ruby](https://github.com/example/ruby-sdk)
- [Go](https://github.com/example/go-sdk)
- [PHP](https://github.com/example/php-sdk)

## API Playground

Try our API directly in your browser:

[Launch API Playground →](https://api.example.com/playground)

## Support

For API support:

- 📚 [API Documentation](/docs/api/)
- 💬 [Developer Forum](https://forum.example.com/developers)
- 📧 [API Support](mailto:api-support@example.com)
- 🐛 [Report API Issues](https://github.com/example/api/issues)

---

<div class="code-example" markdown="1">
**Pro Tip:** Use our Postman collection for quick API testing: [Download Collection](https://api.example.com/postman)
</div>