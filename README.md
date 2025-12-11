<p align="center">
  <img src="https://avatars.githubusercontent.com/u/26124358?s=200&v=4" alt="Mbanq Logo" width="120"/>
</p>

# Core SDK JS

[![NPM](https://nodei.co/npm/@mbanq/core-sdk-js.png)](https://npmjs.org/package/@mbanq/core-sdk-js)

![npm version](https://img.shields.io/npm/v/@mbanq/core-sdk-js.svg)
[![downloads](https://img.shields.io/npm/dt/@mbanq/core-sdk-js.svg)](https://npmjs.org/package/@mbanq/core-sdk-js)
![license](https://img.shields.io/github/license/Mbanq/core-sdk-js)

## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Setup](#setup)
  - [Axios Instance Logger](#axios-instance-logger)
- [Middleware](#middleware)
  - [Logging Middleware](#logging-middleware)
  - [Metrics Middleware](#metrics-middleware)
  - [Custom Middleware](#custom-middleware)
- [API Reference](#api-reference)
  - [Transfer Operations](#transfer-operations)
  - [Client Identifier Operations](#client-identifier-operations)
  - [User Operations](#user-operations)
- [Documentation](#documentation)
- [Type Safety & Validation](#type-safety--validation)
- [Error Handling](#error-handling)
- [Examples](#examples)

## Introduction
This library provides a comprehensive JavaScript SDK for interacting with the Mbanq payment API. It offers type-safe payment operations with built-in validation, multi-tenant support, and a modern fluent API design.
## Installation

```bash
npm install @mbanq/core-sdk-js
```

## Quick Start

```javascript
import { createInstance, CreatePayment, GetTransfers } from '@mbanq/core-sdk-js';

const client = createInstance({
  secret: 'your-jwt-secret',
  signee: 'YOUR-SIGNEE',
  baseUrl: 'https://api.cloud.mbanq.com',
  tenantId: 'your-tenant-id'
});

// Create payment
const payment = await client.request(CreatePayment({
  amount: 100.00,
  currency: 'USD',
  paymentRail: 'ACH',
  paymentType: 'CREDIT',
  originator: { accountId: '123456789' },
  recipient: {
    name: 'Jane Smith',
    accountNumber: '987654321',
    accountType: 'SAVINGS',
    recipientType: 'INDIVIDUAL',
    address: {
      line1: '789 Oak Ave',
      city: 'Another Town',
      stateCode: 'CA',
      countryCode: 'US',
      postalCode: '54321'
    },
    bankInformation: { routingNumber: '321070007' }
  }
}));

// Get transfers
const transfers = await client.request(GetTransfers({
  transferStatus: 'EXECUTION_SCHEDULED',
  executedAt: '2025-01-22',
  paymentType: 'ACH',
  queryLimit: 200
}));
```

## Setup

### Authentication Options

The SDK supports multiple authentication methods. Choose the one that fits your integration:

#### 1. JWT Token Authentication (Recommended)
Use your API secret and signee for JWT-based authentication:

```javascript
const client = createInstance({
  secret: 'your-jwt-secret',
  signee: 'YOUR-SIGNEE', 
  baseUrl: 'https://api.cloud.mbanq.com',
  tenantId: 'your-tenant-id'
});
```

#### 2. Bearer Token Authentication
If you already have a valid access token:

```javascript
// With "Bearer " prefix (recommended)
const client = createInstance({
  bearerToken: 'Bearer your-access-token',
  baseUrl: 'https://api.cloud.mbanq.com', 
  tenantId: 'your-tenant-id'
});

// Without "Bearer " prefix (automatically added)
const client = createInstance({
  bearerToken: 'your-access-token', // "Bearer " will be added automatically
  baseUrl: 'https://api.cloud.mbanq.com', 
  tenantId: 'your-tenant-id'
});
```

#### 3. OAuth Credential Authentication
For OAuth 2.0 password grant flow:

```javascript
const client = createInstance({
  credential: {
    client_id: 'your-client-id',
    client_secret: 'your-client-secret',
    username: 'your-username',
    password: 'your-password',
    grant_type: 'password'
  },
  baseUrl: 'https://api.cloud.mbanq.com',
  tenantId: 'your-tenant-id'
});
```

#### Authentication Priority
When multiple authentication methods are provided, the SDK uses them in this order:
1. **`bearerToken`** - Takes highest priority if provided
2. **`credential`** - OAuth flow is used if no bearerToken 
3. **`secret` + `signee`** - JWT authentication used as fallback

#### Additional Configuration Options
```javascript
const client = createInstance({
  // Choose one authentication method from above
  secret: 'your-jwt-secret',
  signee: 'YOUR-SIGNEE',
  
  // Required configuration
  baseUrl: 'https://api.cloud.mbanq.com',
  tenantId: 'your-tenant-id',
  
  // Optional configuration
  traceId: 'custom-trace-id', // Custom request tracing identifier
  axiosConfig: {
    timeout: 30000, // Request timeout in milliseconds (default: 29000)
    keepAlive: true, // HTTP keep-alive for connection reuse
    headers: {
      'Custom-Header': 'custom-value' // Additional HTTP headers
    }
  }
});
```

### Security Best Practices

#### Credential Management
- **Never hardcode credentials** in your source code
- Use environment variables or secure credential management systems
- Rotate API secrets and tokens regularly
- Use the minimum required permissions for your integration

#### Environment Variables Example
```javascript
const client = createInstance({
  secret: process.env.MBANQ_API_SECRET,
  signee: process.env.MBANQ_API_SIGNEE,
  baseUrl: process.env.MBANQ_API_URL,
  tenantId: process.env.MBANQ_TENANT_ID
});
```

#### Production Considerations
- Use HTTPS endpoints only (`https://`)
- Implement proper error handling to avoid credential leakage in logs
- Configure appropriate request timeouts
- Use connection pooling for high-volume applications

### Axios Instance Logger
You can also configure an Axios instance logger to set up interceptors or other axios-specific configurations:

```javascript
const axiosLogger = (axiosInstance) => {
  // Add request interceptor
  axiosInstance.interceptors.request.use(
    (config) => {
      console.log('Request:', config.method?.toUpperCase(), config.url);
      return config;
    }
  );
  
  // Add response interceptor
  axiosInstance.interceptors.response.use(
    (response) => {
      console.log('Response:', response.status, response.config.url);
      return response;
    }
  );
};

const coreSDK = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  logger: axiosLogger // Configure Axios instance
});
```

## Middleware
The SDK supports middleware for cross-cutting concerns like logging and metrics. Middleware functions are executed automatically around command execution.

**Note**: This is different from the Axios instance logger above. Middleware loggers handle command-level logging, while the Axios logger handles HTTP request/response logging.

### Understanding Middleware Concepts

There are three key concepts to understand:

1. **Middleware** - A wrapper that runs code before/after command execution
2. **Logger Interface** - A contract defining what methods a logger object must have
3. **Custom Middleware** - Your own middleware implementing the Middleware interface

### Quick Comparison

| Concept | What It Is | When to Use |
|---------|-----------|-------------|
| **Logging Middleware** | Pre-built middleware for logging commands | Use `createLoggingMiddleware(logger)` when you want automatic command logging |
| **Logger Interface** | Contract for logger objects | Implement this when creating a custom logger for Logging Middleware |
| **Metrics Middleware** | Pre-built middleware for tracking metrics | Use `createMetricsMiddleware(client)` when you want to track command metrics |
| **Custom Middleware** | Build-your-own middleware | Implement the Middleware interface when you need custom behavior |

---

### 1. Logging Middleware

**Purpose**: Automatically logs command execution details (inputs, outputs, errors).

**Basic Usage** (with console):
```javascript
import { createInstance, createLoggingMiddleware } from '@mbanq/core-sdk-js';

const client = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  middlewares: [createLoggingMiddleware(console)]
});
```

**With Custom Logger**:

First, create a logger that implements the Logger interface:

```typescript
// Logger Interface - what your logger must implement
interface Logger {
  info: (message: string, ...args: unknown[]) => void;   // Required
  error: (message: string, ...args: unknown[]) => void;  // Required
  warn?: (message: string, ...args: unknown[]) => void;  // Optional
  log?: (message: string, ...args: unknown[]) => void;   // Optional
}

// Example: Custom logger implementation
const customLogger = {
  info: (message, ...args) => myLoggingService.info(message, args),
  error: (message, ...args) => myLoggingService.error(message, args),
  warn: (message, ...args) => myLoggingService.warn(message, args)
};

// Use your custom logger with Logging Middleware
const middleware = createLoggingMiddleware(customLogger);

const client = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  middlewares: [middleware]
});
```

---

### 2. Metrics Middleware

**Purpose**: Tracks command execution metrics (counters for started, completed, and error events).

**Usage**:
```javascript
import { createInstance, createMetricsMiddleware } from '@mbanq/core-sdk-js';

// Your metrics client must implement the MetricsClient interface
const metricsClient = {
  incrementCounter: (counterName) => {
    // Increment your counter (e.g., Prometheus, StatsD, etc.)
    console.log(`Counter: ${counterName}`);
  },
  recordError: (error) => {
    // Optional: Record error details
    console.error('Command error:', error);
  }
};

const client = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  middlewares: [createMetricsMiddleware(metricsClient)]
});
```

**MetricsClient Interface**:
```typescript
interface MetricsClient {
  incrementCounter: (counterName: string) => void;  // Required
  recordError?: (error: Error) => void;             // Optional
}
```

---

### 3. Custom Middleware

**Purpose**: Create your own middleware for custom behavior (e.g., performance monitoring, audit logging, caching).

**Middleware Interface**:
```typescript
interface Middleware {
  before?: (command: Command) => Promise<void>;              // Called before execution
  after?: (command: Command, response: any) => Promise<void>; // Called after success
  onError?: (command: Command, error: Error) => Promise<void>; // Called on error
}
```

**Example - Performance Monitoring**:
```javascript
const performanceMiddleware = {
  before: async (command) => {
    command._startTime = Date.now();
    console.log(`[${command.metadata.commandName}] Starting...`);
  },
  after: async (command, response) => {
    const duration = Date.now() - command._startTime;
    console.log(`[${command.metadata.commandName}] Completed in ${duration}ms`);
  },
  onError: async (command, error) => {
    const duration = Date.now() - command._startTime;
    console.error(`[${command.metadata.commandName}] Failed after ${duration}ms:`, error.message);
  }
};

const client = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  middlewares: [performanceMiddleware]
});
```

---

### Using Multiple Middleware

You can combine multiple middleware together. They execute in the order provided:

```javascript
const client = createInstance({
  secret: 'testing123',
  signee: 'TESTING',
  baseUrl: 'https://example.com',
  tenantId: 'testing',
  middlewares: [
    createLoggingMiddleware(console),      // Logs commands
    createMetricsMiddleware(metricsClient), // Tracks metrics
    performanceMiddleware                   // Custom performance tracking
  ]
});
```

**Execution Order**:
1. All `before` hooks run in order
2. Command executes
3. All `after` hooks run in reverse order (or `onError` if command fails)

## API Reference

The SDK uses a Command Pattern for all operations. Commands are created using factory functions and executed via the client's `request()` method. Each command function includes detailed JSDoc documentation describing its parameters, return types, and usage.

### Available Commands

#### Account Operations

| Command | Description |
|---------|-------------|
| `GetAccount` | Retrieve detailed information about a specific savings account |
| `UpdateAccount` | Update account settings and configurations |
| `DeleteAccount` | Delete a savings account |
| `GetAccountsOfClient` | Get all accounts associated with a specific client |
| `CreateAndActivateAccount` | Create and immediately activate a new savings account |
| `ActivateAccount` | Activate an existing account |
| `ApproveAccount` | Approve a pending account |
| `RejectAccount` | Reject a pending account |
| `UndoApprovalAccount` | Undo approval of an account |
| `CloseAccount` | Close an existing account |
| `PostInterestAsOn` | Post interest to an account as of a specific date |
| `CalculateInterestAsOn` | Calculate interest for an account as of a specific date |
| `WithdrawByClientId` | Process a withdrawal for a client's account |
| `DepositByClientId` | Process a deposit to a client's account |

#### Account Product Operations

| Command | Description |
|---------|-------------|
| `CreateAccountProduct` | Create a new account product configuration |
| `UpdateAccountProduct` | Update an existing account product |
| `GetAllAccountProducts` | Retrieve all available account products |
| `GetAccountProductById` | Get details of a specific account product |

#### Account Statement Operations

| Command | Description |
|---------|-------------|
| `GenerateAccountStatement` | Generate a statement for an account |
| `DownloadAccountStatement` | Download a generated account statement |
| `GetAccountDocumentsDetails` | Get details of account documents |

#### Card Operations

| Command | Description |
|---------|-------------|
| `SendAuthorizationToCore` | Send card authorization request to core system |
| `UpdateCardID` | Update card identification information |

#### Card Product Operations

| Command | Description |
|---------|-------------|
| `ListCardProduct` | List all available card products with pagination |
| `GetCardProduct` | Get details of a specific card product by ID |
| `CreateCardProduct` | Create a new card product configuration |
| `UpdateCardProduct` | Update an existing card product |

#### Client Operations

| Command | Description |
|---------|-------------|
| `GetClient` | Retrieve detailed client information with optional related data |
| `UpdateClient` | Update client information |
| `CreateClient` | Create a new client record |
| `GetClients` | List clients with filtering and pagination |
| `DeleteClient` | Delete a client record |
| `VerifyWithActivateClients` | Verify and optionally activate a client |
| `GetStatusOfVerifyClient` | Get verification status of a client |
| `CloseClient` | Close a client account |

#### Client Address Operations

| Command | Description |
|---------|-------------|
| `GetClientAddress` | Retrieve client address information |
| `CreateClientAddress` | Add a new address for a client |
| `UpdateClientAddress` | Update existing client address |
| `SetClientAddressStatus` | Set the status of a client address |

#### Client Classification Operations

| Command | Description |
|---------|-------------|
| `GetClientClassification` | Get current classification of a client |
| `SwitchClientClassification` | Switch client to a different classification |
| `CancelSwitchClientClassification` | Cancel a pending classification switch |

#### Client Identifier Operations

| Command | Description |
|---------|-------------|
| `ListClientDocument` | List all identity documents for a client |
| `GetPermittedDocumentTypes` | Get allowed document types for a client |
| `CreateClientIdentifier` | Create a new identity document for a client |
| `UpdateClientIdentifier` | Update an existing identity document |
| `DeleteClientDocument` | Delete a client identity document |
| `ApproveRejectClientDocument` | Approve or reject a client document |
| `UploadClientIdentifierDocument` | Upload a document file for client identifier |

#### Custom Operations

| Command | Description |
|---------|-------------|
| `CustomUpdate` | Execute a custom update operation |
| `CustomCreate` | Execute a custom create operation |
| `CustomGet` | Execute a custom get operation |

#### Global Configuration Operations

| Command | Description |
|---------|-------------|
| `GetConfigurations` | Retrieve all system configurations |
| `GetConfigurationByName` | Get a specific configuration by name |
| `EnableDisableConfiguration` | Enable or disable a configuration |
| `UpdateConfiguration` | Update a configuration value |

#### Payment Operations

| Command | Description |
|---------|-------------|
| `CreatePayment` | Create a new payment |
| `GetPayment` | Retrieve payment details by ID |
| `UpdatePayment` | Update an existing payment |
| `DeletePayment` | Delete a payment |
| `GetPayments` | List payments with filtering options |

#### Recipient Operations

| Command | Description |
|---------|-------------|
| `GetRecipient` | Get details of a specific recipient |
| `CreateRecipient` | Create a new payment recipient |
| `DeleteRecipient` | Delete a recipient |
| `GetRecipients` | List all recipients for a client |

#### Transaction Operations

| Command | Description |
|---------|-------------|
| `GetPendingTransactions` | Query pending transactions with filters |
| `GetCompletedTransactions` | Query completed transactions with filters |
| `GetRecentTransactions` | Retrieve recent transactions including status (Completed, Pending, Rejected) and card information |
| `GetTransactionById` | Get details of a specific transaction by ID with optional enriched data |
| `GetBankDetailsFromRoutingCode` | Get bank details from routing code for ACH or WIRE transfers |

#### Transfer Operations

| Command | Description |
|---------|-------------|
| `GetTransfers` | List transfers with filtering options |
| `CreateTransfer` | Create a new money transfer |
| `GetTransfer` | Retrieve transfer details by ID |
| `MarkAsSuccess` | Mark a transfer as successfully completed |
| `MarkAsProcessing` | Mark a transfer as currently processing |
| `MarkAsReturned` | Mark a transfer as returned with reason |
| `LogFailTransfer` | Log a failed transfer with error details |
| `MarkAsFail` | Mark a transfer as failed |
| `UpdateTraceNumber` | Update the trace number for a transfer |

#### User Operations

| Command | Description |
|---------|-------------|
| `GetUserDetail` | Get current user details |
| `EnableSelfServiceAccess` | Enable self-service access for a user |
| `UpdateSelfServiceUser` | Update self-service user information |
| `DeleteSelfServiceUser` | Delete a self-service user |

## Documentation

For detailed information about specific features and APIs, refer to the following resources:

### API Documentation
- **[Mbanq API Reference](https://apidocs.cloud.mbanq.com/reference)** - Official API documentation with detailed endpoint information, request/response schemas, and examples

### Quick Links
- [Authentication Options](#authentication-options) - JWT, Bearer Token, and OAuth configuration
- [Middleware Configuration](#middleware) - Logging, metrics, and custom middleware
- [Error Handling Patterns](#error-handling) - Error types and handling strategies
- [Type Definitions](#type-safety--validation) - Complete schema and type reference

## Type Safety & Validation

The SDK uses [Zod](https://zod.dev/) for runtime type validation and TypeScript for compile-time type safety.

### Supported Payment Rails
- `ACH` - Automated Clearing House
- `SAMEDAYACH` - Same Day ACH
- `WIRE` - Domestic Wire Transfer
- `SWIFT` - International Wire Transfer  
- `INTERNAL` - Internal Transfer
- `FXPAY` - Foreign Exchange Payment
- `CARD` - Card Payment

### Payment Statuses
- `DRAFT`, `AML_SCREENING`, `AML_REJECTED`
- `EXECUTION_SCHEDULED`, `EXECUTION_PROCESSING`, `EXECUTION_SUCCESS`, `EXECUTION_FAILURE`
- `RETURNED`, `CANCELLED`, `COMPLIANCE_FAILURE`, `DELETED`, `UNKNOWN`

### Validation Features
- **Input Validation**: All create/update operations validate data structure
- **Response Validation**: API responses are validated before returning
- **Custom Rules**: WIRE transfers require recipient address with state/country
- **Type Safety**: Full TypeScript support with inferred types

## Error Handling
The library uses a consistent error handling pattern. All API calls may throw the following errors:

### Error Types
- **`CommandError`**: Base error type with `code`, `message`, `statusCode`, and optional `requestId`
- **Authentication Errors**: Invalid or missing API credentials
  - Invalid JWT secret/signee combination
  - Expired or invalid bearer token
  - OAuth credential authentication failure
- **Validation Errors**: Invalid parameters provided (uses Zod validation)
- **API Errors**: Server-side errors with specific error codes
- **Network Errors**: Network connectivity or timeout issues

### Common Authentication Error Scenarios
- **Missing credentials**: No authentication method provided
- **Invalid JWT**: Incorrect secret or signee values
- **Expired token**: Bearer token has expired and needs refresh
- **OAuth failure**: Invalid username/password or client credentials

## Examples

### Complete Transfer Flow Example

```javascript
import { createInstance, CreateTransfer, GetTransfer, MarkAsSuccess } from '@mbanq/core-sdk-js';

// Initialize the client
const client = createInstance({ 
  secret: 'your-secret', 
  signee: 'YOUR-SIGNEE', 
  baseUrl: 'https://api.cloud.mbanq.com', 
  tenantId: 'your-tenant-id' 
});

// Create an ACH transfer
const achTransfer = await client.request(CreateTransfer({
  amount: 1500,
  currency: 'USD',
  paymentRail: 'ACH',
  paymentType: 'CREDIT',
  debtor: {
    name: 'Alice Corporation',
    identifier: '111222333',
    accountType: 'CHECKING',
    agent: {
      name: 'First National Bank',
      identifier: '021000021'
    }
  },
  creditor: {
    name: 'Bob Enterprises',
    identifier: '444555666',
    accountType: 'CHECKING',
    agent: {
      name: 'Second Federal Bank',
      identifier: '121000248'
    }
  },
  reference: ['Invoice-2025-001']
}));

console.log('Created transfer:', achTransfer.id);

// Get transfer details
const transfer = await client.request(GetTransfer(achTransfer.id));
console.log('Transfer status:', transfer.status);

// Mark transfer as successful
if (transfer.status === 'EXECUTION_PROCESSING') {
  await client.request(MarkAsSuccess(transfer.externalId, 'ACH'));
  console.log('Transfer marked as successful');
}
```

### Error Handling Example

```javascript
import { isCommandError, CreateTransfer } from '@mbanq/core-sdk-js';

try {
  const transfer = await client.request(CreateTransfer({
    amount: -100, // Invalid: negative amount
    currency: 'INVALID', // Invalid: not 3-character code
    // Missing required fields
  }));
} catch (error) {
  if (isCommandError(error)) {
    console.error('Transfer creation failed:');
    console.error('Code:', error.code);
    console.error('Message:', error.message);
    if (error.requestId) {
      console.error('Request ID:', error.requestId);
    }
  } else {
    console.error('Unexpected error:', error);
  }
}
```

For more detailed information or support, please contact our support team or visit our developer portal.
