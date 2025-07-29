# Apache Fineract - Comprehensive API Documentation and SMS Gateway Implementation Guide

## Table of Contents
1. [System Architecture Overview](#system-architecture-overview)
2. [Core Modules and Functions](#core-modules-and-functions)
3. [API Endpoints by Module](#api-endpoints-by-module)
4. [SMS Gateway Architecture](#sms-gateway-architecture)
5. [AfricasTalking SMS Gateway Implementation Guide](#africastalking-sms-gateway-implementation-guide)

## System Architecture Overview

Apache Fineract is a modular microfinance platform with the following key architectural components:

### Core Architecture Layers

1. **API Layer** (`/v1/*`)
   - RESTful endpoints using JAX-RS annotations
   - Spring Security for authentication/authorization
   - Swagger/OpenAPI documentation

2. **Service Layer**
   - Business logic implementation
   - Transaction management
   - Event handling

3. **Domain Layer**
   - JPA entities with EclipseLink
   - Domain-driven design patterns
   - Command pattern for write operations

4. **Infrastructure Layer**
   - External service integrations
   - Batch job processing
   - Messaging and notifications

## Core Modules and Functions

### 1. **User Administration Module** (`fineract-provider`)
- **Purpose**: User management, roles, and permissions
- **Key Components**:
  - User management
  - Role-based access control
  - Password policies
  - Two-factor authentication

### 2. **Organization Module** (`fineract-provider`)
- **Purpose**: Organizational structure management
- **Key Components**:
  - Office hierarchy
  - Staff management
  - Holidays and working days
  - Provisioning rules

### 3. **Client Management Module** (`fineract-provider`)
- **Purpose**: Client and group management
- **Key Components**:
  - Client registration
  - Group/Center management
  - Documents and identifiers
  - Address management

### 4. **Product Management Modules**
- **Loan Module** (`fineract-loan`)
  - Loan products
  - Loan applications
  - Repayment processing
  - Interest calculations
  
- **Savings Module** (`fineract-savings`)
  - Savings products
  - Account management
  - Transactions
  - Interest posting

### 5. **Accounting Module** (`fineract-accounting`)
- **Purpose**: Financial accounting and GL
- **Key Components**:
  - Chart of accounts
  - Journal entries
  - Financial reporting

### 6. **Infrastructure Modules**
- **Commands** (`fineract-command`)
  - Command pattern implementation
  - Audit trail
  
- **Jobs**
  - Scheduled batch processing
  - COB (Close of Business)
  
- **Notifications**
  - SMS messaging
  - Email notifications
  - Push notifications

## API Endpoints by Module

### Authentication & Security
```
POST   /v1/authentication                    - User login
POST   /v1/authentication/logout             - User logout
GET    /v1/userdetails                      - Get user details
PUT    /v1/userdetails                      - Update user details
POST   /v1/twofactor                        - Two-factor authentication
PUT    /v1/twofactor/configure              - Configure 2FA
```

### User Administration
```
GET    /v1/users                            - List users
POST   /v1/users                            - Create user
GET    /v1/users/{userId}                   - Get user details
PUT    /v1/users/{userId}                   - Update user
DELETE /v1/users/{userId}                   - Delete user

GET    /v1/roles                            - List roles
POST   /v1/roles                            - Create role
GET    /v1/roles/{roleId}                   - Get role details
PUT    /v1/roles/{roleId}                   - Update role
DELETE /v1/roles/{roleId}                   - Delete role

GET    /v1/permissions                      - List permissions
PUT    /v1/permissions                      - Update permissions

GET    /v1/password_preferences             - Get password preferences
PUT    /v1/password_preferences             - Update password preferences
```

### Organization Management
```
GET    /v1/offices                          - List offices
POST   /v1/offices                          - Create office
GET    /v1/offices/{officeId}               - Get office details
PUT    /v1/offices/{officeId}               - Update office

GET    /v1/staff                            - List staff members
POST   /v1/staff                            - Create staff
GET    /v1/staff/{staffId}                  - Get staff details
PUT    /v1/staff/{staffId}                  - Update staff

GET    /v1/holidays                         - List holidays
POST   /v1/holidays                         - Create holiday
GET    /v1/holidays/{holidayId}             - Get holiday details
PUT    /v1/holidays/{holidayId}             - Update holiday

GET    /v1/workingdays                      - Get working days
PUT    /v1/workingdays                      - Update working days
```

### Client Management
```
GET    /v1/clients                          - List clients
POST   /v1/clients                          - Create client
GET    /v1/clients/{clientId}               - Get client details
PUT    /v1/clients/{clientId}               - Update client
DELETE /v1/clients/{clientId}               - Delete client

GET    /v1/clients/{clientId}/identifiers   - List client identifiers
POST   /v1/clients/{clientId}/identifiers   - Create identifier
DELETE /v1/clients/{clientId}/identifiers/{identifierId} - Delete identifier

GET    /v1/clients/{clientId}/charges       - List client charges
POST   /v1/clients/{clientId}/charges       - Create charge
GET    /v1/clients/{clientId}/charges/{chargeId} - Get charge details
PUT    /v1/clients/{clientId}/charges/{chargeId} - Update charge
DELETE /v1/clients/{clientId}/charges/{chargeId} - Delete charge

GET    /v1/groups                           - List groups
POST   /v1/groups                           - Create group
GET    /v1/groups/{groupId}                 - Get group details
PUT    /v1/groups/{groupId}                 - Update group

GET    /v1/centers                          - List centers
POST   /v1/centers                          - Create center
GET    /v1/centers/{centerId}               - Get center details
PUT    /v1/centers/{centerId}               - Update center
```

### Loan Management
```
GET    /v1/loanproducts                     - List loan products
POST   /v1/loanproducts                     - Create loan product
GET    /v1/loanproducts/{productId}         - Get product details
PUT    /v1/loanproducts/{productId}         - Update product

GET    /v1/loans                            - List loans
POST   /v1/loans                            - Create loan application
GET    /v1/loans/{loanId}                   - Get loan details
PUT    /v1/loans/{loanId}                   - Update loan
DELETE /v1/loans/{loanId}                   - Delete loan

POST   /v1/loans/{loanId}/approve           - Approve loan
POST   /v1/loans/{loanId}/disburse          - Disburse loan
POST   /v1/loans/{loanId}/repayment         - Make repayment
POST   /v1/loans/{loanId}/writeoff          - Write off loan
```

### Savings Management
```
GET    /v1/savingsproducts                  - List savings products
POST   /v1/savingsproducts                  - Create savings product
GET    /v1/savingsproducts/{productId}      - Get product details
PUT    /v1/savingsproducts/{productId}      - Update product

GET    /v1/savingsaccounts                  - List savings accounts
POST   /v1/savingsaccounts                  - Create savings account
GET    /v1/savingsaccounts/{accountId}      - Get account details
PUT    /v1/savingsaccounts/{accountId}      - Update account

POST   /v1/savingsaccounts/{accountId}/activate - Activate account
POST   /v1/savingsaccounts/{accountId}/deposit  - Make deposit
POST   /v1/savingsaccounts/{accountId}/withdrawal - Make withdrawal
```

### SMS & Notifications
```
GET    /v1/sms                              - List SMS messages
POST   /v1/sms                              - Send SMS
GET    /v1/sms/{smsId}                      - Get SMS details
PUT    /v1/sms/{smsId}                      - Update SMS
DELETE /v1/sms/{smsId}                      - Delete SMS

GET    /v1/smscampaigns                     - List SMS campaigns
POST   /v1/smscampaigns                     - Create campaign
GET    /v1/smscampaigns/{campaignId}        - Get campaign details
PUT    /v1/smscampaigns/{campaignId}        - Update campaign
DELETE /v1/smscampaigns/{campaignId}        - Delete campaign

GET    /v1/notifications                    - List notifications
PUT    /v1/notifications                    - Update notification status
```

### External Services Configuration
```
GET    /v1/externalservice/{servicename}    - Get service config
PUT    /v1/externalservice/{servicename}    - Update service config
```

### Jobs & Scheduling
```
GET    /v1/jobs                             - List jobs
GET    /v1/jobs/{jobId}                     - Get job details
PUT    /v1/jobs/{jobId}                     - Update job
POST   /v1/jobs/{jobId}/run                 - Run job manually

GET    /v1/scheduler                        - Get scheduler status
POST   /v1/scheduler/start                  - Start scheduler
POST   /v1/scheduler/stop                   - Stop scheduler
```

## SMS Gateway Architecture

### Current SMS Gateway Implementation

Fineract uses a generic SMS gateway architecture with the following components:

1. **SMS Message Entity** (`SmsMessage.java`)
   - Stores SMS messages in database
   - Tracks status (PENDING, SENT, DELIVERED, FAILED)
   - Links to clients, groups, staff, campaigns

2. **SMS Service Layer**
   - `SmsWritePlatformService` - Message creation/updates
   - `SmsReadPlatformService` - Message retrieval
   - `SmsMessageScheduledJobService` - Batch sending

3. **External Gateway Integration**
   - Generic HTTP-based gateway support
   - Configuration via External Services API
   - Message queue for batch processing

4. **Configuration Parameters**
   - `host_name` - Gateway host
   - `port_number` - Gateway port
   - `end_point` - API endpoint
   - `tenant_app_key` - Authentication key

### SMS Processing Flow

```
1. SMS Creation
   ├── API Request → SmsApiResource
   ├── Command Processing → CreateSmsCommand
   ├── Service Layer → SmsWritePlatformService
   └── Database → SmsMessage (status: PENDING)

2. SMS Sending (Batch Job)
   ├── SendMessageToSmsGatewayTasklet
   ├── Fetch PENDING messages
   ├── Build HTTP request to gateway
   ├── Send via RestTemplate
   └── Update status to WAITING_FOR_DELIVERY_REPORT

3. Delivery Report Processing
   ├── GetDeliveryReportsFromSmsGatewayTasklet
   ├── Fetch delivery reports from gateway
   └── Update message status (DELIVERED/FAILED)
```

## AfricasTalking SMS Gateway Implementation Guide

### Step 1: Understand AfricasTalking API

AfricasTalking SMS API requires:
- **Username**: Your AfricasTalking username
- **API Key**: Authentication key from your account
- **Endpoint**: `https://api.africastalking.com/version1/messaging`
- **From**: Optional sender ID

### Step 2: Create AfricasTalking Gateway Provider

Create new file: `/fineract-provider/src/main/java/org/apache/fineract/infrastructure/sms/gateway/AfricasTalkingGatewayProvider.java`

```java
package org.apache.fineract.infrastructure.sms.gateway;

import java.net.URI;
import java.util.HashMap;
import java.util.Map;
import org.apache.fineract.infrastructure.campaigns.sms.data.MessageGatewayConfigurationData;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;

@Component
public class AfricasTalkingGatewayProvider implements SmsGatewayProvider {

    private static final String AFRICASTALKING_URL = "https://api.africastalking.com/version1/messaging";
    
    @Override
    public String getName() {
        return "AFRICASTALKING";
    }
    
    @Override
    public Map<String, Object> buildRequest(MessageGatewayConfigurationData config, 
                                          String recipient, 
                                          String message) {
        Map<String, Object> requestDetails = new HashMap<>();
        
        // Build headers
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.add("apiKey", config.getValue("api_key"));
        headers.add("Accept", "application/json");
        
        // Build request body
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("username", config.getValue("username"));
        params.add("to", recipient);
        params.add("message", message);
        
        // Optional sender ID
        String from = config.getValue("from");
        if (from != null && !from.isEmpty()) {
            params.add("from", from);
        }
        
        HttpEntity<MultiValueMap<String, String>> entity = new HttpEntity<>(params, headers);
        
        requestDetails.put("uri", URI.create(AFRICASTALKING_URL));
        requestDetails.put("entity", entity);
        
        return requestDetails;
    }
    
    @Override
    public SmsDeliveryStatus parseResponse(String response) {
        // Parse AfricasTalking JSON response
        // Return appropriate status based on response
        return SmsDeliveryStatus.SENT;
    }
}
```

### Step 3: Create Gateway Provider Interface

Create new file: `/fineract-provider/src/main/java/org/apache/fineract/infrastructure/sms/gateway/SmsGatewayProvider.java`

```java
package org.apache.fineract.infrastructure.sms.gateway;

import java.util.Map;
import org.apache.fineract.infrastructure.campaigns.sms.data.MessageGatewayConfigurationData;

public interface SmsGatewayProvider {
    String getName();
    Map<String, Object> buildRequest(MessageGatewayConfigurationData config, 
                                   String recipient, 
                                   String message);
    SmsDeliveryStatus parseResponse(String response);
}

enum SmsDeliveryStatus {
    SENT, DELIVERED, FAILED
}
```

### Step 4: Create Gateway Factory

Create new file: `/fineract-provider/src/main/java/org/apache/fineract/infrastructure/sms/gateway/SmsGatewayFactory.java`

```java
package org.apache.fineract.infrastructure.sms.gateway;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class SmsGatewayFactory {
    
    @Autowired
    private List<SmsGatewayProvider> providers;
    
    private Map<String, SmsGatewayProvider> providerMap = new HashMap<>();
    
    @PostConstruct
    public void init() {
        for (SmsGatewayProvider provider : providers) {
            providerMap.put(provider.getName(), provider);
        }
    }
    
    public SmsGatewayProvider getProvider(String providerName) {
        SmsGatewayProvider provider = providerMap.get(providerName);
        if (provider == null) {
            // Default to generic HTTP provider
            provider = providerMap.get("GENERIC");
        }
        return provider;
    }
}
```

### Step 5: Modify SMS Config Utils

Update `/fineract-provider/src/main/java/org/apache/fineract/infrastructure/campaigns/helper/SmsConfigUtils.java`:

```java
@Component
public class SmsConfigUtils {

    @Autowired
    private ExternalServicesPropertiesReadPlatformService propertiesReadPlatformService;
    
    @Autowired
    private SmsGatewayFactory gatewayFactory;

    public Map<String, Object> getMessageGateWayRequestURI(final String apiEndPoint, 
                                                          String apiQueueResourceDatas) {
        MessageGatewayConfigurationData config = this.propertiesReadPlatformService.getSMSGateway();
        
        // Get provider name from config
        String providerName = config.getValue("provider_name");
        if (providerName == null) {
            providerName = "GENERIC";
        }
        
        // Get appropriate gateway provider
        SmsGatewayProvider provider = gatewayFactory.getProvider(providerName);
        
        // Build request using provider
        return provider.buildRequest(config, extractRecipient(apiQueueResourceDatas), 
                                   extractMessage(apiQueueResourceDatas));
    }
}
```

### Step 6: Add Database Migration

Create new file: `/fineract-provider/src/main/resources/db/changelog/tenant/changelog-africastalking.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet author="fineract" id="add-africastalking-sms-config">
        <insert tableName="c_external_service">
            <column name="name" value="AFRICASTALKING_SMS"/>
        </insert>
        
        <sql>
            SET @service_id = LAST_INSERT_ID();
            
            INSERT INTO c_external_service_properties 
            (external_service_id, name, value) VALUES
            (@service_id, 'username', ''),
            (@service_id, 'api_key', ''),
            (@service_id, 'from', ''),
            (@service_id, 'provider_name', 'AFRICASTALKING');
        </sql>
    </changeSet>
</databaseChangeLog>
```

### Step 7: Configure AfricasTalking in Fineract

After deployment, configure AfricasTalking via API:

```bash
# Update AfricasTalking configuration
curl -X PUT https://localhost:8443/fineract-provider/api/v1/externalservice/MESSAGE_GATEWAY \
  -H "Authorization: Basic bWlmb3M6cGFzc3dvcmQ=" \
  -H "Content-Type: application/json" \
  -H "Fineract-Platform-TenantId: default" \
  -d '{
    "username": "YOUR_AFRICASTALKING_USERNAME",
    "api_key": "YOUR_AFRICASTALKING_API_KEY",
    "from": "YOUR_SENDER_ID",
    "provider_name": "AFRICASTALKING"
  }'
```

### Step 8: Test SMS Sending

```bash
# Send test SMS
curl -X POST https://localhost:8443/fineract-provider/api/v1/sms \
  -H "Authorization: Basic bWlmb3M6cGFzc3dvcmQ=" \
  -H "Content-Type: application/json" \
  -H "Fineract-Platform-TenantId: default" \
  -d '{
    "mobileNo": "+254712345678",
    "message": "Test message from Fineract via AfricasTalking"
  }'
```

### Step 9: Additional Considerations

1. **Error Handling**
   - Implement retry logic for failed messages
   - Log API responses for debugging
   - Handle rate limiting

2. **Bulk SMS**
   - AfricasTalking supports bulk SMS
   - Modify batch processing to send multiple recipients

3. **Delivery Reports**
   - Implement webhook endpoint for delivery reports
   - Update message status based on callbacks

4. **Cost Tracking**
   - Store SMS costs from API response
   - Add reporting for SMS expenses

5. **Testing**
   - Create unit tests for gateway provider
   - Integration tests with mock server
   - Load testing for batch operations

### Step 10: Deployment Checklist

- [ ] Add AfricasTalking dependency if using their Java SDK
- [ ] Run database migrations
- [ ] Configure external service properties
- [ ] Test SMS sending functionality
- [ ] Monitor logs for errors
- [ ] Set up delivery report webhook
- [ ] Document configuration for operations team

## Module-Specific Subagents

### 1. User Administration Agent
**Purpose**: Handle user, role, and permission management
**Key Operations**:
- User CRUD operations
- Role assignment
- Permission management
- Password policy enforcement

### 2. Client Management Agent
**Purpose**: Manage clients, groups, and centers
**Key Operations**:
- Client onboarding
- KYC documentation
- Group formation
- Meeting scheduling

### 3. Loan Processing Agent
**Purpose**: Handle loan lifecycle
**Key Operations**:
- Product configuration
- Application processing
- Disbursement
- Repayment tracking
- Delinquency management

### 4. Savings Account Agent
**Purpose**: Manage savings products and accounts
**Key Operations**:
- Account opening
- Transaction processing
- Interest calculation
- Maturity handling

### 5. Accounting Agent
**Purpose**: Financial accounting and reporting
**Key Operations**:
- Journal entry creation
- GL account management
- Financial report generation
- Audit trail maintenance

### 6. Notification Agent
**Purpose**: Handle all communication channels
**Key Operations**:
- SMS gateway integration
- Email notifications
- Push notifications
- Campaign management

### 7. Batch Processing Agent
**Purpose**: Manage scheduled jobs
**Key Operations**:
- COB processing
- Interest posting
- Report generation
- Data migration

### 8. Integration Agent
**Purpose**: External system integration
**Key Operations**:
- Payment gateway integration
- Credit bureau connectivity
- SMS gateway management
- Third-party API handling

## Best Practices

1. **API Design**
   - Follow RESTful conventions
   - Use proper HTTP status codes
   - Implement pagination for lists
   - Version APIs appropriately

2. **Security**
   - Always validate inputs
   - Use parameterized queries
   - Implement proper authentication
   - Audit all operations

3. **Performance**
   - Use database indexes
   - Implement caching
   - Optimize queries
   - Monitor resource usage

4. **Error Handling**
   - Log all errors with context
   - Return meaningful error messages
   - Implement retry mechanisms
   - Handle edge cases

5. **Testing**
   - Unit test all services
   - Integration test APIs
   - Load test critical paths
   - Document test scenarios