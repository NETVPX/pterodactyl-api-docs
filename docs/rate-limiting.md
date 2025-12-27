# Rate Limiting

The Pterodactyl API implements rate limiting to ensure fair usage and maintain API performance for all users.

## Rate Limit Overview

- **Client API**: 240 requests per minute per API key
- **Application API**: 240 requests per minute per API key

## Rate Limit Headers

Every API response includes rate limit information in the headers:

```http
X-RateLimit-Limit: 240
X-RateLimit-Remaining: 237
X-RateLimit-Reset: 1640995200
```

### Header Explanations

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum number of requests allowed per minute |
| `X-RateLimit-Remaining` | Number of requests remaining in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the rate limit window resets |

## Rate Limit Exceeded Response

When you exceed the rate limit, you'll receive a `429 Too Many Requests` response:

```json
{
  "errors": [
    {
      "code": "TooManyRequestsHttpException",
      "status": "429",
      "detail": "Too many requests, please slow down."
    }
  ]
}
```

## Increasing the Rate Limit

You can configure the API rate limits on your Pterodactyl panel instance by editing the `.env` file. By default, these settings are not present, so you'll need to add them manually.

The `.env` file is typically located at:

```
/var/www/pterodactyl/.env
```

Add or update the following lines to set your desired rate limits (values shown are defaults):

```env
APP_API_APPLICATION_RATELIMIT=240
APP_API_CLIENT_RATELIMIT=240
```

- `APP_API_APPLICATION_RATELIMIT`: Sets the rate limit for the Application API (requests per minute per API key).
- `APP_API_CLIENT_RATELIMIT`: Sets the rate limit for the Client API (requests per minute per API key).

After making changes, restart your Pterodactyl panel for the new settings to take effect.

## Best Practices

### 1. Check Rate Limit Headers
Always monitor the rate limit headers in your application:

```javascript
const response = await fetch('https://your-panel.com/api/client', {
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Accept': 'Application/vnd.pterodactyl.v1+json'
  }
});

const remaining = response.headers.get('X-RateLimit-Remaining');
const reset = response.headers.get('X-RateLimit-Reset');

console.log(`Requests remaining: ${remaining}`);
console.log(`Rate limit resets at: ${new Date(reset * 1000)}`);
```

### 2. Implement Exponential Backoff
When you receive a 429 response, implement exponential backoff:

```javascript
async function makeRequest(url, options, retries = 3) {
  try {
    const response = await fetch(url, options);
    
    if (response.status === 429 && retries > 0) {
      const retryAfter = response.headers.get('Retry-After') || 60;
      await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
      return makeRequest(url, options, retries - 1);
    }
    
    return response;
  } catch (error) {
    if (retries > 0) {
      await new Promise(resolve => setTimeout(resolve, 1000 * (4 - retries)));
      return makeRequest(url, options, retries - 1);
    }
    throw error;
  }
}
```

### 3. Batch Operations
When possible, batch multiple operations into single requests rather than making many individual requests.

### 4. Cache Responses
Cache API responses when appropriate to reduce the number of requests needed.

## Rate Limiting by IP vs API Key

Rate limits are applied **per API key**, not per IP address. This means:

- Multiple API keys from the same IP can each have their own rate limit
- Using the same API key from multiple IPs shares the same rate limit counter

## Exceeding Limits

If you consistently exceed rate limits, consider:

1. **Optimizing your requests** - Only request the data you need
2. **Implementing caching** - Store responses locally when possible  
3. **Contacting support** - If you have legitimate high-volume needs

## Monitoring Rate Limits

### Response Header Example
```bash
curl -I "https://your-panel.com/api/client" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Accept: Application/vnd.pterodactyl.v1+json"

# Response headers:
# X-RateLimit-Limit: 240
# X-RateLimit-Remaining: 239
# X-RateLimit-Reset: 1640995260
```

### Rate Limit Status Check
You can check your current rate limit status without affecting your remaining requests by making a `HEAD` request to any endpoint.

## Next Steps

- Learn about [Error Handling](./error-handling)
- Explore the [Client API](./api/client) endpoints
- Review [Authentication](./authentication) requirements 