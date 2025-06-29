# Documented Open-API Handshake Integration

## Overview

The Documented Open-API Handshake integration establishes a secure, standards-based connection between Elise and ResMan using the ResMan Partner-API. This integration requires only four API scopes at launch (Leads, Units, Employees, Documents), with the flexibility to enable additional scopes (Payments, Maintenance) for future ResidentAI features. The handshake process ensures robust authentication, granular permissions, and a clear audit trail for all API interactions.

## Key Benefits

- **Secure Integration**: OAuth2-based authentication and encrypted API traffic
- **Granular Permissions**: Only required scopes enabled for least-privilege access
- **Scalable Architecture**: Easily extendable to new API scopes and endpoints
- **Auditability**: Complete logging of all API calls and data exchanges
- **Future-Proof**: Additional scopes can be enabled as new features are rolled out

## Integration Architecture

### API Handshake Flow

```
Elise Initiates API Handshake
         ↓
OAuth2 Authentication Request
         ↓
ResMan Authorization Server
         ↓
Access Token Issued (with Scopes)
         ↓
API Requests (Leads, Units, Employees, Documents)
         ↓
ResMan API Response
         ↓
Audit Logging & Monitoring
```

### Required API Scopes (Launch)
- **Leads**: Read/write access to lead data and guest cards
- **Units**: Read access to unit availability, pricing, and inventory
- **Employees**: Read/write access to employee records (for AI agent creation)
- **Documents**: Read access to lease packets, applications, and supporting docs

### Optional Future Scopes
- **Payments**: Read/write access to payment and fee data
- **Maintenance**: Read/write access to maintenance requests and logs

## Implementation Details

### 1. OAuth2 Authentication & Authorization

#### OAuth2 Flow
```json
{
  "oauth2": {
    "grant_type": "client_credentials",
    "client_id": "ELISE_CLIENT_ID",
    "client_secret": "ELISE_CLIENT_SECRET",
    "token_url": "https://api.resman.com/oauth2/token",
    "scopes": ["leads", "units", "employees", "documents"]
  }
}
```

#### Token Management
- **Access Token Lifetime**: 1 hour (recommended)
- **Refresh Logic**: Automatic token refresh before expiration
- **Scope Validation**: API calls fail if required scope is missing
- **Revocation Handling**: Immediate deactivation on credential compromise

### 2. API Endpoint Usage

#### Example: Leads API
```json
{
  "endpoint": "/api/leads",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer <access_token>",
    "Content-Type": "application/json"
  },
  "body": {
    "first_name": "Jane",
    "last_name": "Doe",
    "contact_method": "email",
    "source": "eliseai.com"
  }
}
```

#### Example: Units API
```json
{
  "endpoint": "/api/units",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer <access_token>"
  },
  "query_params": {
    "available": true,
    "property_id": "PROP_123"
  }
}
```

#### Example: Employees API
```json
{
  "endpoint": "/api/employees",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer <access_token>",
    "Content-Type": "application/json"
  },
  "body": {
    "first_name": "Meet",
    "last_name": "Elise",
    "role": "Leasing Agent",
    "email": "elise@property.com"
  }
}
```

#### Example: Documents API
```json
{
  "endpoint": "/api/documents",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer <access_token>"
  },
  "query_params": {
    "prospect_id": "PROSPECT_001",
    "document_type": "lease_packet"
  }
}
```

### 3. API Scope Expansion (Future)

#### Enabling Additional Scopes
- **Payments**: `/api/payments` (application fees, deposits, etc.)
- **Maintenance**: `/api/maintenance` (work orders, status updates)
- **Process**: Admin enables new scope in ResMan Partner Portal, Elise requests updated token with new scope

#### Example: Payments API
```json
{
  "endpoint": "/api/payments",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer <access_token>",
    "Content-Type": "application/json"
  },
  "body": {
    "prospect_id": "PROSPECT_001",
    "amount": 500.00,
    "fee_type": "application_fee"
  }
}
```

## Configuration Requirements

### ResMan Setup

#### Partner-API Registration
- **Register Application**: Create Elise integration in ResMan Partner Portal
- **Client Credentials**: Generate client_id and client_secret
- **Scope Selection**: Enable only required scopes (Leads, Units, Employees, Documents)
- **Callback URLs**: Register allowed callback/redirect URIs (if using auth code flow)
- **API Documentation**: Review endpoint and scope documentation

#### Security Settings
- **IP Whitelisting**: Restrict API access to known IPs (recommended)
- **Rate Limiting**: Set appropriate API call limits
- **Audit Logging**: Enable full API call logging for compliance

### Elise Configuration

#### API Client Setup
```json
{
  "api_client": {
    "client_id": "ELISE_CLIENT_ID",
    "client_secret": "ELISE_CLIENT_SECRET",
    "token_url": "https://api.resman.com/oauth2/token",
    "scopes": ["leads", "units", "employees", "documents"],
    "auto_refresh": true,
    "retry_on_failure": true
  }
}
```

#### Error Handling
- **401 Unauthorized**: Refresh token and retry
- **403 Forbidden**: Check scope permissions
- **429 Too Many Requests**: Backoff and retry
- **5xx Server Errors**: Retry with exponential backoff

## Error Handling & Monitoring

### Common Error Scenarios

#### Authentication Failures
- **Invalid Credentials**: Incorrect client_id or client_secret
- **Expired Token**: Token not refreshed in time
- **Scope Denied**: Requested scope not enabled in ResMan
- **Revoked Access**: Credentials revoked by admin

#### API Call Failures
- **Rate Limiting**: Too many requests in a short period
- **Endpoint Not Found**: Incorrect API path or version
- **Malformed Request**: Invalid request body or parameters
- **Permission Denied**: Insufficient scope for requested action

### Recovery Procedures
```json
{
  "error_recovery": {
    "auth_failure": {
      "action": "refresh_token",
      "fallback": "notify_admin",
      "notification": "system_alert"
    },
    "rate_limit": {
      "action": "backoff_and_retry",
      "fallback": "manual_intervention",
      "notification": "admin_alert"
    },
    "scope_error": {
      "action": "request_scope_update",
      "fallback": "log_and_skip",
      "notification": "integration_team"
    }
  }
}
```

### Monitoring & Alerts

#### Key Metrics
- API handshake success rate
- Token refresh success/failure
- API call error rates (4xx, 5xx)
- Scope usage and expansion events
- Latency and response times

#### Alert Thresholds
- **Handshake Success**: < 99% successful handshakes
- **Token Refresh**: < 99% refresh success
- **API Error Rate**: > 5% failed calls
- **Scope Expansion**: Any new scope enabled

## Security & Compliance

### Data Protection
- **Encryption**: All API traffic encrypted via HTTPS/TLS
- **Credential Storage**: Secure storage of client secrets (vault or env vars)
- **Access Control**: Least-privilege scope assignment
- **Audit Logging**: Full logging of all API calls and token events

### Privacy Considerations
- **PII Handling**: Only necessary personal data accessed
- **Consent Management**: Ensure user consent for data access
- **Data Minimization**: Limit data fields to required use cases
- **Right to Revoke**: Admin can revoke access at any time

## Testing & Validation

### Test Scenarios

#### Handshake & Auth Tests
1. **Valid Handshake**: Successful OAuth2 token exchange
2. **Scope Validation**: API calls fail if scope missing
3. **Token Refresh**: Automatic refresh before expiration
4. **Revocation Handling**: Access revoked and denied

#### API Usage Tests
1. **Leads API**: Create, update, and fetch leads
2. **Units API**: Fetch available units and pricing
3. **Employees API**: Create and update employee records
4. **Documents API**: Retrieve lease packets and supporting docs

#### Error Handling Tests
1. **Invalid Credentials**: Proper error and alert
2. **Rate Limiting**: Backoff and retry logic
3. **Scope Expansion**: Enable new scope and validate access

### Validation Criteria
- **Security**: 100% encrypted traffic and secure credential storage
- **Reliability**: 99.9% handshake and API uptime
- **Compliance**: 100% audit trail for all API events
- **Scalability**: Seamless addition of new scopes

## Troubleshooting Guide

### Common Issues

#### Handshake Failures
**Problem**: Unable to obtain access token
**Solution**:
1. Verify client_id and client_secret
2. Check token endpoint URL
3. Confirm required scopes are enabled
4. Review ResMan Partner Portal logs

#### API Call Errors
**Problem**: API calls failing with 4xx/5xx errors
**Solution**:
1. Check access token validity and scopes
2. Validate API endpoint and parameters
3. Review error response for details
4. Retry with exponential backoff

#### Scope Expansion Issues
**Problem**: Unable to access new API scope
**Solution**:
1. Enable scope in ResMan Partner Portal
2. Request new token with updated scopes
3. Validate permissions and retry

### Support Resources
- **API Documentation**: ResMan Partner-API reference
- **OAuth2 Guide**: Authentication and token management
- **Scope Management**: Enabling and updating API scopes
- **Contact Information**: Technical support channels

## Future Enhancements

### Planned Features
- **Automated Scope Discovery**: Dynamic detection of available scopes
- **Granular Audit Analytics**: Deep analysis of API usage and access
- **Multi-tenant Support**: Scalable integration for multiple properties
- **Advanced Security**: Hardware-backed credential storage

### Integration Expansions
- **Payments API**: Resident payment and fee management
- **Maintenance API**: Work order and maintenance tracking
- **Third-party Integrations**: Connect with other property management systems
- **Real-time Webhooks**: Push-based event notifications

## Conclusion

The Documented Open-API Handshake integration provides a secure, scalable, and auditable foundation for all data exchange between Elise and ResMan. By leveraging OAuth2 authentication, granular scope management, and robust error handling, this integration ensures that only the necessary data is accessed, with full transparency and control for property management teams.

This approach enables rapid feature expansion, future-proofs the integration, and maintains the highest standards of security and compliance for all API-driven workflows.

---

**Last Updated**: [Current Date]
**Version**: 1.0
**Status**: Ready for Implementation 