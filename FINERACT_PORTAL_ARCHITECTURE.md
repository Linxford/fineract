# Apache Fineract - Portal Architecture Guide

## 🌐 Fineract Portal Architecture

### 1. **Core API Portal** (Backend)
- **URL**: `https://localhost:8443/fineract-provider/api/v1/`
- **Type**: RESTful API
- **Purpose**: Core banking operations
- **Access**: Direct API access for all operations

### 2. **Community App** (Traditional Web UI)
- **URL**: `http://localhost:9090/?baseApiUrl=https://localhost:8443/fineract-provider&tenantIdentifier=default`
- **Type**: AngularJS Web Application
- **Purpose**: Full-featured web interface for staff
- **Repository**: `https://github.com/openMF/community-app/`
- **Features**: Complete banking operations UI

### 3. **Web App** (Next Generation UI)
- **Type**: Modern Web Application (React/Angular)
- **Purpose**: Redesigned user interface
- **Repository**: `https://github.com/openMF/web-app`
- **Status**: Next-gen UI rewrite

### 4. **Self-Service Portal**
- **URL**: `/v1/self/*` API endpoints
- **Purpose**: Customer self-service operations
- **Features**:
  - Account viewing
  - Transaction history
  - Loan applications
  - Mobile/tablet friendly

### 5. **Swagger UI** (API Documentation)
- **URL**: `https://localhost:8443/fineract-provider/swagger-ui/index.html`
- **Purpose**: Interactive API documentation
- **Features**: Test APIs directly from browser

### 6. **Legacy API Documentation**
- **URL**: `https://localhost:8443/fineract-provider/legacy-docs/apiLive.htm`
- **Purpose**: Traditional API documentation

## 🔄 How It Works

### Architecture Flow:
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Web UIs       │     │  Mobile Apps    │     │ Third-Party     │
│ (Community App) │     │  (Android)      │     │ Integrations    │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                         │
         └───────────────────────┴─────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   Apache Fineract       │
                    │   RESTful API Layer     │
                    │  (/api/v1/* endpoints)  │
                    └────────────┬────────────┘
                                 │
         ┌───────────┬───────────┼───────────┬───────────┐
         │           │           │           │           │
    ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
    │Clients  │ │ Loans   │ │Savings  │ │Accounting│ │  SMS    │
    │Module   │ │ Module  │ │Module   │ │ Module   │ │ Module  │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

### Multi-Portal Access Patterns:

1. **Staff Access** (Back Office)
   ```
   Staff Login → Community App → API → Database
   ```
   - Full access to all operations
   - Role-based permissions
   - Maker-checker workflows

2. **Customer Access** (Self-Service)
   ```
   Customer → Self-Service Portal → Self API (/v1/self/*) → Limited Operations
   ```
   - View-only for most data
   - Limited transactions
   - No administrative functions

3. **Mobile Access**
   ```
   Mobile App → API with OAuth2 → Specific Mobile Endpoints
   ```
   - Optimized for mobile
   - Offline capability
   - Biometric authentication

4. **Third-Party Integration**
   ```
   External System → API with Authentication → Webhook Callbacks
   ```
   - Payment gateways
   - SMS providers (AfricasTalking)
   - Credit bureaus

### Authentication & Security:

1. **Basic Authentication**
   - Username/password
   - Default for Community App

2. **OAuth2** (Optional)
   - For third-party apps
   - Token-based access

3. **Two-Factor Authentication**
   - SMS or TOTP
   - Configurable per tenant

### Portal-Specific Features:

| Portal | Primary Users | Key Features | Authentication |
|--------|--------------|--------------|----------------|
| Community App | Bank Staff | Full CRUD operations, Reports | Basic Auth |
| Web App | Bank Staff | Modern UI, Better UX | Basic/OAuth2 |
| Self-Service | Customers | Limited read, basic transactions | Basic + 2FA |
| Mobile | Field Agents/Customers | Offline sync, GPS | OAuth2 + Biometric |
| API Direct | Systems | Full programmatic access | API Key/OAuth2 |

### Data Flow Example (Loan Application):

1. **Customer Portal**:
   ```
   Customer → Self-Service UI → POST /v1/self/loans → 
   Validation → Pending Status
   ```

2. **Staff Portal**:
   ```
   Staff → Community App → GET /v1/loans/pending →
   Review → POST /v1/loans/{id}/approve → 
   Disburse → Update GL
   ```

3. **Notifications**:
   ```
   Event → SMS Module → AfricasTalking Gateway → Customer Phone
   ```

### Adding New Portals:

To add a new portal (e.g., Agent Banking Portal):

1. Create new API endpoints under `/v1/agent/*`
2. Implement security context for agent roles
3. Build UI consuming these endpoints
4. Configure permissions in `m_permission` table
5. Add authentication strategy

## 🚀 Portal URLs by Environment

### Development Environment
```bash
# Core API
https://localhost:8443/fineract-provider/api/v1/

# Community App
http://localhost:9090/?baseApiUrl=https://localhost:8443/fineract-provider&tenantIdentifier=default

# Swagger UI
https://localhost:8443/fineract-provider/swagger-ui/index.html

# API Documentation
https://localhost:8443/fineract-provider/legacy-docs/apiLive.htm
```

### Docker Environment
```bash
# When using docker-compose
API: https://localhost:8443/fineract-provider/api/v1/
Community App: http://localhost:9090/
```

### Production Environment
```bash
# Replace with your domain
API: https://api.yourbank.com/fineract-provider/api/v1/
Community App: https://app.yourbank.com/
Self-Service: https://portal.yourbank.com/
```

## 📊 Portal Usage Statistics

### Typical Transaction Distribution:
- **Community App**: 60% (Staff operations)
- **Self-Service Portal**: 25% (Customer queries)
- **Mobile App**: 10% (Field operations)
- **API Direct**: 5% (Integrations)

### Performance Considerations:
- Community App: Full page loads, comprehensive features
- Self-Service: Lightweight, mobile-optimized
- API Direct: Highest throughput, no UI overhead
- Mobile: Offline-first, sync when connected

## 🔐 Security Model by Portal

### 1. **Internal Portals** (Staff)
- IP whitelisting
- Strong password policy
- Session timeout: 30 minutes
- Audit all actions

### 2. **External Portals** (Customers)
- Rate limiting
- 2FA mandatory
- Session timeout: 15 minutes
- Limited operation scope

### 3. **API Access** (Systems)
- API key rotation
- Request signing
- Webhook validation
- Rate limits per endpoint

## 🎯 Best Practices

### Portal Selection Guide:
1. **Use Community App when**:
   - Need full banking operations
   - Complex workflows required
   - Reporting and analytics needed

2. **Use Self-Service when**:
   - Customer-facing operations
   - Simple inquiries
   - Mobile access required

3. **Use Direct API when**:
   - Building custom interfaces
   - System integrations
   - Batch processing

### Development Tips:
1. Always test with multi-tenant setup
2. Use proper authentication per portal
3. Implement proper error handling
4. Cache frequently accessed data
5. Monitor API usage patterns

The multi-portal architecture allows different user types to access appropriate functionality while maintaining security and data integrity through the central API layer.