# Execution Flow Documentation

This document provides a detailed explanation of the execution flow for the CMP RA Component and CMP Client Component.

## Table of Contents

- [Overview](#overview)
- [CMP RA Component Execution Flow](#cmp-ra-component-execution-flow)
- [CMP Client Component Execution Flow](#cmp-client-component-execution-flow)
- [Message Processing Pipeline](#message-processing-pipeline)
- [Key Components and Their Roles](#key-components-and-their-roles)
- [Sequence Flows](#sequence-flows)

---

## Overview

The CMP RA Component library provides two main execution contexts:

1. **Registration Authority (RA) Mode** - Acts as an intermediary between clients (End Entities) and Certificate Authorities
2. **Client Mode** - Acts as an end entity requesting certificates or performing PKI operations

Both modes follow the Certificate Management Protocol (CMP) as defined in RFC 9483 and RFC 9810.

---

## CMP RA Component Execution Flow

### 1. Component Instantiation

The RA component can be instantiated in two modes:

#### Mode A: Full CMP RA Component

**Entry Point:** `CmpRaComponent.instantiateCmpRaComponent()`

**Location:** `src/main/java/com/siemens/pki/cmpracomponent/main/CmpRaComponent.java:99`

```java
CmpRaInterface ra = CmpRaComponent.instantiateCmpRaComponent(
    configuration,      // RA configuration
    upstreamExchange   // Interface to upstream CA
);
```

**Initialization Flow:**

1. **Configuration Loading** - The embedding application provides:
   - `Configuration` object containing:
     - Verification context (trusted certificates, CRL/OCSP settings)
     - Credential context (private keys, certificates)
     - Inventory interface (for authorization and logging)
     - Persistency interface (for transaction state management)
   - `UpstreamExchange` interface implementation for CA communication

2. **Internal Component Creation** (`CmpRaImplementation.java:74`):
   - Creates `PersistencyContextManager` for transaction state tracking
   - Wraps `UpstreamExchange` with logging and error handling
   - Instantiates `CmpRaUpstream` for upstream (CA) communication
   - Instantiates `RaDownstream` for downstream (client/EE) communication

#### Mode B: P10/X.509 RA Component

**Entry Point:** `CmpRaComponent.instantiateP10X509CmpRaComponent()`

**Location:** `src/main/java/com/siemens/pki/cmpracomponent/main/CmpRaComponent.java:144`

```java
Function<byte[], byte[]> ra = CmpRaComponent.instantiateP10X509CmpRaComponent(
    configuration,           // RA configuration
    upstreamP10X509Exchange // Function for PKCS#10/X.509 exchange
);
```

This mode handles legacy PKI systems using PKCS#10 CSRs and X.509 certificates instead of full CMP.

### 2. Request Processing Flow

#### Downstream Request (from Client/EE)

**Entry Point:** `CmpRaImplementation.processRequest()`

**Location:** `src/main/java/com/siemens/pki/cmpracomponent/msgprocessing/CmpRaImplementation.java:122`

**Execution Steps:**

```
1. Receive byte[] request
   └─> Parse to PKIMessage
       └─> Log request (if trace enabled)
           └─> FileTracer.logMessage()

2. Downstream Processing
   └─> RaDownstream.handleInputMessage()
       ├─> Validate message format
       ├─> Validate message header
       ├─> Validate message protection
       ├─> Validate message body
       ├─> Check transaction state (via PersistencyContextManager)
       ├─> Apply inventory checks (authorization)
       └─> Route to appropriate handler based on message type:
           ├─> TYPE_INIT_REQ (0) - Initial Request
           ├─> TYPE_CERT_REQ (2) - Certificate Request
           ├─> TYPE_P10_CERT_REQ (4) - PKCS#10 Certificate Request
           ├─> TYPE_KEY_UPDATE_REQ (7) - Key Update Request
           ├─> TYPE_POLL_REQ (25) - Polling Request
           ├─> TYPE_CERT_CONFIRM (24) - Certificate Confirmation
           ├─> TYPE_REVOCATION_REQ (11) - Revocation Request
           └─> TYPE_GEN_MSG (21) - General Message

3. Upstream Processing (if needed)
   └─> CmpRaUpstream handles forwarding to CA
       ├─> Modify request if needed
       ├─> Add/update protection
       ├─> Send via UpstreamExchange.sendReceiveMessage()
       └─> Handle response or delayed delivery

4. Response Generation
   └─> Construct PKIMessage response
       ├─> Apply output protection
       ├─> Update transaction state
       └─> Encode to byte[]

5. Return Response
   └─> Log response (if trace enabled)
       └─> Return byte[] to embedding application
```

#### Asynchronous Response (Delayed Delivery)

**Entry Point:** `CmpRaImplementation.gotResponseAtUpstream()`

**Location:** `src/main/java/com/siemens/pki/cmpracomponent/msgprocessing/CmpRaImplementation.java:111`

**Flow:**

```
1. Receive delayed byte[] response from upstream CA
   └─> Parse to PKIMessage
       └─> Log async response

2. Upstream Processing
   └─> CmpRaUpstream.gotResponseAtUpstream()
       ├─> Match response to pending transaction
       ├─> Validate response
       └─> Store for next client poll

3. Client Polling
   └─> Client sends TYPE_POLL_REQ
       └─> RA retrieves stored response
           └─> Returns to client
```

---

## CMP Client Component Execution Flow

### 1. Component Instantiation

**Entry Point:** `CmpClient` constructor

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:162`

```java
CmpClient client = new CmpClient(
    certProfile,           // Certificate profile (optional)
    upstreamExchange,      // Interface to server (RA/CA)
    upstreamConfiguration, // CMP message configuration
    clientContext          // Client-specific configuration
);
```

**Initialization Flow:**

1. **Configuration Setup:**
   - `certProfile` - Identifies the certificate type being requested
   - `upstreamExchange` - Application-provided interface for network communication
   - `upstreamConfiguration` - Message protection and verification settings
   - `clientContext` - Contains enrollment or revocation context

2. **Internal Component Creation:**
   - Creates `ClientRequestHandler` with configuration
   - Sets up message validators and protectors
   - Initializes header provider for request generation

### 2. Certificate Enrollment Flow

**Entry Point:** `CmpClient.invokeEnrollment()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:409`

**Execution Steps:**

```
1. Prepare Enrollment Request
   └─> Get EnrollmentContext from ClientContext
       ├─> Determine enrollment type:
       │   ├─> TYPE_INIT_REQ (0) - Initial enrollment
       │   ├─> TYPE_CERT_REQ (2) - Certificate request
       │   ├─> TYPE_P10_CERT_REQ (4) - PKCS#10 request
       │   └─> TYPE_KEY_UPDATE_REQ (7) - Key update
       │
       ├─> Get or generate key pair
       │   └─> If null, central key generation is requested
       │
       └─> Build certificate template:
           ├─> Subject DN
           ├─> Public key
           ├─> Extensions
           └─> Controls (for KUR: oldCertID)

2. Generate Request Message
   └─> ClientRequestHandler.buildInitialRequest()
       ├─> Create PKIHeader
       │   ├─> Sender (client DN)
       │   ├─> Recipient (CA/RA DN)
       │   ├─> Transaction ID
       │   ├─> Nonce
       │   └─> Certificate profile (in generalInfo)
       │
       ├─> Create PKIBody with enrollment type
       │
       └─> Apply message protection:
           ├─> Signature-based (with client certificate)
           ├─> MAC-based (with shared secret)
           └─> Password-based MAC

3. Send and Receive
   └─> ClientRequestHandler.sendReceiveValidateMessage()
       ├─> Encode PKIMessage to byte[]
       ├─> Send via UpstreamExchange.sendReceiveMessage()
       ├─> Receive byte[] response
       ├─> Decode to PKIMessage
       └─> Validate response:
           ├─> Header validation
           ├─> Protection validation
           └─> Body validation

4. Handle Delayed Delivery (if needed)
   └─> If response is null (timeout):
       ├─> Enter polling loop
       ├─> Send TYPE_POLL_REQ periodically
       └─> Wait for TYPE_POLL_REP with certificate

5. Process Enrollment Response
   └─> Extract certificate from CertRepMessage
       ├─> Check status (GRANTED or GRANTED_WITH_MODS)
       ├─> Handle encrypted certificate (if present)
       │   └─> Decrypt using private key
       ├─> For central key generation:
       │   ├─> Decrypt private key from response
       │   └─> Verify signature on private key
       └─> Build certificate chain (if trust provided)

6. Certificate Confirmation (if needed)
   └─> If implicit confirm not granted:
       ├─> Build TYPE_CERT_CONFIRM message
       ├─> Send to server
       └─> Receive TYPE_CONFIRM (PKIConf)

7. Return EnrollmentResult
   └─> Return object containing:
       ├─> Enrolled certificate (X509Certificate)
       ├─> Certificate chain (List<X509Certificate>)
       └─> Private key (PrivateKey)
```

### 3. Certificate Revocation Flow

**Entry Point:** `CmpClient.invokeRevocation()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:634`

**Execution Steps:**

```
1. Prepare Revocation Request
   └─> Get RevocationContext from ClientContext
       ├─> Issuer DN
       ├─> Serial number
       └─> Revocation reason

2. Generate Revocation Request
   └─> PkiMessageGenerator.generateRrBody()
       ├─> Create RevDetails with CertId
       └─> Include revocation reason

3. Send and Receive
   └─> ClientRequestHandler.sendReceiveInitialBody()
       ├─> Build and protect message
       ├─> Send via upstream
       └─> Validate response

4. Process Revocation Response
   └─> Extract RevRepContent
       ├─> Check status (GRANTED)
       └─> Return success/failure

5. Return Result
   └─> Return boolean (true = success)
```

### 4. Support Message Flows

#### Get CA Certificates

**Entry Point:** `CmpClient.getCaCertificates()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:205`

```
1. Build General Message (GENM)
   └─> InfoTypeAndValue with id_it_caCerts

2. Send Request
   └─> ClientRequestHandler.sendReceiveInitialBody()

3. Parse General Response (GENREP)
   └─> Extract certificate sequence
       └─> Convert to List<X509Certificate>

4. Return Certificates
```

#### Get Certificate Request Template

**Entry Point:** `CmpClient.getCertificateRequestTemplate()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:232`

```
1. Build GENM with id_it_certReqTemplate

2. Send and receive GENREP

3. Extract CertReqTemplateContent

4. Return as byte[] (ASN.1 encoded)
```

#### Get CRLs

**Entry Point:** `CmpClient.getCrls()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:261`

```
1. Build CRLStatus with distribution point info

2. Send GENM with id_it_crlStatusList

3. Parse GENREP with id_it_crls

4. Extract and return List<X509CRL>
```

#### Get Root CA Certificate Update

**Entry Point:** `CmpClient.getRootCaCertificateUpdate()`

**Location:** `src/main/java/com/siemens/pki/cmpclientcomponent/main/CmpClient.java:330`

```
1. Build GENM with id_it_rootCaCert
   └─> Include old root CA cert (optional)

2. Send and receive GENREP

3. Extract RootCaKeyUpdateContent
   └─> Contains:
       ├─> newWithNew (new CA cert signed by new key)
       ├─> newWithOld (new CA cert signed by old key)
       └─> oldWithNew (old CA cert signed by new key)

4. Return RootCaCertificateUpdateResponse
```

---

## Message Processing Pipeline

### Common Processing Stages

Both RA and Client components follow a similar message processing pipeline:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. INCOMING MESSAGE RECEPTION                               │
│    - Receive byte[] from transport layer                    │
│    - Parse ASN.1 DER to PKIMessage object                   │
│    - Log message (if tracing enabled)                       │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. HEADER VALIDATION                                        │
│    - Check protocol version (pvno)                          │
│    - Validate sender/recipient                              │
│    - Check transaction ID format                            │
│    - Verify nonce presence (if required)                    │
│    - Extract certificate profile (from generalInfo)         │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. PROTECTION VALIDATION                                    │
│    - Determine protection type:                             │
│      ├─> Signature-based (verify with trusted certs)        │
│      ├─> MAC-based (verify with shared secret)              │
│      ├─> Password-based MAC (PBMAC1 or PasswordBasedMac)    │
│      └─> No protection (only for specific message types)    │
│    - Verify protection integrity                            │
│    - Validate signer certificate chain (if signature)       │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. BODY VALIDATION                                          │
│    - Validate message type is expected                      │
│    - Check body content structure                           │
│    - Validate certificates/keys (if present)                │
│    - Verify request parameters                              │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. BUSINESS LOGIC PROCESSING                                │
│    RA Component:                                            │
│    - Check inventory authorization                          │
│    - Modify request if needed                               │
│    - Forward to CA (if applicable)                          │
│    - Manage transaction state                               │
│                                                             │
│    Client Component:                                        │
│    - Extract enrollment result                              │
│    - Decrypt encrypted content (if needed)                  │
│    - Verify certificate chain                               │
│    - Build confirmation (if needed)                         │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. RESPONSE GENERATION                                      │
│    - Create PKIHeader (mirror transaction ID, sender/recip) │
│    - Create PKIBody with response content                   │
│    - Apply output protection                                │
│    - Build complete PKIMessage                              │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. OUTGOING MESSAGE TRANSMISSION                            │
│    - Encode PKIMessage to ASN.1 DER                         │
│    - Log message (if tracing enabled)                       │
│    - Return byte[] to transport layer                       │
└─────────────────────────────────────────────────────────────┘
```

### Error Handling

Errors are handled at multiple levels:

1. **CMP-level Errors** - Returned as PKIMessage with TYPE_ERROR:
   - Invalid request format
   - Authorization failures
   - Certificate request rejections
   - Uses PKIFailureInfo to specify error type

2. **Java Exceptions** - Thrown for fatal errors:
   - Invalid configuration
   - Crypto failures
   - Network errors
   - Parse errors

3. **Logging** - Via SLF4J at various levels:
   - ERROR - Critical failures
   - WARN - Unexpected conditions
   - INFO - Normal operations
   - DEBUG - Detailed flow information
   - TRACE - Full message dumps

---

## Key Components and Their Roles

### RA Component Architecture

```
CmpRaComponent (Entry Point)
    │
    ├─> CmpRaImplementation
    │   │
    │   ├─> RaDownstream
    │   │   ├─> Receives requests from clients (EEs)
    │   │   ├─> Validates incoming messages
    │   │   ├─> Applies inventory checks
    │   │   ├─> Routes to upstream if needed
    │   │   └─> Generates responses to clients
    │   │
    │   └─> CmpRaUpstream
    │       ├─> Forwards requests to CA
    │       ├─> Modifies request protection
    │       ├─> Handles delayed delivery
    │       └─> Validates CA responses
    │
    ├─> P10X509RaImplementation (Legacy PKI)
    │   ├─> Handles PKCS#10 CSR
    │   └─> Returns X.509 certificates
    │
    └─> Supporting Components
        ├─> PersistencyContextManager
        │   └─> Manages transaction state across restarts
        │
        ├─> MsgOutputProtector
        │   └─> Applies message protection
        │
        ├─> ProtectionValidator
        │   └─> Validates message protection
        │
        ├─> MessageHeaderValidator
        │   └─> Validates PKI headers
        │
        └─> MessageBodyValidator
            └─> Validates PKI bodies
```

### Client Component Architecture

```
CmpClient (Entry Point)
    │
    ├─> ClientRequestHandler
    │   ├─> Builds requests
    │   ├─> Sends to server (RA/CA)
    │   ├─> Handles polling for delayed delivery
    │   └─> Validates responses
    │
    └─> Supporting Components
        ├─> PkiMessageGenerator
        │   └─> Creates PKI messages
        │
        ├─> MsgOutputProtector
        │   └─> Protects outgoing messages
        │
        ├─> ProtectionValidator
        │   └─> Validates incoming protection
        │
        ├─> MessageHeaderValidator
        │   └─> Validates response headers
        │
        ├─> MessageBodyValidator
        │   └─> Validates response bodies
        │
        ├─> CmsDecryptor
        │   └─> Decrypts encrypted certificates and keys
        │
        └─> TrustCredentialAdapter
            └─> Validates certificate chains
```

### Crypto Services Layer

```
cryptoservices/
    ├─> AlgorithmHelper - OID and algorithm utilities
    ├─> CertUtility - Certificate parsing and conversion
    ├─> DataSigner - Creates signatures
    ├─> DataSignVerifier - Verifies signatures
    ├─> CmsEncryptorBase - Encrypts data (CMS)
    ├─> CmsDecryptor - Decrypts data (CMS)
    ├─> KeyAgreementEncryptor - Key agreement encryption
    ├─> KeyTransportEncryptor - Key transport encryption
    ├─> PasswordEncryptor - Password-based encryption
    └─> TrustCredentialAdapter - Certificate chain validation
```

### Protection Layer

```
protection/
    ├─> ProtectionProvider (interface)
    ├─> SignatureBasedProtection - Signature protection
    ├─> MacProtection - MAC protection (shared secret)
    ├─> PBMAC1Protection - Password-based MAC (PBMAC1)
    ├─> PasswordBasedMacProtection - Password-based MAC (legacy)
    └─> NoProtection - No protection (error messages only)
```

### Validation Layer

```
msgvalidation/
    ├─> InputValidator - Entry point validation
    ├─> MessageHeaderValidator - Header validation
    ├─> ProtectionValidator - Protection validation
    │   ├─> SignatureProtectionValidator
    │   ├─> MacValidator
    │   ├─> PBMAC1ProtectionValidator
    │   └─> PasswordBasedMacValidator
    ├─> MessageBodyValidator - Body validation
    └─> Exception hierarchy:
        ├─> BaseCmpException
        ├─> CmpValidationException - Validation failures
        ├─> CmpProcessingException - Processing errors
        └─> CmpEnrollmentException - Enrollment errors
```

---

## Sequence Flows

### RA Synchronous Enrollment (IR/CR)

```
Client (EE)          RA Component              CA Server
    │                     │                         │
    │  1. CMP IR/CR       │                         │
    │ ──────────────────> │                         │
    │                     │                         │
    │                     │  2. Validate Request    │
    │                     │     - Header            │
    │                     │     - Protection        │
    │                     │     - Body              │
    │                     │                         │
    │                     │  3. Check Inventory     │
    │                     │     - Authorize         │
    │                     │     - Modify (if needed)│
    │                     │                         │
    │                     │  4. Forward Request     │
    │                     │ ──────────────────────> │
    │                     │                         │
    │                     │                         │ 5. Process
    │                     │                         │    - Validate
    │                     │                         │    - Issue cert
    │                     │                         │
    │                     │  6. CMP IP/CP           │
    │                     │ <────────────────────── │
    │                     │                         │
    │                     │  7. Validate Response   │
    │                     │     - Protection        │
    │                     │     - Certificate       │
    │                     │                         │
    │                     │  8. Update Inventory    │
    │                     │     - Log enrollment    │
    │                     │                         │
    │  9. CMP IP/CP       │                         │
    │ <────────────────── │                         │
    │                     │                         │
    │ 10. CertConfirm     │                         │
    │ ──────────────────> │                         │
    │                     │                         │
    │                     │ 11. Forward Confirm     │
    │                     │ ──────────────────────> │
    │                     │                         │
    │                     │ 12. PKIConfirm          │
    │                     │ <────────────────────── │
    │                     │                         │
    │ 13. PKIConfirm      │                         │
    │ <────────────────── │                         │
    │                     │                         │
```

### RA Asynchronous Enrollment (with Polling)

```
Client (EE)          RA Component              CA Server
    │                     │                         │
    │  1. CMP IR/CR       │                         │
    │ ──────────────────> │                         │
    │                     │                         │
    │                     │  2. Validate & Forward  │
    │                     │ ──────────────────────> │
    │                     │                         │
    │                     │  3. null (timeout)      │
    │                     │ <────────────────────── │
    │                     │                         │
    │                     │  4. Store transaction   │
    │                     │     state               │
    │                     │                         │
    │  5. IP/CP (waiting) │                         │
    │ <────────────────── │                         │
    │                     │                         │
    │  Wait...            │                         │
    │                     │                         │ 6. Process
    │                     │                         │    complete
    │                     │                         │
    │                     │  7. gotResponseAtUpstream│
    │                     │ <────────────────────── │
    │                     │                         │
    │                     │  8. Store response      │
    │                     │     for transaction     │
    │                     │                         │
    │  9. PollReq         │                         │
    │ ──────────────────> │                         │
    │                     │                         │
    │                     │ 10. Check state         │
    │                     │     - Response ready?   │
    │                     │                         │
    │ 11. PollRep (wait)  │                         │
    │ <────────────────── │                         │
    │                     │                         │
    │  Wait more...       │                         │
    │                     │                         │
    │ 12. PollReq         │                         │
    │ ──────────────────> │                         │
    │                     │                         │
    │                     │ 13. Retrieve response   │
    │                     │                         │
    │ 14. IP/CP (cert)    │                         │
    │ <────────────────── │                         │
    │                     │                         │
    │ 15. CertConfirm...  │                         │
    │                     │                         │
```

### Client Enrollment with Central Key Generation

```
Client              Server (RA/CA)
  │                       │
  │  1. Build IR/CR       │
  │     - No private key  │
  │     - No public key   │
  │     - pvno = CMP_2021 │
  │                       │
  │  2. Send Request      │
  │ ────────────────────> │
  │                       │
  │                       │  3. Generate key pair
  │                       │     - Create private key
  │                       │     - Create public key
  │                       │
  │                       │  4. Issue certificate
  │                       │     - Use generated public key
  │                       │
  │                       │  5. Encrypt private key
  │                       │     - Use client's credentials
  │                       │     - Sign encrypted key
  │                       │
  │  6. Receive IP/CP     │
  │ <──────────────────── │
  │     - Certificate     │
  │     - Encrypted key   │
  │                       │
  │  7. Decrypt key       │
  │     - Use own creds   │
  │                       │
  │  8. Verify key        │
  │     - Check signature │
  │                       │
  │  9. Return result     │
  │     - Certificate     │
  │     - Private key     │
  │     - Chain           │
  │                       │
```

### Client General Message (GetCACerts)

```
Client              Server (RA/CA)
  │                       │
  │  1. Build GENM        │
  │     - id_it_caCerts   │
  │                       │
  │  2. Send Request      │
  │ ────────────────────> │
  │                       │
  │                       │  3. Retrieve CA certs
  │                       │     - Current CA cert
  │                       │     - Intermediate certs
  │                       │
  │  4. Receive GENREP    │
  │ <──────────────────── │
  │     - Certificate seq │
  │                       │
  │  5. Parse certs       │
  │     - Convert to X509 │
  │                       │
  │  6. Return list       │
  │                       │
```

---

## Configuration and Customization

### Configuration Interfaces

The component uses a hierarchical configuration system:

1. **Configuration (RA)** - Main configuration interface
   - `getVerification()` - Trust anchors and verification settings
   - `getOutputCredentials()` - RA credentials for signing
   - `getUpstreamConfiguration()` - Settings for upstream CA communication
   - `getInventory()` - Authorization and logging hooks
   - `getPersistency()` - Transaction state persistence
   - `getCkgConfiguration()` - Central key generation settings
   - `getRetryAfterTimeInSeconds()` - Polling interval

2. **ClientContext (Client)** - Client configuration
   - `getEnrollmentContext()` - Enrollment parameters
   - `getRevocationContext()` - Revocation parameters

3. **CmpMessageInterface** - Message-level settings
   - `getRecipient()` - Server identifier
   - `getSender()` - Client identifier
   - `getInputVerification()` - Trust for validating responses
   - `getOutputCredentials()` - Credentials for protecting requests

### Extension Points

1. **InventoryInterface** - Custom authorization logic
   - `checkAndModifyRequest()` - Authorize and modify requests
   - `updateEnrollmentStatus()` - Track enrollment state

2. **PersistencyInterface** - Custom state storage
   - `persistState()` - Save transaction state
   - `restoreState()` - Load transaction state

3. **UpstreamExchange** - Custom transport
   - `sendReceiveMessage()` - Send request, receive response
   - Support for synchronous and asynchronous delivery

4. **Support Message Handlers**
   - `GetCaCertificatesHandler` - Custom CA cert retrieval
   - `GetCertificateRequestTemplateHandler` - Custom template
   - `GetRootCaCertificateUpdateHandler` - Custom root update
   - `CrlUpdateRetrievalHandler` - Custom CRL retrieval

---

## Threading and Concurrency

### Thread Safety

- **RA Component**: Thread-safe for concurrent request processing
  - Each request is processed independently
  - Shared state (persistency) uses synchronization
  - Configuration is read-only after initialization

- **Client Component**: NOT thread-safe
  - Create separate instances for concurrent operations
  - Single transaction per instance

### Asynchronous Operations

- Delayed delivery uses polling, not callbacks
- No internal threading - relies on embedding application
- Transaction state stored for recovery

---

## Logging and Debugging

### Log Levels

- **ERROR**: Configuration errors, enrollment failures, validation errors
- **WARN**: Unexpected responses, deprecated features
- **INFO**: Transaction start/complete (not currently used)
- **DEBUG**: Configuration access, detailed flow
- **TRACE**: Full message dumps (ASN.1 structure)

### File Tracing

Enable file tracing by setting system property:
```
-Dcom.siemens.pki.cmpracomponent.util.FileTracer.directory=/path/to/trace
```

Messages are logged as:
- `<timestamp>_<interface>_request.der`
- `<timestamp>_<interface>_response.der`

### Configuration Logging

Enable configuration access logging:
```
-Dorg.slf4j.simpleLogger.log.com.siemens.pki.cmpracomponent.util.ConfigLogger=debug
```

---

## Error Recovery and Resilience

### Transaction Recovery

1. **Persistency Support**
   - Transaction state saved before upstream communication
   - State includes: transaction ID, request, response (if received)
   - On restart, pending transactions can be resumed

2. **Retry Logic**
   - Embedding application controls retry
   - Component provides error information via exceptions
   - Delayed delivery provides built-in retry via polling

### Failure Modes

1. **Validation Failure** → CMP Error response
2. **CA Unavailable** → CMP Error or waiting indication
3. **Network Timeout** → Delayed delivery (polling)
4. **Configuration Error** → Exception during instantiation
5. **Crypto Failure** → Exception with details

---

## Performance Considerations

### Optimization Points

1. **Message Validation**
   - Can skip certain validations based on configuration
   - Trust anchors cached by BouncyCastle

2. **Crypto Operations**
   - Private key operations are main bottleneck
   - Use hardware crypto if available (via JCE)

3. **Persistency**
   - Use efficient storage (database vs. file)
   - Consider transaction expiry

4. **Logging**
   - Disable TRACE in production
   - File tracing has I/O overhead

### Scalability

- **RA Component**: Stateless processing (except persistency)
  - Scale horizontally with shared persistency backend
  - Load balancer needs session affinity for polling

- **Client Component**: One instance per transaction
  - Create new instances for parallel enrollments

---

## References

- [Main README](README.md) - Component overview and features
- [Sequence Diagrams](doc/) - UML sequence diagrams
- [JavaDoc](target/site/apidocs/) - Complete API documentation
- [RFC 9483](https://datatracker.ietf.org/doc/rfc9483/) - Lightweight CMP Profile
- [RFC 9810](https://datatracker.ietf.org/doc/rfc9810/) - CMP Updated Specification
- [RFC 9481](https://datatracker.ietf.org/doc/rfc9481/) - CMP Algorithms
