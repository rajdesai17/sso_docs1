# Single Sign-On (SSO) Integration Documentation

## Overview

This document outlines how to integrate your application with our Single Sign-On (SSO) authentication system. Our SSO solution provides a seamless authentication experience for users across multiple applications, allowing them to log in once and access all integrated services without re-entering credentials.

## Key Concepts

- **Single Sign-On**: Users authenticate once and gain access to multiple applications without additional login prompts.
- **Single Logout**: When users log out from one application, they are automatically logged out from all integrated applications.
- **Tokens**: The SSO system uses three types of tokens:
  - `access_token`: Short-lived token for API authorization
  - `refresh_token`: Long-lived token to obtain new access tokens
  - `trusted_device_token`: Optional token for remembered devices to bypass MFA

## How SSO Works

1. **Authentication**: Users log in through our centralized authentication service
2. **Token Issuance**: The SSO system issues tokens upon successful authentication
3. **Cross-Domain Cookie Setting**: Tokens are shared across domains using secure iframes
4. **Redirection**: Users are redirected back to the originating application
5. **Session Validation**: Applications validate tokens to maintain the session

## Integration Requirements

### Prerequisites

1. Register your application with our SSO service to obtain a `clientId`
2. Implement cookie management endpoints on your domain
3. Configure your application to handle redirects from our SSO service

### Client Registration

Contact our team to register your application. You'll need to provide:

- Application name
- Domain URL
- Platform type (web, mobile, etc.)
- Logo (optional)
- User agreement URL (optional)

Upon registration, you'll receive a unique `clientId` for your application.

## Integration Steps

### 1. Redirect to Login Page

When a user needs to authenticate, redirect them to our login page:

```
https://[sso-domain]/login?clientId=[your-clientId]&callBackURL=[your-redirect-url]
```

Parameters:
- `clientId`: Your registered client ID
- `callBackURL`: URL to redirect the user after authentication

### 2. Implement Cookie Management Endpoints

Your application must implement two API endpoints:

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
    res.cookie('accessToken', acTkn, { 
      httpOnly: true, 
      expires: new Date(parseInt(exp)), 
      secure: true 
    });
    
    res.cookie('refreshToken', rfTkn, { 
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
  res.clearCookie('accessToken');
  res.clearCookie('refreshToken');
  res.status(200).send('Cookies cleared successfully');
});
```

### 3. Handle Authentication Callback

After successful authentication, the user will be redirected to your `callBackURL` with session information. Your application should:

1. Validate the user's session
2. Redirect to the appropriate page within your application

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
- `formidiumRefreshToken`: The refresh token

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
                                        │          │
                                        │  Clear   │
                                        │ cookies  │
                                        │via iframe│
                                        │          │
                                        └──────────┘
                                               │
                                               │
                                               ▼
┌──────────┐    4. Redirect to login page   ┌──────────┐
│          │<─────────────────────────────────          │
│  Client  │                                 │  Client  │
│   App    │                                 │  Domain  │
└──────────┘                                 └──────────┘
```

## Common Integration Scenarios

### Web Application Integration

For a typical web application:

1. Redirect users to SSO login when authentication is required
2. Implement the required cookie management endpoints
3. Use the tokens for API authorization

### Mobile Application Integration

For mobile applications:

1. Use a WebView to handle the SSO login flow
2. Implement secure storage for tokens
3. Use tokens for API authorization

## Troubleshooting

### Common Issues

1. **"Invalid clientId" error**
   - Ensure your clientId is correctly registered and passed in the URL

2. **Cookies not being set**
   - Verify your setCookies endpoint is correctly implemented
   - Check that your domain is correctly registered in our system

3. **Cross-domain issues**
   - Ensure CORS is properly configured
   - Verify that your domain matches the registered domain

### Support

For integration support, contact our team at support@example.com.

## Security Considerations

- Always use HTTPS for all endpoints
- Implement proper CSRF protection
- Set appropriate cookie attributes (HttpOnly, Secure, SameSite)
- Never store sensitive token information in localStorage
- Implement proper token validation and expiration handling 
