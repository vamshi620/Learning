# Kafka Security Fundamentals
## File 14: Authentication, Encryption, and Authorization in .NET

---

## What You'll Learn

- Why Kafka needs security
- **Encryption:** TLS/SSL for data in transit
- **Authentication:** Proving *who* you are (SASL/PLAIN, SASL/SCRAM, mTLS)
- **Authorization:** Restricting *what* you can do (ACLs)
- Configuring `.NET Confluent.Kafka` clients for secure environments

---

## 1. The Security Triad in Kafka

By default, Kafka runs in `PLAINTEXT` mode. Anyone who can reach the broker IP can read any topic, write to any topic, and delete any topic. **Never run PLAINTEXT in production.**

Kafka secures the cluster using three layers:

1. **Encryption (TLS/SSL):** Encrypts data flowing between clients and brokers so it can't be read on the network.
2. **Authentication (SASL or mTLS):** Verifies the identity of the client connecting.
3. **Authorization (ACLs):** Checks if the authenticated user has permission to perform the requested action (e.g., "Can `UserA` write to topic `orders`?").

---

## 2. Encryption (TLS / SSL)

Encryption ensures that data cannot be sniffed on the wire. This is identical to how HTTPS works for web traffic.

### Broker Side
The broker must be configured with a keystore containing its SSL certificate, and it exposes an `SSL` or `SASL_SSL` listener.

### .NET Client Side
If your broker uses a certificate signed by a public Certificate Authority (like Azure Event Hubs or Confluent Cloud), you just set the protocol:

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "kafka.production.mycompany.com:9093",
    SecurityProtocol = SecurityProtocol.Ssl // Just encryption, no authentication yet
};
```

If your company uses a **private/internal CA** (very common for self-hosted Kafka), you must give the .NET client the CA certificate so it trusts the broker:

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "kafka.internal.mycompany.com:9093",
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCaLocation = "/path/to/company-internal-ca.pem" // Path to the CA cert
};
```

---

## 3. Authentication (SASL)

Authentication verifies identity. The most common mechanism in Kafka is **SASL** (Simple Authentication and Security Layer).

> ⚠️ **Important:** SASL transmits credentials. You should *always* combine SASL with SSL (`SecurityProtocol.SaslSsl`). If you use `SaslPlaintext`, your passwords are sent in clear text!

### 3.1 SASL/PLAIN
The simplest form. Just a username and password. Used frequently with Azure Event Hubs and Confluent Cloud.

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "kafka.production.mycompany.com:9093",
    SecurityProtocol = SecurityProtocol.SaslSsl,
    
    SaslMechanism = SaslMechanism.Plain,
    SaslUsername = "my-app-user",
    SaslPassword = "SuperSecretPassword123!"
};
```

### 3.2 SASL/SCRAM
SCRAM (Salted Challenge Response Authentication Mechanism) is much safer than PLAIN for self-hosted clusters. It hashes the credentials and supports dynamic credential updates without restarting the broker.

**SCRAM-SHA-256** or **SCRAM-SHA-512** are the standards.

```csharp
var config = new ConsumerConfig
{
    BootstrapServers = "kafka.production.mycompany.com:9093",
    SecurityProtocol = SecurityProtocol.SaslSsl,
    GroupId = "order-processor",
    
    SaslMechanism = SaslMechanism.ScramSha512, // More secure than PLAIN
    SaslUsername = "order-service",
    SaslPassword = "ScramPassword123!"
};
```

### 3.3 Mutual TLS (mTLS)
Instead of usernames and passwords, mTLS uses certificates for authentication. The broker verifies the client's certificate, and the client verifies the broker's certificate.

This is highly secure but operationally complex (you have to manage and rotate client certificates).

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "kafka.production.mycompany.com:9093",
    SecurityProtocol = SecurityProtocol.Ssl, // Note: SSL, not SaslSsl
    
    SslCaLocation = "/certs/ca.pem",                 // Verify the broker
    SslCertificateLocation = "/certs/client.pem",    // Identify the client (public key)
    SslKeyLocation = "/certs/client.key",            // Identify the client (private key)
    SslKeyPassword = "KeyPasswordIfEncrypted"
};
```

---

## 4. Authorization (ACLs)

Once a client is authenticated (e.g., as `User:order-service`), Kafka uses **ACLs** (Access Control Lists) to determine what they can do.

### The Problem
If `order-service` and `inventory-service` both authenticate, how do we stop `inventory-service` from writing to the `orders.placed` topic?

### The Solution: ACLs
Kafka administrators define ACL rules. An ACL looks like this:
`Principal P is [Allowed/Denied] Operation O from Host H on Resource R`

**Example CLI Commands (Run by Ops/Admins):**

```powershell
# Allow 'order-service' to WRITE to the 'orders.placed' topic
kafka-acls.sh --bootstrap-server broker:9092 `
  --command-config admin-client.properties `
  --add `
  --allow-principal User:order-service `
  --operation Write `
  --topic orders.placed

# Allow 'inventory-service' to READ from 'orders.placed'
kafka-acls.sh --bootstrap-server broker:9092 `
  --command-config admin-client.properties `
  --add `
  --allow-principal User:inventory-service `
  --operation Read `
  --topic orders.placed

# Allow 'inventory-service' to use its Consumer Group
kafka-acls.sh --bootstrap-server broker:9092 `
  --command-config admin-client.properties `
  --add `
  --allow-principal User:inventory-service `
  --operation Read `
  --group inventory-processor-group
```

### What happens in .NET if ACLs reject you?
If your `.NET` app tries to produce to a topic it doesn't have `Write` access to, `ProduceAsync` will throw a `ProduceException` with an error code like `TopicAuthorizationFailed`.

If a consumer tries to subscribe to a topic it doesn't have `Read` access to, it won't receive messages and the background thread will log authorization errors.

---

## 5. Security Best Practices for .NET Teams

1. **Never commit secrets:** Never put `SaslPassword` in your code or `appsettings.json`. Use Azure Key Vault, AWS Secrets Manager, or Environment Variables.
2. **Use unique credentials per service:** `order-service` and `inventory-service` should have different usernames and different ACLs. Do not share a global "kafka-admin" account.
3. **Handle Auth Exceptions:** If your credentials rotate or expire, `Confluent.Kafka` will report errors via the `.SetErrorHandler()` callback. Monitor these closely.
4. **Prefer Managed Identity in Azure:** If using Azure Event Hubs, avoid SAS keys entirely and use Azure Active Directory (OAuth) Managed Identities (as covered in File 09).

---

## Summary

You now know:
- ✅ Kafka is completely open by default (`PLAINTEXT`).
- ✅ **TLS/SSL** encrypts data on the network.
- ✅ **SASL/PLAIN** and **SASL/SCRAM** authenticate services via username/password.
- ✅ **mTLS** authenticates via certificates.
- ✅ **ACLs** restrict which topics a service can read, write, or manage.

**Next:** [15-KAFKA-CONNECT.md](./15-KAFKA-CONNECT.md) — Moving data without code using Kafka Connect.
