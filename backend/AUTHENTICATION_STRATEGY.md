<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Authentication Strategy
## Secure JWT-Based Authentication Pattern

> **For AI:** This pattern defines authentication for user-facing and service-to-service communication.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Authentication Method:** JWT with RS256 (asymmetric signing)

**Token Lifetimes:**
- Access token: 15 minutes (short-lived for security)
- Refresh token: 7 days (long-lived, httpOnly cookie)

**Storage:**
- Access token: localStorage (client-side)
- Refresh token: httpOnly cookie (protected from XSS)

**Guards Available:**
- `JwtAuthGuard` - User authentication
- `ServiceAuthGuard` - Service-to-service auth
- `PermissionsGuard` - Role/permission-based access

---

## 🔐 Why RS256 (Not HS256)

**RS256 uses asymmetric keys:**
- Private key (signs tokens) - Only auth service has this
- Public key (verifies tokens) - All services can validate

**Security benefit:**
```
If service X is compromised:
  ❌ With HS256: Attacker can forge ANY token
  ✅ With RS256: Attacker can only verify, not forge
```

**This matters for microservices** - one compromised service can't impersonate users.

---

## 🚀 Implementation Guide

### Step 1: JWT Validation Setup

```typescript
// src/config/configuration.ts
export default () => ({
  auth: {
    jwtPublicKey: process.env.JWT_PUBLIC_KEY,  // From Vault
    jwtExpiresIn: '15m',
    refreshExpiresIn: '7d',
  }
})

// src/auth/jwt.strategy.ts
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { ExtractJwt, Strategy } from 'passport-jwt'
import { ConfigService } from '@nestjs/config'

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('auth.jwtPublicKey'),
      algorithms: ['RS256'],  // Explicit algorithm check
    })
  }

  async validate(payload: any) {
    // Payload already verified by Passport
    return {
      userId: payload.sub,
      email: payload.email,
      permissions: payload.permissions || [],
    }
  }
}
```

**Key points:**
- Extract token from `Authorization: Bearer <token>` header
- Verify signature using public key (stored in Vault)
- Return user object attached to request

---

### Step 2: Guards Implementation

#### JwtAuthGuard (User Authentication)

```typescript
// src/auth/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common'
import { AuthGuard } from '@nestjs/passport'

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// Usage in controllers:
import { Controller, Get, UseGuards } from '@nestjs/common'
import { JwtAuthGuard } from './auth/guards/jwt-auth.guard'

@Controller('users')
export class UsersController {
  @Get('profile')
  @UseGuards(JwtAuthGuard)  // Protected endpoint
  getProfile(@Request() req) {
    return req.user  // User from JWT payload
  }
}
```

#### PermissionsGuard (Fine-Grained Access)

```typescript
// src/auth/guards/permissions.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common'
import { Reflector } from '@nestjs/core'

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions',
      context.getHandler()
    )

    if (!requiredPermissions) return true

    const request = context.switchToHttp().getRequest()
    const user = request.user

    return requiredPermissions.every(permission =>
      user.permissions?.includes(permission)
    )
  }
}

// Usage with decorator:
import { SetMetadata } from '@nestjs/common'
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata('permissions', permissions)

// In controller:
@Delete(':id')
@UseGuards(JwtAuthGuard, PermissionsGuard)
@RequirePermissions('users:delete')
deleteUser(@Param('id') id: string) {
  // Only users with 'users:delete' permission can access
}
```

---

### Step 3: Token Refresh Flow

```typescript
// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  async login(@Body() loginDto: LoginDto, @Res() response: Response) {
    const { accessToken, refreshToken } = await this.authService.login(loginDto)

    // Set refresh token as httpOnly cookie (XSS protection)
    response.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days
    })

    // Return access token in response body
    return response.json({ accessToken })
  }

  @Post('refresh')
  async refresh(@Req() request: Request, @Res() response: Response) {
    const refreshToken = request.cookies['refreshToken']

    if (!refreshToken) {
      throw new UnauthorizedException('Refresh token not found')
    }

    const { accessToken, refreshToken: newRefreshToken } =
      await this.authService.refresh(refreshToken)

    // Rotate refresh token (security best practice)
    response.cookie('refreshToken', newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    })

    return response.json({ accessToken })
  }

  @Post('logout')
  async logout(@Res() response: Response) {
    // Clear refresh token cookie
    response.clearCookie('refreshToken')
    return response.json({ message: 'Logged out successfully' })
  }
}
```

**Refresh flow:**
1. Access token expires (15 min)
2. Client calls `/auth/refresh` with httpOnly cookie
3. Backend validates refresh token
4. Backend issues new access token + rotates refresh token
5. Client stores new access token in localStorage

---

### Step 4: Service-to-Service Authentication

For microservices calling each other:

```typescript
// src/auth/guards/service-auth.guard.ts
@Injectable()
export class ServiceAuthGuard implements CanActivate {
  constructor(private configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest()
    const apiKey = request.headers['x-api-key']

    if (!apiKey) return false

    // Validate against service API keys stored in Vault
    const validKeys = this.configService.get<string[]>('serviceApiKeys')
    return validKeys.includes(apiKey)
  }
}

// Usage:
@Controller('internal')
export class InternalController {
  @Post('process')
  @UseGuards(ServiceAuthGuard)  // Only other services can call
  processTask(@Body() taskDto: TaskDto) {
    // Internal endpoint
  }
}
```

**Service API keys stored in Vault:**
```
vault/services/video-processing/api-key
vault/services/mail-service/api-key
vault/services/registration/api-key
```

---

## ✅ Implementation Checklist

When adding authentication to a service:

- [ ] Get JWT public key from Vault
- [ ] Configure JWT module in NestJS
- [ ] Implement JwtStrategy for token validation
- [ ] Create JwtAuthGuard
- [ ] Create PermissionsGuard (if using RBAC)
- [ ] Implement token refresh endpoint
- [ ] Add httpOnly cookie for refresh token
- [ ] Implement logout (clear cookies)
- [ ] Add ServiceAuthGuard for internal endpoints
- [ ] Store service API keys in Vault
- [ ] Test token expiration and refresh flow
- [ ] Test permission-based access control
- [ ] Add error handling for auth failures

---

## 🔗 Related Patterns

→ [API_INTEGRATION.md](../frontend/API_INTEGRATION.md) - Frontend: Token injection in API calls
→ [MESSAGING_STRATEGY.md](./MESSAGING_STRATEGY.md) - Propagating user context in events
→ [OBSERVABILITY.md](./OBSERVABILITY.md) - Logging auth failures and security events

---

## 🚨 Security Considerations

### ✅ DO:
- Use RS256 (asymmetric) for JWT signing
- Store refresh tokens in httpOnly cookies
- Rotate refresh tokens on each use
- Validate token algorithms explicitly
- Use short access token lifetimes (15 min)
- Store secrets in Vault, never in code
- Use HTTPS in production (secure cookies)
- Implement rate limiting on login/refresh

### ❌ DON'T:
- Use HS256 in microservices (symmetric)
- Store refresh tokens in localStorage
- Reuse refresh tokens (security risk)
- Trust tokens without signature verification
- Use long-lived access tokens (1 hour+)
- Hardcode keys or secrets
- Allow refresh without token rotation
- Skip rate limiting (brute force risk)

---

## 📊 Token Payload Example

```json
{
  "sub": "user-uuid-here",
  "email": "user@example.com",
  "permissions": ["users:read", "users:write", "videos:upload"],
  "iat": 1704804000,
  "exp": 1704804900
}
```

**Fields:**
- `sub` - Subject (user ID)
- `email` - User email
- `permissions` - Array of permission strings
- `iat` - Issued at (Unix timestamp)
- `exp` - Expiration (Unix timestamp)

**Note:** Keep payload small (JWT sent with every request)

---

## 🎓 How AI Uses This Pattern

When you request: **"Add authentication to the user service"**

**AI will:**
1. Read this pattern
2. Extract JWT validation code
3. Implement all three guards (JWT, Permissions, Service)
4. Set up token refresh endpoint
5. Configure httpOnly cookies
6. Add Vault integration for public key
7. Include all checklist items
8. Add proper error handling

**Result:** Production-ready authentication in 15-20 minutes

**Without this pattern:** 2-3 hours of back-and-forth, likely missing security best practices

---

## 📝 Notes

- This is a **demo pattern** - simplified for demonstration
- Full production pattern includes: token revocation, MFA, OAuth integration
- See enterprise package for complete authentication implementation

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09
**Status:** Active

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
