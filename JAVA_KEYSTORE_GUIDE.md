# Java KeyStore - Step-by-Step Command Guide

**Quick Reference:** Ready-to-run commands with one-line explanations for team presentations.

---

## Table of Contents
- [Getting Started](#getting-started)
- [Basic KeyStore Operations](#basic-keystore-operations)
- [Working with Certificates](#working-with-certificates)
- [Certificate Signing Requests (CSR)](#certificate-signing-requests-csr)
- [Import/Export Operations](#importexport-operations)
- [KeyStore Conversion](#keystore-conversion)
- [Security Operations](#security-operations)
- [Troubleshooting Commands](#troubleshooting-commands)
- [Java Code Examples](#java-code-examples)

---

## Getting Started

### What is Java KeyStore?

A secure database for storing:
- **Private Keys** - Your secret keys for signing/decryption
- **Certificates** - Your public certificates
- **Trusted Certificates** - CA certificates you trust

### KeyStore Types

| Type | Description | Extension | Since |
|------|-------------|-----------|-------|
| **PKCS12** | Industry standard (RECOMMENDED) | `.p12`, `.pfx` | Java 9+ default |
| **JKS** | Legacy Java format | `.jks` | Before Java 9 |
| **JCEKS** | Enhanced security, supports symmetric keys | `.jceks` | - |

---

## Basic KeyStore Operations

### 1. Create a New KeyStore with Key Pair

```bash
keytool -genkeypair -alias myapp -keyalg RSA -keysize 2048 -validity 365 -keystore mykeystore.p12 -storepass changeit -keypass changeit -dname "CN=My Application, OU=IT, O=MyCompany, L=NewYork, ST=NY, C=US"
```
**What it does:** Creates new KeyStore with RSA key pair and self-signed certificate

**Parameters explained:**
- `-alias myapp` - Name to identify this key entry
- `-keyalg RSA` - Use RSA algorithm
- `-keysize 2048` - 2048-bit key (minimum recommended)
- `-validity 365` - Certificate valid for 365 days
- `-keystore mykeystore.p12` - Output KeyStore file name
- `-storepass changeit` - KeyStore password
- `-keypass changeit` - Private key password
- `-dname` - Distinguished Name (certificate subject)

---

### 2. List All Entries in KeyStore

```bash
keytool -list -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Shows all aliases and certificate fingerprints in KeyStore

---

### 3. List with Full Details (Verbose)

```bash
keytool -list -v -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Shows complete certificate details, validity dates, signature algorithm, etc.

---

### 4. Check Specific Entry

```bash
keytool -list -alias myapp -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Shows details for a specific alias only

---

### 5. List Without Password Prompt (if password is default)

```bash
keytool -list -keystore mykeystore.p12 -storepass changeit -noprompt
```
**What it does:** Lists entries without interactive prompts

---

## Working with Certificates

### 6. Export Certificate to File

```bash
keytool -exportcert -alias myapp -file myapp.cer -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Exports certificate in DER format (binary)

---

### 7. Export Certificate in PEM Format (Human Readable)

```bash
keytool -exportcert -alias myapp -file myapp.pem -rfc -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Exports certificate in PEM format (Base64 encoded, starts with -----BEGIN CERTIFICATE-----)

---

### 8. Import Trusted CA Certificate

```bash
keytool -importcert -alias root-ca -file ca-certificate.pem -keystore mykeystore.p12 -storepass changeit -noprompt
```
**What it does:** Imports a CA certificate as a trusted entry (no private key)

---

### 9. Import Certificate Reply (After CSR was Signed)

```bash
keytool -importcert -alias myapp -file signed-certificate.pem -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Imports CA-signed certificate, replacing self-signed cert (keeps private key)

---

### 10. Import Certificate Chain

```bash
keytool -importcert -alias root-ca -file root.pem -keystore mykeystore.p12 -storepass changeit -noprompt
keytool -importcert -alias intermediate-ca -file intermediate.pem -keystore mykeystore.p12 -storepass changeit -noprompt
keytool -importcert -alias myapp -file myapp-signed.pem -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Imports complete chain: root → intermediate → end-entity (MUST be in this order)

---

## Certificate Signing Requests (CSR)

### 11. Generate CSR from Existing Key

```bash
keytool -certreq -alias myapp -file myapp.csr -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Creates CSR file to send to CA for signing

---

### 12. Generate CSR with Extensions

```bash
keytool -certreq -alias myapp -file myapp.csr -ext san=dns:example.com,dns:www.example.com,ip:192.168.1.1 -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Creates CSR with Subject Alternative Names (SAN) for multiple domains/IPs

---

### 13. View CSR Contents

```bash
keytool -printcertreq -file myapp.csr
```
**What it does:** Displays CSR details (subject, public key, extensions)

---

## Import/Export Operations

### 14. Delete Entry from KeyStore

```bash
keytool -delete -alias myapp -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Removes specified entry (private key + certificate or trusted cert)

---

### 15. Rename Alias

```bash
keytool -changealias -alias oldname -destalias newname -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Renames an existing alias without changing the entry

---

### 16. Print Certificate from File

```bash
keytool -printcert -file certificate.pem
```
**What it does:** Displays certificate details without needing KeyStore

---

### 17. Print Certificate from URL

```bash
keytool -printcert -sslserver google.com:443
```
**What it does:** Shows SSL certificate from a remote server

---

## KeyStore Conversion

### 18. Convert JKS to PKCS12

```bash
keytool -importkeystore -srckeystore old.jks -destkeystore new.p12 -srcstoretype JKS -deststoretype PKCS12 -srcstorepass changeit -deststorepass changeit
```
**What it does:** Converts legacy JKS format to modern PKCS12 format

---

### 19. Convert PKCS12 to JKS

```bash
keytool -importkeystore -srckeystore keystore.p12 -destkeystore keystore.jks -srcstoretype PKCS12 -deststoretype JKS -srcstorepass changeit -deststorepass changeit
```
**What it does:** Converts PKCS12 to JKS (for legacy Java applications)

---

### 20. Merge Two KeyStores

```bash
keytool -importkeystore -srckeystore source.p12 -destkeystore destination.p12 -srcstorepass changeit -deststorepass changeit
```
**What it does:** Copies all entries from source to destination KeyStore

---

## Security Operations

### 21. Change KeyStore Password

```bash
keytool -storepasswd -keystore mykeystore.p12
```
**What it does:** Prompts to change the KeyStore password interactively

---

### 22. Change KeyStore Password (Non-Interactive)

```bash
keytool -storepasswd -new newpassword -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Changes KeyStore password without prompts

---

### 23. Change Private Key Password

```bash
keytool -keypasswd -alias myapp -keystore mykeystore.p12 -storepass changeit
```
**What it does:** Changes password for a specific private key

---

### 24. Change Private Key Password (Non-Interactive)

```bash
keytool -keypasswd -alias myapp -new newkeypass -keystore mykeystore.p12 -storepass changeit -keypass changeit
```
**What it does:** Changes key password without prompts

---

## Troubleshooting Commands

### 25. Check KeyStore Integrity

```bash
keytool -list -keystore mykeystore.p12 -storepass changeit > /dev/null && echo "KeyStore is valid" || echo "KeyStore is corrupted"
```
**What it does:** Verifies KeyStore can be read successfully

---

### 26. Find Certificate Expiry Date

```bash
keytool -list -v -alias myapp -keystore mykeystore.p12 -storepass changeit | grep "Valid"
```
**What it does:** Shows validity period (from/until dates)

---

### 27. Check Certificate Chain Completeness

```bash
keytool -list -v -alias myapp -keystore mykeystore.p12 -storepass changeit | grep "Certificate chain length"
```
**What it does:** Shows how many certificates in chain (should be ≥2 for CA-signed)

---

### 28. Verify Certificate Matches Private Key

```bash
keytool -list -v -alias myapp -keystore mykeystore.p12 -storepass changeit | grep "Signature algorithm"
```
**What it does:** Shows signature algorithm (RSA, ECDSA, etc.)

---

### 29. Export and View Certificate Details

```bash
keytool -exportcert -alias myapp -file temp.cer -keystore mykeystore.p12 -storepass changeit && keytool -printcert -file temp.cer && rm temp.cer
```
**What it does:** Exports, displays, and cleans up temporary certificate file

---

### 30. List Only Trusted Certificates

```bash
keytool -list -keystore mykeystore.p12 -storepass changeit | grep "trustedCertEntry"
```
**What it does:** Filters to show only trusted CA certificates

---

### 31. List Only Private Key Entries

```bash
keytool -list -keystore mykeystore.p12 -storepass changeit | grep "PrivateKeyEntry"
```
**What it does:** Filters to show only entries with private keys

---

## Complete Workflow Examples

### Workflow A: Create Self-Signed Certificate for Development

```bash
# Step 1: Generate key pair with self-signed certificate
keytool -genkeypair -alias dev-app -keyalg RSA -keysize 2048 -validity 365 -keystore dev.p12 -storepass devpass -keypass devpass -dname "CN=DevApp, OU=Development, O=MyCompany, C=US"

# Step 2: Verify creation
keytool -list -v -keystore dev.p12 -storepass devpass

# Step 3: Export certificate for sharing
keytool -exportcert -alias dev-app -file dev-app.pem -rfc -keystore dev.p12 -storepass devpass
```

---

### Workflow B: Get CA-Signed Certificate

```bash
# Step 1: Generate key pair
keytool -genkeypair -alias prod-app -keyalg RSA -keysize 2048 -validity 365 -keystore prod.p12 -storepass prodpass -keypass prodpass -dname "CN=app.example.com, OU=Production, O=MyCompany, C=US"

# Step 2: Generate CSR
keytool -certreq -alias prod-app -file prod-app.csr -ext san=dns:app.example.com,dns:www.app.example.com -keystore prod.p12 -storepass prodpass

# Step 3: Send prod-app.csr to CA and receive signed certificate
# (Wait for CA to sign and return: root.pem, intermediate.pem, prod-app-signed.pem)

# Step 4: Import root CA certificate
keytool -importcert -alias root-ca -file root.pem -keystore prod.p12 -storepass prodpass -noprompt

# Step 5: Import intermediate CA certificate
keytool -importcert -alias intermediate-ca -file intermediate.pem -keystore prod.p12 -storepass prodpass -noprompt

# Step 6: Import signed certificate (replaces self-signed)
keytool -importcert -alias prod-app -file prod-app-signed.pem -keystore prod.p12 -storepass prodpass

# Step 7: Verify certificate chain
keytool -list -v -alias prod-app -keystore prod.p12 -storepass prodpass
```

---

### Workflow C: Setup TrustStore for Java Application

```bash
# Step 1: Create empty truststore
keytool -genkeypair -alias temp -keyalg RSA -keystore truststore.p12 -storepass trustpass -keypass trustpass -dname "CN=temp"
keytool -delete -alias temp -keystore truststore.p12 -storepass trustpass

# Step 2: Import trusted CA certificate
keytool -importcert -alias company-root-ca -file company-root-ca.pem -keystore truststore.p12 -storepass trustpass -noprompt

# Step 3: Import public CA certificate (e.g., Let's Encrypt)
keytool -importcert -alias letsencrypt-root -file letsencrypt-root.pem -keystore truststore.p12 -storepass trustpass -noprompt

# Step 4: List all trusted certificates
keytool -list -keystore truststore.p12 -storepass trustpass

# Step 5: Use in Java application
# java -Djavax.net.ssl.trustStore=truststore.p12 -Djavax.net.ssl.trustStorePassword=trustpass -jar myapp.jar
```

---

### Workflow D: Migrate from JKS to PKCS12

```bash
# Step 1: Backup original JKS file
cp old.jks old.jks.backup

# Step 2: Convert JKS to PKCS12
keytool -importkeystore -srckeystore old.jks -destkeystore new.p12 -srcstoretype JKS -deststoretype PKCS12 -srcstorepass oldpass -deststorepass newpass

# Step 3: Verify conversion
keytool -list -keystore new.p12 -storepass newpass

# Step 4: Compare entry counts
echo "Old JKS entries:" && keytool -list -keystore old.jks -storepass oldpass | grep -c "Entry type"
echo "New PKCS12 entries:" && keytool -list -keystore new.p12 -storepass newpass | grep -c "Entry type"
```

---

## Java Code Examples

### Example 1: Load KeyStore and Get Private Key

```java
import java.io.FileInputStream;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.Certificate;

public class LoadKeyStore {
    public static void main(String[] args) throws Exception {
        // Load KeyStore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream("mykeystore.p12")) {
            keyStore.load(fis, "changeit".toCharArray());
        }

        // Get private key
        String alias = "myapp";
        PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, "changeit".toCharArray());
        System.out.println("Private Key Algorithm: " + privateKey.getAlgorithm());

        // Get certificate chain
        Certificate[] chain = keyStore.getCertificateChain(alias);
        System.out.println("Certificate Chain Length: " + chain.length);

        // Print certificate subject
        if (chain.length > 0) {
            java.security.cert.X509Certificate cert = (java.security.cert.X509Certificate) chain[0];
            System.out.println("Certificate Subject: " + cert.getSubjectX500Principal());
            System.out.println("Valid Until: " + cert.getNotAfter());
        }
    }
}
```

**Run:** `javac LoadKeyStore.java && java LoadKeyStore`

---

### Example 2: Create New KeyStore Programmatically

```java
import java.io.FileOutputStream;
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.io.FileInputStream;

public class CreateKeyStore {
    public static void main(String[] args) throws Exception {
        // Create empty KeyStore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(null, null);  // Initialize empty

        // Load certificate from file
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        Certificate cert;
        try (FileInputStream fis = new FileInputStream("certificate.pem")) {
            cert = cf.generateCertificate(fis);
        }

        // Add certificate to KeyStore
        keyStore.setCertificateEntry("trusted-ca", cert);

        // Save KeyStore to file
        try (FileOutputStream fos = new FileOutputStream("newkeystore.p12")) {
            keyStore.store(fos, "password".toCharArray());
        }

        System.out.println("KeyStore created successfully!");
    }
}
```

**Run:** `javac CreateKeyStore.java && java CreateKeyStore`

---

### Example 3: List All Aliases in KeyStore

```java
import java.io.FileInputStream;
import java.security.KeyStore;
import java.util.Enumeration;

public class ListAliases {
    public static void main(String[] args) throws Exception {
        // Load KeyStore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream("mykeystore.p12")) {
            keyStore.load(fis, "changeit".toCharArray());
        }

        // List all aliases
        System.out.println("Aliases in KeyStore:");
        Enumeration<String> aliases = keyStore.aliases();
        while (aliases.hasMoreElements()) {
            String alias = aliases.nextElement();
            boolean isKey = keyStore.isKeyEntry(alias);
            boolean isCert = keyStore.isCertificateEntry(alias);

            System.out.printf("  - %s [%s]%n",
                alias,
                isKey ? "PrivateKeyEntry" : (isCert ? "TrustedCertEntry" : "Unknown"));
        }
    }
}
```

**Run:** `javac ListAliases.java && java ListAliases`

---

### Example 4: Setup SSL Context with KeyStore

```java
import javax.net.ssl.*;
import java.io.FileInputStream;
import java.security.KeyStore;

public class SSLContextSetup {
    public static void main(String[] args) throws Exception {
        // Load KeyStore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream("keystore.p12")) {
            keyStore.load(fis, "password".toCharArray());
        }

        // Load TrustStore
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream("truststore.p12")) {
            trustStore.load(fis, "trustpass".toCharArray());
        }

        // Initialize KeyManagerFactory (for client authentication)
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(
            KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(keyStore, "password".toCharArray());

        // Initialize TrustManagerFactory (for server validation)
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(trustStore);

        // Create SSL Context
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);

        // Set as default for all HTTPS connections
        HttpsURLConnection.setDefaultSSLSocketFactory(sslContext.getSocketFactory());

        System.out.println("SSL Context configured successfully!");
    }
}
```

**Run:** `javac SSLContextSetup.java && java SSLContextSetup`

---

### Example 5: Using KeyStore with CMP RA Component

```java
import java.io.FileInputStream;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.X509Certificate;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;
import com.siemens.pki.cmpracomponent.configuration.SignatureCredentialContext;

public class CmpRaKeyStoreConfig {
    public static SignatureCredentialContext loadCredentials(
            String keystorePath,
            String keystorePassword,
            String alias,
            String keyPassword) throws Exception {

        // Load KeyStore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, keystorePassword.toCharArray());
        }

        // Get private key
        PrivateKey privateKey = (PrivateKey) keyStore.getKey(
            alias,
            keyPassword.toCharArray());

        // Get certificate chain
        List<X509Certificate> chain = Arrays.stream(keyStore.getCertificateChain(alias))
            .map(cert -> (X509Certificate) cert)
            .collect(Collectors.toList());

        // Create credential context for CMP RA
        return new SignatureCredentialContext() {
            @Override
            public PrivateKey getPrivateKey() {
                return privateKey;
            }

            @Override
            public List<X509Certificate> getCertificateChain() {
                return chain;
            }
        };
    }

    public static void main(String[] args) throws Exception {
        SignatureCredentialContext credentials = loadCredentials(
            "ra-keystore.p12",
            "rapass",
            "ra-signing-key",
            "rapass"
        );

        System.out.println("RA credentials loaded successfully!");
        System.out.println("Private Key: " + credentials.getPrivateKey().getAlgorithm());
        System.out.println("Certificate Chain Length: " + credentials.getCertificateChain().size());
    }
}
```

**Run:** `javac -cp ".:lib/*" CmpRaKeyStoreConfig.java && java -cp ".:lib/*" CmpRaKeyStoreConfig`

---

## Common Use Cases

### Use Case 1: Development Environment Setup

```bash
# Create development keystore with self-signed certificate
keytool -genkeypair -alias dev-server -keyalg RSA -keysize 2048 -validity 365 -keystore dev-server.p12 -storepass devpass -keypass devpass -dname "CN=localhost, OU=Development, O=MyCompany, C=US" -ext san=dns:localhost,ip:127.0.0.1

# Export for browser import (to trust self-signed cert)
keytool -exportcert -alias dev-server -file dev-server.cer -rfc -keystore dev-server.p12 -storepass devpass

# Run Java app with this keystore
# java -Djavax.net.ssl.keyStore=dev-server.p12 -Djavax.net.ssl.keyStorePassword=devpass -jar app.jar
```

---

### Use Case 2: Production Environment Setup

```bash
# Import production certificates from CA
keytool -importcert -alias root-ca -file prod-root-ca.pem -keystore prod.p12 -storepass prodpass -noprompt
keytool -importcert -alias intermediate-ca -file prod-intermediate-ca.pem -keystore prod.p12 -storepass prodpass -noprompt
keytool -importcert -alias prod-server -file prod-server-signed.pem -keystore prod.p12 -storepass prodpass

# Set restrictive permissions
chmod 400 prod.p12

# Run Java app
# java -Djavax.net.ssl.keyStore=prod.p12 -Djavax.net.ssl.keyStorePassword=prodpass -jar app.jar
```

---

### Use Case 3: Mutual TLS (mTLS) Setup

```bash
# Server side: Create server keystore
keytool -genkeypair -alias server -keyalg RSA -keysize 2048 -validity 365 -keystore server.p12 -storepass serverpass -keypass serverpass -dname "CN=server.example.com"

# Client side: Create client keystore
keytool -genkeypair -alias client -keyalg RSA -keysize 2048 -validity 365 -keystore client.p12 -storepass clientpass -keypass clientpass -dname "CN=client.example.com"

# Export server certificate
keytool -exportcert -alias server -file server.cer -rfc -keystore server.p12 -storepass serverpass

# Export client certificate
keytool -exportcert -alias client -file client.cer -rfc -keystore client.p12 -storepass clientpass

# Server: Import client cert as trusted
keytool -importcert -alias trusted-client -file client.cer -keystore server-truststore.p12 -storepass serverpass -noprompt

# Client: Import server cert as trusted
keytool -importcert -alias trusted-server -file server.cer -keystore client-truststore.p12 -storepass clientpass -noprompt

# Server runs with: -Djavax.net.ssl.keyStore=server.p12 -Djavax.net.ssl.trustStore=server-truststore.p12
# Client runs with: -Djavax.net.ssl.keyStore=client.p12 -Djavax.net.ssl.trustStore=client-truststore.p12
```

---

## Quick Reference Commands

### Must-Know Commands

```bash
# 1. List keystore contents
keytool -list -v -keystore keystore.p12 -storepass password

# 2. Generate key pair
keytool -genkeypair -alias mykey -keyalg RSA -keysize 2048 -keystore keystore.p12 -storepass password

# 3. Export certificate
keytool -exportcert -alias mykey -file cert.pem -rfc -keystore keystore.p12 -storepass password

# 4. Import certificate
keytool -importcert -alias trustedca -file ca.pem -keystore keystore.p12 -storepass password -noprompt

# 5. Generate CSR
keytool -certreq -alias mykey -file request.csr -keystore keystore.p12 -storepass password

# 6. Delete entry
keytool -delete -alias mykey -keystore keystore.p12 -storepass password

# 7. Change password
keytool -storepasswd -keystore keystore.p12

# 8. Convert format
keytool -importkeystore -srckeystore old.jks -destkeystore new.p12 -deststoretype PKCS12
```

---

## Security Best Practices

### 1. File Permissions

```bash
# Set read-only for owner
chmod 400 keystore.p12

# Verify permissions
ls -l keystore.p12
# Should show: -r-------- 1 user group
```

---

### 2. Use Strong Passwords

```bash
# Generate random password
openssl rand -base64 32

# Use different passwords for keystore and keys
keytool -genkeypair -alias mykey -keystore keystore.p12 -storepass $(openssl rand -base64 24) -keypass $(openssl rand -base64 24)
```

---

### 3. Don't Commit to Version Control

```bash
# Add to .gitignore
echo "*.p12" >> .gitignore
echo "*.jks" >> .gitignore
echo "*.pfx" >> .gitignore
```

---

### 4. Use Environment Variables

```bash
# Export passwords as environment variables
export KEYSTORE_PASS="mySecurePassword"
export KEY_PASS="mySecureKeyPassword"

# Use in Java
java -Djavax.net.ssl.keyStorePassword=$KEYSTORE_PASS -jar app.jar
```

---

### 5. Regular Rotation

```bash
# Check expiry dates
keytool -list -v -keystore keystore.p12 -storepass password | grep "Valid until"

# Set calendar reminder 30 days before expiry
# Regenerate certificates before expiration
```

---

## Troubleshooting Guide

### Problem: "Keystore was tampered with, or password was incorrect"

```bash
# Try with default password
keytool -list -keystore keystore.p12 -storepass changeit

# If still fails, keystore may be corrupted - restore from backup
```

---

### Problem: "Certificate chain not found for alias"

```bash
# Import certificates in correct order: Root → Intermediate → End Entity
keytool -importcert -alias root -file root.pem -keystore keystore.p12 -storepass password -noprompt
keytool -importcert -alias intermediate -file intermediate.pem -keystore keystore.p12 -storepass password -noprompt
keytool -importcert -alias myapp -file myapp.pem -keystore keystore.p12 -storepass password
```

---

### Problem: "Public keys in reply and keystore don't match"

```bash
# This means the CSR was generated from a different key
# Solution: Generate new CSR from correct keystore
keytool -certreq -alias myapp -file new.csr -keystore keystore.p12 -storepass password

# Submit new CSR to CA
```

---

### Problem: "Alias already exists"

```bash
# Delete existing alias first
keytool -delete -alias myapp -keystore keystore.p12 -storepass password

# Then import new certificate
keytool -importcert -alias myapp -file newcert.pem -keystore keystore.p12 -storepass password
```

---

## Presentation Checklist

- [ ] Ensure Java/OpenJDK is installed: `java -version`
- [ ] Prepare sample certificates and keystores beforehand
- [ ] Set passwords to simple values for demo (e.g., "changeit")
- [ ] Have terminal with large font for audience visibility
- [ ] Keep backup of all files before live demo
- [ ] Test all commands before presentation
- [ ] Prepare rollback plan if demo fails
- [ ] Have this guide open in browser for reference

---

## Additional Resources

- **Oracle KeyTool Documentation:** https://docs.oracle.com/en/java/javase/17/docs/specs/man/keytool.html
- **PKCS#12 Specification:** https://www.rfc-editor.org/rfc/rfc7292
- **X.509 Certificates:** https://www.rfc-editor.org/rfc/rfc5280
- **CMP RA Component:** [README.md](README.md)
- **Execution Flow:** [EXECUTION_FLOW.md](EXECUTION_FLOW.md)

---

**Last Updated:** 2026-01-07
