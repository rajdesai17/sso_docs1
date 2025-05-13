# Single Sign-On (SSO) Integration Documentation

## Overview

This document outlines how to integrate your application with our Single Sign-On (SSO) authentication system. Our SSO solution provides a seamless authentication experience for users across multiple applications, allowing them to log in once and access all integrated services without re-entering credentials.

## Key Concepts

- **Single Sign-On**: Users authenticate once and gain access to multiple applications without additional login prompts.
- **Single Logout**: When users log out from one application, they are automatically logged out from all integrated applications.
- **Tokens**: The SSO system uses three types of tokens:
  - `accessToken`: Short-lived token for API authorization
  - `refreshToken`: Long-lived token to obtain new access tokens
  - `trustDeviceToken`: Optional token for remembered devices to bypass MFA
- **Cookies**: The tokens are stored as cookies on the respective domains:
  - `formidiumAccessToken`: Cookie containing the access token
  - `formidiumRefreshToken`: Cookie containing the refresh token
  - `trustDeviceToken`: Cookie for trusted devices

## How SSO Works

1. **Authentication**: Users log in through our centralized authentication service
2. **Token Issuance**: The SSO system issues tokens upon successful authentication
3. **Cross-Domain Cookie Setting**: Tokens are shared across domains using secure iframes
4. **Redirection**: Users are redirected back to the originating application via the callBackURL
5. **Session Validation**: Applications validate tokens to maintain the session

## Integration Requirements

### Prerequisites

1. Register your application with our SSO service to obtain a `clientId`
2. Implement cookie management endpoints on your domain:
   - `/api/setCookies` - For setting cross-domain cookies
   - `/api/delete` - For clearing cookies during logout
3. Configure your application to handle redirects from our SSO service

### Client Registration

Contact our team to register your application in our client database. You'll need to provide:

- Application name
- Base URL (domain where your application is hosted)
- Platform type (web, mobile, etc.)
- Logo (optional)
- User agreement URL (optional)

Upon registration, you'll receive a unique `clientId` for your application, and your domain will be stored in our database for integration purposes.

## Integration Steps

### 1. Redirect to Login Page

When a user needs to authenticate, redirect them to our login page:

```
https://[sso-domain]/login?clientId=[your-clientId]&callBackURL=[your-redirect-url]
```

Parameters:
- `clientId`: Your registered client ID
- `callBackURL`: URL to redirect the user after authentication (must match your registered domain)

### 2. Implement Cookie Management Endpoints

Your application must implement two API endpoints at your base URL:

#### a. Set Cookies Endpoint

```
/api/setCookies
```

This endpoint receives encoded session data via iframe and sets cookies on your domain:

```javascript
// Example implementation (Express.js)
app.get('/api/setCookies', (req, res) => {
  const { session } = req.query;
  if (!session) return res.status(400).send('Session data required');
  
  try {
    const sessionData = JSON.parse(atob(session));
    const { acTkn, rfTkn, exp } = sessionData;
    
    // Set cookies on your domain
    res.cookie('formidiumAccessToken', acTkn, { 
      httpOnly: true, 
      expires: new Date(parseInt(exp)), 
      secure: true 
    });
    
    res.cookie('formidiumRefreshToken', rfTkn, { 
      httpOnly: true, 
      expires: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days 
      secure: true 
    });
    
    res.status(200).send('Cookies set successfully');
  } catch (error) {
    res.status(400).send('Invalid session data');
  }
});
```

#### b. Delete Cookies Endpoint

```
/api/delete
```

This endpoint clears all auth-related cookies from your domain:

```javascript
// Example implementation (Express.js)
app.get('/api/delete', (req, res) => {
  res.clearCookie('formidiumAccessToken');
  res.clearCookie('formidiumRefreshToken');
  res.clearCookie('trustDeviceToken'); // If applicable
  res.status(200).send('Cookies cleared successfully');
});
```

### 3. Handle Authentication Callback

After successful authentication, the user will be redirected to your `callBackURL` with session information. Our SSO system will:

1. Use an iframe to call your `/api/setCookies` endpoint to set cookies on your domain
2. Redirect the user to your application

Your application should be ready to receive the authenticated user and validate their session using the cookies.

## Cross-Domain Authentication Flow

Our SSO system uses iframes to facilitate cross-domain cookie setting. When a user authenticates:

1. The SSO service generates tokens
2. The tokens are encoded and passed to an iframe that loads your `/api/setCookies` endpoint
3. Your endpoint decodes the tokens and sets them as cookies on your domain
4. The user is redirected to your application with active cookies

This approach ensures that cookies can be set across different domains without violating same-origin policies.

## API Reference

### Authentication Endpoints

#### Login

```
POST /api/v1/authentication?clientId=[client-id]
```

Request body:
```json
{
  "username": "user@example.com",
  "password": "password123"
}
```

Response:
```json
{
  "status": "success",
  "message": "User logged in successfully",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1...",
    "refreshToken": "eyJhbGciOiJIUzI1...",
    "expiresIn": 3600
  }
}
```

#### Logout

```
DELETE /api/v1/authentication
```

Response:
```json
{
  "message": "Logged out successfully",
  "success": true
}
```

#### Validate Token

```
POST /api/v1/validate
```

Headers:
```
Authorization: Bearer [access_token]
```

Response:
```json
{
  "message": "Token is valid.",
  "valid": true
}
```

#### Refresh Access Token

```
POST /api/v1/refreshAccessToken
```

Cookies:
- `formidiumRefreshToken`: The refresh token cookie used to obtain a new access token

Response:
```json
{
  "status": "success",
  "message": "Access token refreshed successfully",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1...",
    "expiresIn": 3600
  }
}
```

### Client Management

#### Get Client

```
GET /api/v1/client?clientId=[client-id]
```

Response:
```json
{
  "message": "GET Clients",
  "client": [
    {
      "clientId": "example-app",
      "domain": "https://example.com",
      "platform": "web"
    }
  ],
  "success": true
}
```

## SSO Flow Diagrams

### Login Flow

```
┌──────────┐     1. Redirect to SSO     ┌──────────┐
│          │─────────────────────────────>          │
│  Client  │                             │   SSO    │
│   App    │                             │  Service │
│          │<─────────────────────────────          │
└──────────┘     2. Authentication       └──────────┘
      │                                        │
      │                                        │
      │                                        │
      │           3. Redirect back             │
      │<───────────────────────────────────────┘
      │
      │
┌─────▼────┐    4. Set cookies via iframe   ┌──────────┐
│          │─────────────────────────────────>          │
│  Client  │                                 │  Client  │
│   App    │                                 │  Domain  │
│          │<─────────────────────────────────          │
└──────────┘        5. Cookies set           └──────────┘
```

### Logout Flow

```
┌──────────┐     1. Logout request     ┌──────────┐
│          │─────────────────────────────>          │
│  Client  │                             │   SSO    │
│   App    │                             │  Service │
│          │<─────────────────────────────          │
└──────────┘     2. Clear SSO cookies    └──────────┘
                                               │
                                               │
                                               ▼
                                        ┌──────────┐
                                        │ For each │
                                        │registered│
                                        │  client  │
                                        └──────────┘
                                               │
                                               ▼
┌──────────┐    3. Clear cookies via iframe  ┌──────────┐
│          │<────────────────────────────────│          │
│ Client 1 │                                 │   SSO    │
│  Domain  │────────────────────────────────>│  Service │
└──────────┘       4. Cookies cleared        └──────────┘
                                               │
                                               ▼
┌──────────┐    5. Clear cookies via iframe  ┌──────────┐
│          │<────────────────────────────────│          │
│ Client 2 │                                 │   SSO    │
│  Domain  │────────────────────────────────>│  Service │
└──────────┘       6. Cookies cleared        └──────────┘
                                               │
                                               ▼
┌──────────┐    7. Redirect to login page   ┌──────────┐
│          │<─────────────────────────────────          │
│ Original │                                 │   SSO    │
│  Client  │                                 │  Service │
└──────────┘                                 └──────────┘
```

## Common Integration Scenarios

### Web Application Integration

For a typical web application:

1. Register your application with our SSO service
2. Implement the required cookie management endpoints at your base URL:
   - `/api/setCookies`
   - `/api/delete`
3. Redirect users to our SSO login page when authentication is required
4. Handle the redirect back to your application after authentication

### Development Environment Setup

During development, you can use the following:

1. SSO Service: Our development environment is available at `https://dev.[sso-domain]`
2. Testing: Use your development domain (e.g., `https://dev.yourdomain.com`) as the callBackURL
3. Ensure your development environment has the cookie management endpoints properly configured

## Troubleshooting

### Common Issues

1. **"Invalid clientId" error**
   - Ensure your clientId is correctly registered and passed in the URL
   - Verify your application is properly registered in our client database

2. **Cookies not being set**
   - Verify your `/api/setCookies` endpoint is correctly implemented
   - Check that your domain matches exactly what is registered in our system
   - Ensure your endpoint is accessible via iframe

3. **Cross-domain issues**
   - Ensure CORS is properly configured for iframe communication
   - Verify that your domain matches the registered domain
   - Check that cookies are being set with the correct domain attributes

4. **Logout not working across all applications**
   - Ensure your `/api/delete` endpoint is correctly implemented
   - Verify it's accessible via iframe

### Support

For integration support, contact our team at support@example.com.

## Security Considerations

- Always use HTTPS for all endpoints
- Implement proper CSRF protection
- Set appropriate cookie attributes (HttpOnly, Secure, SameSite)
- Never store sensitive token information in localStorage
- Implement proper token validation and expiration handling
- Ensure your `/api/setCookies` and `/api/delete` endpoints are protected against unauthorized access 
