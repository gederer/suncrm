# Better Auth Integration - Product Requirements Document

## Document Information
**Product Name:** SunCRM Authentication System with Better Auth  
**Version:** 1.0  
**Document Version:** 1.0  
**Date:** January 2025  
**Status:** Draft  
**Author:** Product Team  
**Stakeholders:** Engineering, Security, DevOps, QA, Product Management

## Executive Summary

This document outlines the comprehensive integration of Better Auth with SunCRM using the Convex Better Auth adapter. The authentication system will provide secure, scalable, and role-based access control for all SunCRM users including Setters, Consultants, Managers, and Administrators. Given the alpha status of the Convex integration, this PRD includes risk mitigation strategies, comprehensive testing requirements, and fallback options.

### Key Objectives
- Implement secure, modern authentication for web and mobile platforms
- Enable role-based access control (RBAC) for different user types
- Support real-time session management for Sundialer features
- Ensure compliance with security standards
- Provide seamless user experience with SSO capabilities
- Enable multi-organization support for enterprise clients

## Current State Analysis

### Existing Challenges
1. **No Authentication System**: SunCRM currently lacks user authentication
2. **Security Requirements**: Need enterprise-grade security for sensitive customer data
3. **Multiple User Types**: Complex role hierarchy requiring granular permissions
4. **Real-time Features**: Sundialer requires persistent, secure sessions
5. **Multi-platform Access**: Support needed for web, mobile, and API access

### User Roles & Requirements

#### User Hierarchy
```
Admins
├── Sales Managers
│   ├── Setter Managers
│   │   └── Setters
│   └── Consultant Managers
│       └── Consultants
└── Lead Managers
```

#### Role Capabilities
- **Setters**: Make calls, set appointments, update lead data
- **Consultants**: View appointments, manage sales pipeline, create proposals
- **Setter Managers**: Monitor setter performance, manage campaigns, listen to calls
- **Consultant Managers**: Territory management, consultant scheduling
- **Lead Managers**: Import leads, manage vendors, configure campaigns
- **Sales Managers**: Full sales operations oversight
- **Admins**: System configuration, user management, compliance

## Solution Architecture

### Technical Stack
- **Authentication**: Better Auth v1.3.7+
- **Integration**: @convex-dev/better-auth (alpha)
- **Database**: Convex real-time database
- **Frontend**: Next.js 15 with React 19
- **Session Management**: Better Auth sessions with Convex real-time sync
- **Security**: JWT tokens, refresh tokens, HTTPS-only

### Integration Approach

#### Phase 1: Core Authentication (Weeks 1-3)
- Basic Better Auth setup with Convex adapter
- Email/password authentication
- Session management
- User creation and profile management

#### Phase 2: Advanced Features (Weeks 4-6)
- Social login (Google, Microsoft)
- Two-factor authentication
- Password reset flows
- Remember me functionality

#### Phase 3: RBAC Implementation (Weeks 7-9)
- Role hierarchy implementation
- Permission system
- Organization/campaign access control
- API authorization

#### Phase 4: Enterprise Features (Weeks 10-12)
- Single Sign-On (SSO)
- Multi-organization support
- Audit logging
- Advanced security features

## Detailed Requirements

### 1. Authentication Methods

#### 1.1 Email/Password Authentication
```typescript
interface EmailPasswordAuth {
  features: {
    signup: {
      enabled: true,
      requireEmailVerification: true,
      passwordRequirements: {
        minLength: 12,
        requireUppercase: true,
        requireLowercase: true,
        requireNumbers: true,
        requireSpecialChars: true
      },
      customFields: {
        firstName: required,
        lastName: required,
        phone: optional,
        role: required // Set during invitation
      }
    },
    signin: {
      rememberMe: true,
      maxAttempts: 5,
      lockoutDuration: 15, // minutes
      supportEmailOrUsername: false // Email only
    },
    passwordReset: {
      tokenExpiration: 60, // minutes
      requireOldPassword: false,
      notifyOnReset: true
    }
  }
}
```

#### 1.2 Social Authentication
```typescript
interface SocialAuth {
  providers: {
    google: {
      enabled: true,
      clientId: process.env.GOOGLE_CLIENT_ID,
      allowedDomains: ['*'], // Or restrict to company domains
      autoLinkAccounts: true
    },
    microsoft: {
      enabled: true,
      clientId: process.env.MICROSOFT_CLIENT_ID,
      tenantId: 'common', // Support personal and work accounts
      autoLinkAccounts: true
    }
  },
  onboarding: {
    requireProfileCompletion: true,
    assignDefaultRole: 'pending_approval'
  }
}
```

#### 1.3 Two-Factor Authentication
```typescript
interface TwoFactorAuth {
  methods: {
    totp: {
      enabled: true,
      appName: 'SunCRM',
      issuer: 'SunCRM Solar Sales'
    },
    sms: {
      enabled: true,
      provider: 'twilio',
      fallbackToEmail: true
    },
    email: {
      enabled: true,
      codeLength: 6,
      expiration: 10 // minutes
    }
  },
  enforcement: {
    requiredForRoles: ['admin', 'manager'],
    optionalForRoles: ['setter', 'consultant'],
    gracePeriod: 7 // days
  }
}
```

### 2. User Management

#### 2.1 User Data Model
```typescript
// Better Auth user table (managed by Better Auth)
interface BetterAuthUser {
  id: string,
  email: string,
  emailVerified: boolean,
  createdAt: Date,
  updatedAt: Date
}

// SunCRM user table (application-specific)
interface SunCRMUser {
  id: string, // Links to BetterAuthUser.id
  firstName: string,
  lastName: string,
  phone?: string,
  role: UserRole,
  organizationId?: string,
  managerId?: string,
  avatar?: string,
  
  // Setter-specific fields
  campaignId?: string,
  dailyCallTarget?: number,
  performanceScore?: number,
  
  // Consultant-specific fields
  territoryId?: string,
  certifications?: string[],
  commissionRate?: number,
  
  // Metadata
  status: 'active' | 'inactive' | 'suspended',
  lastActive: Date,
  preferences: UserPreferences,
  createdAt: Date,
  updatedAt: Date
}

interface UserRole {
  id: string,
  name: 'admin' | 'sales_manager' | 'lead_manager' | 'setter_manager' | 'consultant_manager' | 'setter' | 'consultant',
  permissions: Permission[],
  inheritsFrom?: string // Parent role ID
}
```

#### 2.2 User Lifecycle Management
```typescript
interface UserLifecycle {
  invitation: {
    sendInviteEmail: true,
    inviteExpiration: 7, // days
    allowReinvite: true,
    preAssignRole: true,
    preAssignManager: true
  },
  onboarding: {
    steps: [
      'verify_email',
      'complete_profile',
      'set_password',
      'enable_2fa', // Optional/required based on role
      'acknowledge_policies'
    ],
    trackProgress: true,
    sendWelcomeEmail: true
  },
  suspension: {
    reasons: ['security', 'performance', 'compliance', 'voluntary'],
    preserveData: true,
    notifyManager: true,
    revokeApiKeys: true
  },
  deletion: {
    softDelete: true,
    anonymizeData: true,
    retentionPeriod: 90, // days
    requireConfirmation: true
  }
}
```

### 3. Session Management

#### 3.1 Session Configuration
```typescript
interface SessionConfig {
  session: {
    type: 'jwt', // JWT for stateless, scalable sessions
    expiresIn: '24h',
    refreshTokenExpiresIn: '30d',
    slidingWindow: true, // Extend on activity
    
    // Device management
    multiDevice: {
      enabled: true,
      maxDevices: 5,
      trackDeviceInfo: true,
      allowManualRevoke: true
    },
    
    // Security
    security: {
      sameSite: 'strict',
      secure: true, // HTTPS only
      httpOnly: true,
      detectAnomalies: true
    }
  },
  
  // Real-time session sync with Convex
  convexSync: {
    syncActiveStatus: true,
    broadcastPresence: true,
    trackLocation: false // Privacy
  }
}
```

#### 3.2 Real-time Session Features
```typescript
interface RealtimeSession {
  // For Sundialer active setter tracking
  setterSession: {
    campaignId: string,
    status: 'active' | 'paused' | 'offline',
    activeCallId?: string,
    lastActivity: Date,
    dailyStats: {
      calls: number,
      completedScripts: number,
      appointments: number
    }
  },
  
  // Presence for managers
  presence: {
    online: boolean,
    lastSeen: Date,
    currentPage?: string,
    activity?: string // 'on_call', 'in_meeting', etc.
  }
}
```

### 4. Role-Based Access Control (RBAC)

#### 4.1 Permission System
```typescript
interface Permission {
  resource: string, // 'leads', 'appointments', 'users', etc.
  action: string, // 'create', 'read', 'update', 'delete', 'export'
  scope: 'own' | 'team' | 'organization' | 'global',
  conditions?: {
    timeRestriction?: TimeWindow,
    ipRestriction?: string[],
    requiresMFA?: boolean
  }
}

// Permission definitions
const permissions = {
  // Setter permissions
  setter: [
    { resource: 'leads', action: 'read', scope: 'own' },
    { resource: 'leads', action: 'update', scope: 'own' },
    { resource: 'appointments', action: 'create', scope: 'own' },
    { resource: 'calls', action: 'create', scope: 'own' },
    { resource: 'notes', action: '*', scope: 'own' }
  ],
  
  // Setter Manager permissions (inherits setter + more)
  setter_manager: [
    ...permissions.setter,
    { resource: 'setters', action: 'read', scope: 'team' },
    { resource: 'calls', action: 'listen', scope: 'team' },
    { resource: 'campaigns', action: 'manage', scope: 'team' },
    { resource: 'reports', action: 'view', scope: 'team' }
  ],
  
  // Admin permissions
  admin: [
    { resource: '*', action: '*', scope: 'global' }
  ]
}
```

#### 4.2 Authorization Middleware
```typescript
// Convex function authorization
export const authorizedMutation = customMutation({
  args: { /* ... */ },
  handler: async (ctx, args) => {
    const user = await ctx.auth.getUserIdentity();
    if (!user) throw new Error('Unauthorized');
    
    const sunCRMUser = await ctx.db
      .query('users')
      .withIndex('by_auth_id', q => q.eq('authId', user.subject))
      .unique();
    
    if (!hasPermission(sunCRMUser, 'resource', 'action')) {
      throw new Error('Insufficient permissions');
    }
    
    // Proceed with mutation
  }
});
```

### 5. Organization & Multi-Tenancy

#### 5.1 Organization Structure
```typescript
interface Organization {
  id: string,
  name: string,
  type: 'enterprise' | 'franchise' | 'single',
  
  // Hierarchy
  parentOrgId?: string, // For franchises
  childOrgIds: string[],
  
  // Settings
  settings: {
    branding: {
      logo?: string,
      primaryColor?: string,
      emailTemplates?: object
    },
    features: {
      ssoEnabled: boolean,
      customDomain?: string,
      apiAccess: boolean
    },
    limits: {
      maxUsers: number,
      maxLeadsPerMonth: number,
      maxCampaigns: number
    }
  },
  
  // Billing
  subscription: {
    plan: 'starter' | 'professional' | 'enterprise',
    status: 'active' | 'suspended' | 'cancelled',
    billingEmail: string
  }
}
```

#### 5.2 Campaign Access Control
```typescript
interface Campaign {
  id: string,
  organizationId: string,
  name: string,
  
  // Access control
  access: {
    setterGroups: string[], // Groups allowed to join
    managers: string[], // User IDs with management access
    visibility: 'public' | 'private' | 'invite_only'
  },
  
  // Setter assignment
  setterAssignment: {
    mode: 'open' | 'assigned' | 'approval_required',
    maxSetters: number,
    currentSetters: string[]
  }
}
```

### 6. Security Requirements

#### 6.1 Security Policies
```typescript
interface SecurityPolicies {
  password: {
    minLength: 12,
    maxLength: 128,
    requireComplexity: true,
    preventReuse: 5, // Last 5 passwords
    expirationDays: 90,
    warningDays: 14
  },
  
  session: {
    maxIdleTime: 30, // minutes
    absoluteTimeout: 12, // hours
    requireReauthFor: ['password_change', 'user_management', 'export_data']
  },
  
  access: {
    ipWhitelist: string[], // Optional IP restrictions
    geofencing: boolean, // Restrict by location
    deviceTrust: boolean, // Require device registration
    anomalyDetection: true
  },
  
  audit: {
    logAllAuth: true,
    logSensitiveActions: true,
    retentionDays: 365,
    encryptLogs: true
  }
}
```

#### 6.2 Compliance Features
```typescript
interface ComplianceFeatures {
  gdpr: {
    dataPortability: true,
    rightToErasure: true,
    consentTracking: true,
    dataMinimization: true
  },
  
  sox: {
    segregationOfDuties: true,
    auditTrails: true,
    accessReviews: 'quarterly'
  },
  
  pci: {
    // Even though not handling payments directly
    secureTransmission: true,
    encryption: 'AES-256',
    keyRotation: 'annual'
  }
}
```

### 7. Integration Implementation

#### 7.1 File Structure
```
/convex
  /auth
    - config.ts         # Better Auth configuration
    - schema.ts         # Auth-related schema
    - hooks.ts          # User lifecycle hooks
    - permissions.ts    # RBAC definitions
  /auth.ts             # Main auth setup
  
/src
  /lib
    /auth
      - client.ts      # Auth client setup
      - provider.tsx   # Auth context provider
      - hooks.ts       # useAuth, useSession hooks
  /middleware
    - auth.ts          # Next.js middleware
  /components
    /auth
      - LoginForm.tsx
      - SignupForm.tsx
      - ForgotPassword.tsx
      - TwoFactorSetup.tsx
```

#### 7.2 Configuration Files
```typescript
// convex/auth.config.ts
import { betterAuth } from '@convex-dev/better-auth';

export const authConfig = {
  // Database
  database: {
    type: 'convex',
    convex: {
      deployment: process.env.CONVEX_DEPLOYMENT,
      adminKey: process.env.CONVEX_ADMIN_KEY
    }
  },
  
  // Email
  email: {
    from: 'noreply@suncrm.com',
    provider: 'sendgrid',
    sendgrid: {
      apiKey: process.env.SENDGRID_API_KEY
    }
  },
  
  // Providers
  providers: {
    credentials: {
      enabled: true,
      // ... configuration
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET
    }
  },
  
  // Plugins
  plugins: [
    twoFactor(),
    organization(),
    rateLimit({
      window: 15 * 60, // 15 minutes
      max: 5, // attempts
      storage: 'memory' // or 'redis' for production
    })
  ],
  
  // Hooks
  hooks: {
    onCreateUser: async (user) => {
      // Create SunCRM user record
      await createSunCRMUser(user);
    },
    onUpdateUser: async (user) => {
      // Sync with SunCRM user
      await updateSunCRMUser(user);
    },
    onDeleteUser: async (userId) => {
      // Soft delete SunCRM user
      await softDeleteSunCRMUser(userId);
    }
  }
};
```

### 8. Testing Strategy

#### 8.1 Test Coverage Requirements
- **Unit Tests**: 90% coverage for auth logic
- **Integration Tests**: All auth flows
- **E2E Tests**: Critical user journeys
- **Security Tests**: Penetration testing
- **Performance Tests**: Load testing for 10,000 concurrent users

#### 8.2 Test Scenarios
```typescript
interface TestScenarios {
  authentication: [
    'successful_login',
    'failed_login_invalid_credentials',
    'failed_login_locked_account',
    'password_reset_flow',
    'email_verification_flow',
    'social_login_flow',
    '2fa_setup_and_login',
    'session_expiration',
    'refresh_token_flow',
    'concurrent_device_login',
    'logout_all_devices'
  ],
  
  authorization: [
    'role_based_access',
    'permission_inheritance',
    'organization_isolation',
    'campaign_access_control',
    'api_key_authentication',
    'resource_ownership_validation'
  ],
  
  security: [
    'brute_force_protection',
    'sql_injection_prevention',
    'xss_prevention',
    'csrf_protection',
    'session_hijacking_prevention',
    'password_policy_enforcement',
    'audit_logging_verification'
  ],
  
  performance: [
    'login_response_time',
    'token_generation_speed',
    'permission_check_performance',
    'concurrent_auth_requests',
    'database_query_optimization'
  ]
}
```

#### 8.3 Test Implementation
```typescript
// Example test suite
describe('Authentication Integration', () => {
  describe('Email/Password Login', () => {
    it('should successfully authenticate valid credentials', async () => {
      const response = await authClient.signIn({
        email: 'test@example.com',
        password: 'ValidPassword123!'
      });
      
      expect(response.user).toBeDefined();
      expect(response.session).toBeDefined();
      expect(response.session.expiresAt).toBeInTheFuture();
    });
    
    it('should enforce rate limiting after failed attempts', async () => {
      for (let i = 0; i < 5; i++) {
        await authClient.signIn({
          email: 'test@example.com',
          password: 'wrong'
        });
      }
      
      const response = await authClient.signIn({
        email: 'test@example.com',
        password: 'ValidPassword123!'
      });
      
      expect(response.error).toBe('Too many attempts');
    });
  });
  
  describe('RBAC', () => {
    it('should enforce setter permissions', async () => {
      const setter = await createTestUser('setter');
      const ctx = await authenticate(setter);
      
      // Should succeed - own lead
      await expect(
        updateLead(ctx, { leadId: setter.assignedLeads[0] })
      ).resolves.toBeDefined();
      
      // Should fail - other's lead
      await expect(
        updateLead(ctx, { leadId: 'other-lead-id' })
      ).rejects.toThrow('Insufficient permissions');
    });
  });
});
```

### 9. Migration & Rollback Plan

#### 9.1 Migration Strategy
Given the alpha status of Convex Better Auth integration:

```typescript
interface MigrationPlan {
  phases: [
    {
      phase: 'Development Testing',
      duration: '2 weeks',
      environment: 'development',
      rollbackRisk: 'low',
      successCriteria: [
        'All auth flows working',
        'No critical bugs',
        'Performance acceptable'
      ]
    },
    {
      phase: 'Staging Deployment',
      duration: '2 weeks',
      environment: 'staging',
      rollbackRisk: 'medium',
      successCriteria: [
        'Load testing passed',
        'Security audit passed',
        'Integration tests passing'
      ]
    },
    {
      phase: 'Limited Production',
      duration: '1 week',
      environment: 'production',
      userPercentage: 10,
      rollbackRisk: 'high',
      successCriteria: [
        'No critical issues',
        'User satisfaction > 90%',
        'Performance SLAs met'
      ]
    },
    {
      phase: 'Full Production',
      duration: 'ongoing',
      environment: 'production',
      userPercentage: 100,
      rollbackRisk: 'critical'
    }
  ]
}
```

#### 9.2 Fallback Options
```typescript
interface FallbackStrategy {
  primary: 'convex-better-auth',
  
  fallbacks: [
    {
      trigger: 'critical_bug_in_alpha',
      solution: 'Implement Better Auth with custom Convex adapter',
      effort: 'medium',
      timeline: '1 week'
    },
    {
      trigger: 'performance_issues',
      solution: 'Move auth to edge functions, keep session in Convex',
      effort: 'high',
      timeline: '2 weeks'
    },
    {
      trigger: 'complete_failure',
      solution: 'Switch to Clerk or Auth0 with Convex sync',
      effort: 'medium',
      timeline: '1 week'
    }
  ],
  
  rollback: {
    procedure: [
      'Enable maintenance mode',
      'Export user data',
      'Deploy fallback solution',
      'Migrate sessions',
      'Verify all users can login',
      'Disable maintenance mode'
    ],
    estimatedDowntime: '2 hours',
    dataLossRisk: 'none'
  }
}
```

### 10. Monitoring & Observability

#### 10.1 Metrics to Track
```typescript
interface AuthMetrics {
  performance: {
    loginTime: Histogram,
    tokenGenerationTime: Histogram,
    permissionCheckTime: Histogram,
    sessionCreationTime: Histogram
  },
  
  reliability: {
    authSuccessRate: Gauge,
    authFailureRate: Gauge,
    timeoutRate: Gauge,
    errorRate: Gauge
  },
  
  security: {
    failedLoginAttempts: Counter,
    suspiciousActivity: Counter,
    blockedRequests: Counter,
    passwordResets: Counter
  },
  
  usage: {
    dailyActiveUsers: Gauge,
    concurrentSessions: Gauge,
    newRegistrations: Counter,
    socialLoginUsage: Counter
  }
}
```

#### 10.2 Alerting Rules
```yaml
alerts:
  - name: HighAuthFailureRate
    condition: auth_failure_rate > 0.1
    severity: critical
    action: page_oncall
    
  - name: SuspiciousLoginActivity
    condition: suspicious_activity > 10 per 5m
    severity: high
    action: notify_security
    
  - name: SlowAuthentication
    condition: p95_login_time > 2s
    severity: medium
    action: notify_engineering
    
  - name: SessionAnomalies
    condition: session_creation_failures > 5 per 1m
    severity: high
    action: investigate_immediately
```

## Success Criteria

### Launch Metrics
- **Authentication Success Rate**: > 99.5%
- **Average Login Time**: < 1 second
- **2FA Adoption**: > 50% for managers, > 20% for users
- **Session Reliability**: > 99.9% uptime
- **Security Incidents**: 0 breaches

### Long-term Goals
- **User Satisfaction**: > 90% satisfaction with auth experience
- **Support Tickets**: < 5% of tickets auth-related
- **Compliance**: Pass all security audits
- **Scalability**: Support 100,000+ concurrent users
- **Integration**: Seamless SSO with enterprise clients

## Risk Analysis

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|---------|------------|
| Alpha integration instability | High | High | Comprehensive testing, fallback options ready |
| Performance degradation | Medium | High | Load testing, caching, optimization |
| Security vulnerabilities | Low | Critical | Security audit, penetration testing |
| Data loss during migration | Low | Critical | Backup strategy, gradual rollout |
| Convex-Better Auth incompatibility | Medium | High | Custom adapter development plan |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|---------|------------|
| User adoption resistance | Low | Medium | Training, intuitive UX, gradual rollout |
| Compliance violations | Low | High | Legal review, audit trails |
| Vendor lock-in | Medium | Medium | Abstraction layer, portable design |
| Cost overruns | Low | Medium | Phased approach, monitoring |

## Implementation Timeline

### Phase 1: Foundation (Weeks 1-3)
- Set up development environment
- Implement basic Better Auth with Convex
- Create user registration and login
- Basic session management
- Initial testing framework

### Phase 2: Core Features (Weeks 4-6)
- Social authentication
- Password reset flows
- Email verification
- User profile management
- Session persistence

### Phase 3: Security & RBAC (Weeks 7-9)
- Two-factor authentication
- Role-based permissions
- Organization support
- Campaign access control
- Security hardening

### Phase 4: Advanced Features (Weeks 10-12)
- SSO implementation
- Advanced session management
- Audit logging
- Performance optimization
- API authentication

### Phase 5: Testing & Deployment (Weeks 13-15)
- Comprehensive testing
- Security audit
- Load testing
- Documentation
- Training materials

### Phase 6: Launch (Week 16)
- Staged rollout
- Monitoring setup
- Support preparation
- Go-live

## Appendices

### A. Environment Variables
```env
# Better Auth
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=

# Convex
CONVEX_DEPLOYMENT=
CONVEX_ADMIN_KEY=
NEXT_PUBLIC_CONVEX_URL=

# OAuth Providers
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=

# Email
SENDGRID_API_KEY=
EMAIL_FROM=

# Security
ENCRYPTION_KEY=
JWT_SECRET=

# Features
ENABLE_2FA=true
ENABLE_SSO=false
ENABLE_SOCIAL_LOGIN=true
```

### B. Database Schema
```typescript
// Users collection
defineTable({
  authId: v.string(), // Better Auth user ID
  email: v.string(),
  firstName: v.string(),
  lastName: v.string(),
  role: v.string(),
  organizationId: v.optional(v.string()),
  status: v.string(),
  createdAt: v.number(),
  updatedAt: v.number()
}).index("by_auth_id", ["authId"])
  .index("by_email", ["email"])
  .index("by_organization", ["organizationId"])

// Sessions collection
defineTable({
  userId: v.string(),
  token: v.string(),
  deviceInfo: v.optional(v.object({...})),
  expiresAt: v.number(),
  createdAt: v.number()
}).index("by_user", ["userId"])
  .index("by_token", ["token"])

// Permissions collection
defineTable({
  roleId: v.string(),
  resource: v.string(),
  action: v.string(),
  scope: v.string()
}).index("by_role", ["roleId"])
```

### C. API Endpoints
```typescript
// Authentication endpoints
POST   /api/auth/signup
POST   /api/auth/signin
POST   /api/auth/signout
POST   /api/auth/refresh
POST   /api/auth/forgot-password
POST   /api/auth/reset-password
POST   /api/auth/verify-email
POST   /api/auth/resend-verification

// User management
GET    /api/users/me
PATCH  /api/users/me
DELETE /api/users/me
POST   /api/users/change-password
POST   /api/users/enable-2fa
POST   /api/users/verify-2fa

// Session management
GET    /api/sessions
DELETE /api/sessions/:id
DELETE /api/sessions/all
```

## Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Product Manager | | | |
| Engineering Lead | | | |
| Security Officer | | | |
| QA Lead | | | |
| DevOps Lead | | | |

---

*This document is version controlled and subject to change. Please refer to the latest version in the document management system.*