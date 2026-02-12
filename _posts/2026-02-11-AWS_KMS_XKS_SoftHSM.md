---
layout: single
title: "AWS KMS External Key Store: Keep Your Encryption Keys Out of the Cloud"
date: 2026-02-11
categories: [blogging]
---

A proof of concept demonstrating AWS KMS encryption backed by a local SoftHSM key, using the [XKS Proxy API](https://docs.aws.amazon.com/kms/latest/developerguide/keystore-external.html). The cryptographic master key lives on a machine I fully own, not in AWS, not in a managed service, but on my personal hardware.

## Motivation

I wanted to back AWS KMS with a hardware security module under my physical control. SoftHSM on a local machine serves as a stand-in for a real HSM for this POC. The EC2 instance and SSH reverse tunnel exist only because AWS KMS requires an HTTPS endpoint with a valid TLS certificate to reach the XKS proxy. The actual key material never leaves the local machine.

---

## Architecture

When you upload a file to S3 with `--sse aws:kms`, KMS calls the xks-proxy's `/encrypt` endpoint. The proxy performs AES-256-GCM encryption via PKCS#11, remoted to SoftHSM on your machine through p11-kit over an SSH tunnel. Decryption follows the reverse path.

```
S3 (SSE-KMS) → KMS → XKS Proxy (EC2) → p11-kit client
                                              ↓
                                     SSH reverse tunnel
                                              ↓
                                     p11-kit server (Mac) → SoftHSM
```

| Component | Location | Purpose |
|-----------|----------|---------|
| xks-proxy | EC2 (aarch64, t4g.nano) | [AWS reference XKS proxy](https://github.com/aws-samples/aws-kms-xks-proxy) (Rust) |
| ALB | AWS | TLS termination with valid certificate |
| p11-kit 0.26.2 | EC2 + Mac | PKCS#11 remoting over Unix sockets |
| SoftHSM v2 | Mac (Homebrew) | Software HSM holding the AES-256 key |

---

## Setup

### Create the SoftHSM AES-256 key

```bash
softhsm2-util --init-token --slot 0 --label foo --pin 1234 --so-pin 0000
softhsm2-util --show-slots

pkcs11-tool --module /opt/homebrew/lib/softhsm/libsofthsm2.so \
  --login --pin 1234 \
  --keygen --key-type AES:32 --label foo \
  --token-label foo
```

Expected output:
```
Secret Key Object; AES length 32
  label:      foo
  Usage:      encrypt, decrypt, sign, verify, wrap, unwrap
  Access:     never extractable, local
```

### Deploy the infrastructure

A single `create.sh` script handles everything: cross-compiling xks-proxy for aarch64 via `cargo zigbuild`, deploying CloudFormation (ALB, EC2, Route53, CloudWatch), SCPing the binary and config to EC2, and starting the systemd service.

### The p11-kit version issue

This is where I spent most of my time. The AL2023 repo ships p11-kit **0.24.1**, which lacks AES-GCM RPC serialization entirely, every encrypt/decrypt call fails with `CKR_MECHANISM_INVALID`.

| Version | AES-GCM RPC | Object Handles | Status |
|---------|-------------|----------------|--------|
| 0.24.1 (AL2023 repo) | No | N/A | `CKR_MECHANISM_INVALID` |
| 0.25.x | Yes | Broken across sessions | `CKR_OBJECT_HANDLE_INVALID` |
| **0.26.2** | **Yes** | **Works** | **Working** |

Version 0.26.2 must be built from source on EC2. And it must match on both ends -- a version mismatch between client (EC2) and server (Mac) causes RPC protocol errors.

```bash
# On EC2 (t4g.nano, 512MB RAM)
sudo dnf install -y meson ninja-build gcc libtasn1-devel libffi-devel
curl -sL https://github.com/p11-glue/p11-kit/releases/download/0.26.2/p11-kit-0.26.2.tar.xz | tar xJ
cd p11-kit-0.26.2
meson setup _build --prefix=/usr --libdir=/usr/lib64
ninja -C _build -j1    # -j1 required: t4g.nano OOMs on parallel builds
sudo ninja -C _build install
```

### Start the PKCS#11 tunnel

Two terminals on the Mac:

**Terminal 1 -- p11-kit server:**
```bash
p11-kit server --provider /opt/homebrew/lib/softhsm/libsofthsm2.so "pkcs11:"
```

**Terminal 2 -- SSH reverse tunnel:**
```bash
export P11_KIT_SERVER_ADDRESS=unix:path=/var/folders/.../pkcs11-XXXX

ssh -i ~/Downloads/EC2Tutorial2.pem \
  -R /home/ec2-user/.p11-kit.sock:${P11_KIT_SERVER_ADDRESS#unix:path=} \
  ec2-user@<EC2_IP>
```

This forwards the EC2 Unix socket to the Mac's p11-kit server socket. From EC2's perspective, PKCS#11 operations on `/home/ec2-user/.p11-kit.sock` transparently reach SoftHSM on the Mac.

### Test end-to-end

```bash
# Health check
curl https://xks.lemaire.tel/ping

# Upload with XKS encryption
echo "hello from softhsm" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://xks-proxy-poc-test/test.txt \
  --region eu-west-3 \
  --sse aws:kms \
  --sse-kms-key-id cd0608a9-0726-4187-b1b1-d0b08370d8f9

# Download (decryption goes through SoftHSM)
aws s3 cp s3://xks-proxy-poc-test/test.txt /tmp/downloaded.txt --region eu-west-3
cat /tmp/downloaded.txt
# hello from softhsm
```

---

## A Rust Undefined Behavior Bug in the XKS Proxy

The interesting problem I hit wasn't infrastructure, it was a compiler optimization bug in the AWS reference implementation.

The `GetKeyMetadata` handler declares stack variables as immutable (`let key_type = 0;`) and passes pointers to them via `set_ck_ulong()` for `C_GetAttributeValue` to write into. The C function writes the correct values (e.g., `key_type=31` for CKK_AES), but the Rust compiler's release-mode optimizer treats the immutable bindings as compile-time constants and inlines `0` wherever they're subsequently read.

Impact: `keyspec(0, 0)` = `"RSA_0"` instead of `keyspec(31, 32)` = `"AES_256"`. KMS rejected the key metadata.

This only manifests in release builds. Debug builds work fine because the optimizer doesn't inline the constants.

**To fix:** Force the compiler to re-read from actual memory after the C call:

```rust
let key_type = unsafe { std::ptr::read_volatile(&key_type) };
let key_size = unsafe { std::ptr::read_volatile(&key_size) };
```

The root cause is Rust undefined behavior: writing through a raw pointer derived from an immutable reference. The proper long-term fix would be `UnsafeCell` or `MaybeUninit` in the `rust-pkcs11` crate's `CK_ATTRIBUTE::set_ck_ulong` implementation.

---

## What XKS Does and Does Not Protect

### XKS protects against

- **Future unauthorized access by AWS.** Disconnect the tunnel, shut down the proxy, and AWS cannot decrypt data going forward.
- **Regulatory and sovereignty requirements.** Cryptographic master keys remain under your control, with an audit trail of all key operations.
- **Cloud provider key management concerns.** The master key material stays entirely outside AWS.

### XKS does not protect against

- **AWS actively retaining or exfiltrating data.** They could copy plaintext before encryption or retain data encryption keys.
- **Legal compulsion.** If DEKs were retained in AWS systems, AWS could be compelled to produce them.
- **Retrospective decryption.** If AWS legitimately decrypted an object to serve a GET request, they could have retained the plaintext.

### Bottom line

External key management is about **governance**, **operational control**, and **trust reduction**. It is not about protecting against a malicious provider -- they see plaintext anyway. If the cloud provider itself is your threat model, use client-side encryption before uploading, or don't use cloud.

---

## Cloud Provider Comparison

### AWS -- External Key Store (XKS)

**DIY implementation: YES.** AWS publishes the [XKS Proxy API specification](https://github.com/aws/aws-kms-xksproxy-api-spec) and a reference implementation. You can build your own proxy backed by any PKCS#11-compatible HSM or key manager.

### GCP -- Cloud External Key Manager (EKM)

**DIY implementation: NO.** No public API specification, no reference implementation. You must use certified partner solutions (Thales CipherTrust, Fortanix DSM).

### Azure -- No external key store equivalent

**DIY implementation: N/A.** Azure offers BYOK (keys end up in Azure), Managed HSM (single-tenant, still in Azure), and Dedicated HSM (Thales Luna in Azure). No equivalent to XKS or EKM where keys remain outside the cloud.

---

## Source Code

Available on GitHub: [aws-kms-xks-poc](https://github.com/jlemaire/aws-kms-xks-poc)

```
aws-kms-xks-poc/
  cloudformation.yaml          # EC2 + ALB + Route53 + CloudWatch
  configuration/settings.toml  # xks-proxy config
  create.sh                    # One-shot deploy script
```
