---
layout: single
title: "AWS KMS XKS Without an HSM: Threshold Cryptography Across Cloud Providers"
date: 2026-02-17
categories: [blogging]
---

In a [previous post](/blogging/2026/02/11/AWS_KMS_XKS_SoftHSM.html), I backed AWS KMS with a SoftHSM on my local machine via an SSH tunnel, based on the [AWS XKS proxy sample](https://github.com/aws-samples/aws-kms-xks-proxy). It proved the concept: the encryption key lives on hardware I control, and AWS KMS calls my proxy for every cryptographic operation. 

This post takes a different approach. Instead of storing the AES key in one place and guarding it, I split the key so it never exists in any single location. Two independent services on two different cloud providers each hold a share. They cooperate to perform encryption, but neither can derive the key alone. No HSM required.

## The Problem with a Single Key in a Single Place

HSMs are expensive, complex to operate, and tied to a specific vendor and facility. If you want data sovereignty guarantees across European jurisdictions, you need HSMs in those jurisdictions, and you need to double the hardware for resilience, with all the procurement and compliance overhead that entails.

There is a way to make it simpler.

## Split Key Across Cloud Providers

The idea is 2-of-3 threshold cryptography over elliptic curves (P-256). Three key shares are generated in a one-time ceremony. Any two shares can derive the AES-256 encryption key for a given key identifier. One share alone is mathematically useless.

For normal operation, two shares participate:

- **Share 1** lives in a Scaleway Serverless Container (fr-par, Paris). This is the XKS proxy that AWS KMS calls. Scaleway provides the HTTPS endpoint; the container runs plain HTTP behind Scaleway's TLS termination.
- **Share 2** lives on an Exoscale compute instance (ch-gva-2, Geneva). A minimal service that computes one elliptic curve operation per request, protected by mutual TLS.
- **Share 3** is offline backup. Stored securely, never deployed. Used only if one of the other two services is permanently lost.

When AWS KMS needs to encrypt or decrypt, the XKS proxy computes its part locally, asks the EU service for the second part over mTLS, combines the two parts mathematically, and derives the AES-256 key. The key exists in memory for the duration of one request, then is zeroed.

The critical property: the private key is never reconstructed, not even transiently. Each party computes its own partial result on its own share. The proxy combines these partial results using Lagrange interpolation on the elliptic curve. The math guarantees this produces the same shared secret as if the full private key had been used, but no single party ever held that full key.

```
AWS KMS
  |
  |  HTTPS + SigV4
  v
Scaleway (fr-par)                      Exoscale (ch-gva-2)
  XKS Proxy — Holds: Share 1            EU Share Service — Holds: Share 2
  |                                      |
  | 1. V = hash_to_curve(keyId)          |
  | 2. partial_1 = share_1 * V           |
  |                                      |
  |------------ mTLS -----------------> |
  |    "compute share_2 * V"             | 3. partial_2 = share_2 * V
  | <----------------------------------- |
  |                                      |
  | 4. Lagrange combine partials         |
  | 5. HKDF -> AES-256 key              |
  | 6. AES-GCM encrypt/decrypt          |
  | 7. Zeroize key                       |
  |
  v
AWS KMS (response)
```

The virtual point `V` is derived deterministically from the key identifier using hash-to-curve (RFC 9380). This means the same `keyId` always produces the same AES-256 key -- a requirement for KMS, where encrypt and decrypt must use the same key.

## What Each Party Sees

| Party | Knows | Cannot derive |
|-------|-------|--------------|
| XKS Proxy — Scaleway (Share 1) | Its share, the virtual point, its own partial result | The AES key without Share 2's partial |
| EU Share Service — Exoscale (Share 2) | Its share, the virtual point, its own partial result | The AES key, the plaintext, the ciphertext, anything about the AWS request |
| AWS KMS | Ciphertext, AAD | The AES key (no shares, no partials) |
| Attacker with 1 share | One share scalar | The AES key (need 2 of 3) |

The Exoscale service is deliberately minimal. It receives a point, multiplies it by its share scalar, and returns the result. It never sees plaintext, ciphertext, additional authenticated data, or the derived AES key. It doesn't even know whether the request is for an encrypt or decrypt operation.

## What This Protects and What It Doesn't

Like the previous HSM-based approach, this is about **data at rest**. When you upload a file to S3 with KMS encryption, the file is encrypted before it hits storage. When you download it, S3 asks KMS to decrypt, KMS asks my proxy, my proxy cooperates with Exoscale, and the plaintext is returned through the chain. At each step of that chain, the data passes through systems in the clear -- AWS sees the plaintext when it serves the GET request. There is no way around this when using server-side encryption.

The protection is against someone gaining access to the stored data without going through the live system. If S3 storage is exfiltrated, the encrypted objects are useless without the key. And the key cannot be derived without the cooperation of two independent services.

### The monitoring advantage

This is where split keys differ meaningfully from a single HSM. Every decryption operation requires the Exoscale service to participate. Every request is logged: which key path was used, when, and from which client. If someone -- including the operator of the XKS proxy -- tries to mass-decrypt stored data, the Exoscale service sees a sudden flood of requests. This is visible in Exoscale's logs, independently of anything happening on the AWS or Scaleway side.

Here are the actual logs from the deployed system. First, the Scaleway XKS proxy starting up and handling requests:

```
# Scaleway XKS Proxy (fr-par)
18:07:01  Share store initialized, share_id=1, key_count=5
18:07:01  EU client mTLS configured (client cert + custom CA)
18:07:01  EU client initialized, base_url=https://92.39.60.218:8443
18:07:01  XKS Proxy listening, addr=0.0.0.0:8080
18:20:57  Encrypt   kmsRequestId=f6be4217-...  key_id=test-key-1
18:21:06  Decrypt   kmsRequestId=b836a273-...  key_id=test-key-1
18:34:23  Decrypt   kmsRequestId=544d2f24-...  key_id=test-key-1
18:34:26  Decrypt   kmsRequestId=45b3ecaa-...  key_id=test-key-1
18:35:34  Decrypt   kmsRequestId=1ab54f6b-...  key_id=test-key-1
```

And the Exoscale EU share service, which sees only partial ECDH requests — no plaintext, no ciphertext, no indication of whether it's an encrypt or decrypt:

```
# Exoscale EU Share Service (ch-gva-2)
17:02:54  Share store initialized, share_id=2, key_count=5
17:02:54  EU Share Service listening (mTLS), addr=0.0.0.0:8443
17:20:57  Partial ECDH request  request_id=6a7621ce772b8015  key_path=encryption/key-1
17:21:06  Partial ECDH request  request_id=1d5d05530cdc70a4  key_path=encryption/key-1
17:34:23  Partial ECDH request  request_id=8defb91565227a92  key_path=encryption/key-1
17:34:26  Partial ECDH request  request_id=ed1cf2052a008935  key_path=encryption/key-1
17:35:34  Partial ECDH request  request_id=22c75b772d54ffaa  key_path=encryption/key-1
```

Every operation leaves a trace on both providers independently. The KMS request UUID from AWS flows through to the Scaleway proxy logs. The Exoscale service logs the key derivation path but has no visibility into what the operation is for.

More importantly, the Exoscale operator can act on anomalies. If they see thousands of partial computation requests in a few minutes for key paths that normally see a handful per hour, they can shut down the instance. The XKS proxy still holds its share, but one share alone is useless -- mathematically, it reveals nothing about the key.

This is a property that no single-HSM setup provides. When the key is in one place, whoever controls that place can silently decrypt everything. With split keys, any large-scale decryption is necessarily visible to the other party, and either party can pull the plug.

## Performance: The 250ms Budget

AWS KMS requires the XKS proxy to respond within 250ms. The threshold protocol adds a network round-trip to a second cloud provider in a different country. Can it stay within budget?

The XKS proxy runs on Scaleway Serverless Containers (fr-par, Paris). The EU share service runs on Exoscale (ch-gva-2, Geneva). Paris to Geneva is ~340km, with network round-trips typically 5-15ms.

### Warm requests

With connection reuse (the typical case after the first request), the round-trip to Geneva is fast. The elliptic curve operations take about 1ms on each side. The bottleneck is network latency, not computation. All operations stay well within the 250ms budget.

### Cold start

The Scaleway container starts in under a second. KMS periodic health checks keep it warm. Since both the XKS proxy and the EU share service are static Rust binaries in distroless Docker images, startup is fast. The XKS proxy initializes its share store, configures the mTLS client to Exoscale, and begins listening -- all in under a second.

### Exoscale service

The EU share service runs on an Exoscale compute instance with rustls handling mTLS directly (no TLS termination proxy). It performs a single scalar-point multiplication per request (~1ms of computation). The mTLS connection ensures only the XKS proxy -- holding the correct client certificate signed by the pinned CA -- can reach it.

## The Sovereignty Argument

The real value isn't performance -- it's the trust model.

With a traditional HSM-backed XKS, the key is in one place. Whoever controls that place controls the key. With threshold cryptography, the key is split across providers:

- **No single cloud provider** can derive the key. Scaleway holds one share but needs Exoscale's cooperation. Exoscale holds one share but never sees the key or the data. AWS holds no shares at all.
- **Revocation is instant.** Shut down the Exoscale instance, and AWS KMS can no longer encrypt or decrypt. The kill switch is a single command to a Swiss provider that has no relationship with AWS or Scaleway.
- **No HSM vendor lock-in.** The system is pure software. The shares are 34-byte values. They can be deployed to any compute platform that runs a Rust binary.

## Three Providers, Three Countries

The current deployment already spans two providers and two countries: Scaleway (Paris, France) for the XKS proxy and Exoscale (Geneva, Switzerland) for the EU share service. But the 2-of-3 threshold scheme is designed for three:

| Share | Provider | Location | Jurisdiction | Status |
|-------|----------|----------|-------------|--------|
| Share 1 | Scaleway Serverless Containers | fr-par (Paris) | France | Active — XKS proxy |
| Share 2 | Exoscale Compute | ch-gva-2 (Geneva) | Switzerland | Active — EU share service |
| Share 3 | IONOS or Hetzner | Frankfurt | Germany | Cold standby |

Any two of the three can derive the key. This gives you:

- **Provider resilience.** If one provider has an outage, the other two can still operate. Reconfigure the XKS proxy to call the remaining provider. Since the Lagrange interpolation is parameterized by share IDs, switching from Share 2 to Share 3 is a configuration change, not a key change. The derived AES key is identical regardless of which two shares are used.
- **Jurisdictional resilience.** If one country's regulator orders a provider to freeze or hand over data, they get one share -- which is mathematically useless without a second share from a different jurisdiction. The other two providers can continue operating.
- **No single point of legal compulsion.** A French court order to Scaleway yields Share 1. A Swiss court order to Exoscale yields Share 2. Neither alone produces the key. Compelling two jurisdictions simultaneously requires international cooperation -- a significantly higher bar than a single domestic order.
- **Mutual oversight.** Each share service logs every request. Unusual decryption patterns are visible to at least two independent operators in two different countries. A government quietly compelling mass decryption through one provider is immediately visible to the other.

### Cross-border latency

Paris to Geneva is ~340km. Paris to Frankfurt is ~480km. Network round-trips between these cities are typically 5-15ms.

The current Paris-Geneva deployment already works within the 250ms budget. Adding a third share in Frankfurt would not change the latency significantly -- the XKS proxy only calls one partner per request, and Paris to Frankfurt is a comparable distance.

### Swiss neutrality

Switzerland is particularly interesting as a share location. It's not an EU member state, so it's outside the reach of EU-wide data access orders. Swiss data protection law (nDSG/LPD) is independently strong. Providers like Infomaniak and Exoscale specifically market sovereignty and data localization. A share stored with a Swiss provider adds jurisdictional diversity that goes beyond having multiple EU locations.

### The operational model

In normal operation, only two shares are active. The third is cold standby. If the XKS proxy needs to switch to a different partner:

1. Activate the standby share service (deploy the container to the standby provider)
2. Update the XKS proxy's configuration: new URL and the new share ID
3. The Lagrange coefficients adjust automatically based on the share IDs involved
4. The derived AES key is identical -- the math guarantees it

## Cost

The entire system runs on near-free infrastructure:

| Component | Cost |
|-----------|------|
| Scaleway Serverless Container (XKS proxy) | Free tier (400k GB-s/month) |
| Exoscale Compute (EU share service) | ~$15/month (standard.small) |
| **Total** | **~$15/month** |

Scaleway provides the HTTPS endpoint for free with its serverless containers. The Exoscale VM is the only fixed cost.

Compare this to a Thales Luna HSM ($15-25k/year) or a cloud-managed HSM billed per key per hour. The threshold approach costs almost nothing at low to moderate volumes.

## What This Does Not Replace

This is not FIPS 140-3 Level 3. There is no tamper-resistant hardware. If you need a compliance checkbox that says "FIPS", you need an HSM.

And as with any server-side encryption scheme, the data passes through AWS in the clear during normal operations. AWS could retain plaintext, log decrypted data, or be compelled to do so. This is inherent to the model: you're asking AWS to encrypt and decrypt on your behalf.

What threshold cryptography adds is control over the key at rest and visibility into its use. The key cannot be derived without active cooperation from two independent parties on two different cloud providers in two different countries. Unusual access patterns are visible to both. And either party can revoke access instantly by shutting down their service.

For the actual security properties most people want from external key management -- control over key material, revocation capability, multi-jurisdiction sovereignty, defense against silent key extraction -- threshold cryptography provides stronger guarantees than a single HSM. A compromised HSM exposes all its keys. A compromised share exposes nothing.

## Regulatory Landscape

The threshold approach maps to several EU regulations, existing and upcoming.

### DORA (Digital Operational Resilience Act)

DORA Article 7 (RTS) requires key lifecycle management, controls against loss and unauthorized access proportional to risk, key replacement procedures, and certificate registers. The threshold approach addresses each:

- **Loss protection** -- 2-of-3 means losing one share doesn't lose the key. Offline backup share provides recovery.
- **Unauthorized access** -- no single party holds the full key. Compromising one provider reveals nothing mathematically useful.
- **Key replacement** -- re-run the ceremony, generate new shares, redeploy.
- **Third-party concentration risk** (DORA Art. 28+) -- shares on separate providers in separate jurisdictions directly addresses this. DORA specifically calls out concentration risk with single ICT providers.
- **Auditability** -- every operation logged independently on each provider.
- **Revocability** -- either party can shut down their service instantly.

### NIS2 Directive

Article 21.2(h) requires encryption and key management proportional to risk for essential and important entities (energy, healthcare, transport, digital infrastructure). AES-256 meets the minimum. Splitting the key across jurisdictions is a stronger control than storing it in one place.

### GDPR

Article 32 requires "appropriate technical measures" including encryption. Article 34 says if data is encrypted and the key is not exposed, breach notification to individuals may not be required. With threshold splitting, the cloud provider never holds the key -- they are a processor with no access to key material. If S3 storage is breached, the encrypted objects are useless without cooperation of two independent parties.

### EU Data Act

Article 28 (applicable since September 2025) requires cloud providers to take "all reasonable measures, including encryption" to prevent unlawful access to data, especially from third-country government requests. This is where threshold splitting has its strongest argument.

The US CLOUD Act allows US law enforcement to compel American companies to hand over data stored abroad. But the CLOUD Act is encryption-neutral -- it does not compel providers to decrypt data they cannot decrypt. A subpoena to AWS is mathematically useless.

### EUCS (European Cybersecurity Certification Scheme for Cloud Services)

Still being finalized by ENISA. Earlier drafts included a "high+" level requiring encryption keys to be held outside the cloud provider's control and within EU jurisdiction. Even though sovereignty requirements were dropped from the latest draft, individual member states can still impose them. The threshold approach is ready for the strictest interpretation: keys are split across European providers, none held by a US-subject entity.

---
