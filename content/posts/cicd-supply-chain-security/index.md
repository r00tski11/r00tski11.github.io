---
title: "Building a CI/CD Supply-Chain Integrity System"
date: 2026-03-30
description: "How I built a build artifact signing and verification system using OpenSSL, named pipes, and Python."
tags: ["security", "supply-chain", "ci-cd", "python", "cryptography"]
categories: ["cloud"]
toc: true
---

I was working on a project where there was no way to confirm that the build artifact running in production was the same one that came out of CI. Between build, staging, and prod, there were multiple handoff points. Any of them could've been tampered with. Nobody was checking.

This isn't a theoretical problem. SolarWinds happened because attackers injected malicious code into the build process. Codecov got hit when someone modified their bash uploader script and harvested credentials for months. 3CX was the same thing. Supply chain, build pipeline, compromised.

We needed something simple. Sign the build after CI produces it, verify the signature before deploying it. If the signature doesn't match, block the deploy.

## The problem

Most CI/CD setups look like this:

```
CI builds artifact → artifact goes to staging → artifact goes to prod
```

At each step, the artifact could be modified. Corrupted during transfer, swapped out by an attacker, or just accidentally overwritten. Without cryptographic verification, you're basically trusting that nothing went wrong.

Frameworks like [SLSA](https://slsa.dev/) exist for this, but they're heavy. We didn't need a full provenance system. We needed to answer one question: is this artifact the same one we built?

## How it works

Two Python services running as daemons:

1. **Signer**: takes a build artifact path, generates a SHA-256 digest, signs it with the private key, outputs a `.sign` file
2. **Verifier**: takes a build artifact path, finds the corresponding `.sign` file, verifies the signature against the public key

They communicate through named pipes (FIFOs). CI writes a file path to `/tmp/sign_build`, the signer picks it up. Before deploy, the path goes to `/tmp/verify_build`, the verifier checks it.

```
CI build output
    |
    v
[/tmp/sign_build]  -->  Signer  -->  artifact.sign
    |
    v
[/tmp/verify_build]  -->  Verifier  -->  VERIFIED / FAILED
    |
    v
Deploy (only if VERIFIED)
```

Both services log everything to dated log files for audit trails.

## The signing service

Here's the core of the signing service, stripped down:

```python
import os
import subprocess
import logging

PIPE_PATH = '/tmp/sign_build'
CERT_NAME = 'signing_cert.pem'

def sign_build(build_file, cert_name, cert_passphrase):
    signed_file = f'{os.path.splitext(build_file)[0]}.sign'
    result = subprocess.run(
        ['openssl', 'dgst', '-sha256', '-sign', cert_name,
         '-passin', f'pass:{cert_passphrase}',
         '-out', signed_file, build_file],
        stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
    )

    if result.returncode != 0:
        logging.error(f'Signing failed: {result.stderr.strip()}')
        return {'status': 400, 'message': 'FAILED',
                'filename': os.path.basename(build_file)}

    logging.info(f'Signed: {build_file}')
    return {'status': 200, 'message': 'SIGNED',
            'filename': os.path.basename(build_file)}
```

What's happening:

- `openssl dgst -sha256 -sign` computes a SHA-256 hash of the build file and signs it with the RSA private key from the PEM certificate
- Output is a `.sign` file (256 bytes for a 2048-bit RSA key)
- The service returns a JSON response. `200` for success, `400` for failure

The main loop reads from the named pipe and calls `sign_build` for each path it gets:

```python
def main():
    if not os.path.exists(PIPE_PATH):
        os.mkfifo(PIPE_PATH)

    with open(PIPE_PATH, 'r') as pipe:
        while True:
            build_path = pipe.readline().strip()
            if build_path:
                result = sign_build(build_path, CERT_NAME, CERT_PASSPHRASE)
                logging.info(f'Result: {result}')
```

On startup, it also extracts the public key from the certificate so the verifier can use it:

```python
def create_pub_key(cert_filename, pub_key_filename):
    with open(pub_key_filename, 'wb') as pub_key_file:
        subprocess.run(
            ['openssl', 'x509', '-pubkey', '-noout', '-in', cert_filename],
            stdout=pub_key_file, stderr=subprocess.PIPE
        )
```

## The verification service

The verifier is simpler. It takes a build file path, looks for the matching `.sign` file, and runs the OpenSSL verify command:

```python
import os
import subprocess
import logging
import json

PIPE_PATH = '/tmp/verify_build'
PUBLIC_KEY = 'public_key.pem'

def build_verification(build_file, public_key):
    signed_file = f'{os.path.splitext(build_file)[0]}.sign'
    result = subprocess.run(
        ['openssl', 'dgst', '-sha256', '-verify', public_key,
         '-signature', signed_file, build_file],
        stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
    )

    if result.returncode == 0:
        logging.info(f'Verified: {build_file}')
        return {'status': 200, 'message': 'VERIFIED',
                'filename': os.path.basename(build_file)}
    else:
        logging.error(f'Verification failed: {build_file}')
        return {'status': 400, 'message': 'FAILED',
                'filename': os.path.basename(build_file)}
```

Same pattern. Reads paths from a named pipe, verifies each one, returns JSON. The deploy script only proceeds if it gets a `200`.

One edge case we ran into: the `.sign` file wasn't always there when the verifier tried to read it. The signer hadn't finished writing it yet. Fixed it by having the verifier check if the file exists before running verification, and logging an error if it's missing.

## Design decisions

We went with **named pipes** for decoupling. The signer doesn't need to know who's producing builds, and the verifier doesn't care who's consuming results. Any process can write a path to the pipe. We could swap out the entire CI system without touching the signing infrastructure.

**Separate services** because of key isolation. The signer has the private key. The verifier only has the public key. If the verifier gets compromised, the attacker still can't forge signatures.

**JSON responses** so the deploy scripts can parse the result and decide whether to proceed. Also makes it easy to plug into monitoring or alerting.

We used **OpenSSL CLI** instead of a Python library because it was already installed everywhere. No extra dependencies to manage. The `cryptography` library would've been cleaner code, but OpenSSL got us running faster.

Every sign and verify operation gets **logged** with a timestamp to dated log files. If something goes wrong, you have a full audit trail of what was signed, when, and whether verification passed.

## What I'd change now

This was built to solve an immediate problem, and it worked. But looking back, there are things I'd do differently:

**Ed25519 instead of RSA.** Ed25519 keys are 32 bytes vs 256+ for RSA. Signatures are faster to generate and verify. No padding scheme to worry about (RSA has had multiple padding-related vulnerabilities over the years). If I was building this today, I'd use Ed25519 through Python's `cryptography` library instead of shelling out to OpenSSL.

**Key management.** The passphrase shouldn't live in the source code or in an environment variable. In production, it should come from a secrets manager like HashiCorp Vault or AWS Secrets Manager. An HSM would be ideal but that's expensive for most teams.

**Key rotation.** We didn't build key rotation in from the start, and adding it later was painful. Design for it from day one. Have the verifier accept multiple public keys during a rotation window.

**Sigstore instead of custom.** If I was starting from scratch, I'd look at [Sigstore](https://www.sigstore.dev/) and [Rekor](https://docs.sigstore.dev/logging/overview/) for transparency logging. It solves the same problem with better tooling and a public audit trail.

**Structured signature metadata.** Right now the `.sign` file is just the raw signature. It should include the build ID, pipeline run ID, timestamp, and environment so you can trace exactly where and when a build was signed. Store that in a database for query-ability.

## If you're building something similar

- Start with signing only, don't enforce verification right away. Run both in parallel for a while. Sign everything, verify everything, but don't block deploys yet. Once you trust it, flip the switch.
- Keep the signer and verifier as separate services. Don't combine them. Key isolation matters.
- Log everything. Signing events, verification events, failures. You'll need this for incident response.
- Use Ed25519 if you're starting new. RSA works but Ed25519 is simpler and faster.
- Read the [SLSA specification](https://slsa.dev/spec/v1.0/) even if you don't implement the full framework. The threat model is useful.
- [in-toto](https://in-toto.io/) is worth looking at for full supply-chain attestation if you need more than just artifact signing.

## References

- [SLSA Framework](https://slsa.dev/): Supply-chain Levels for Software Artifacts
- [Sigstore](https://www.sigstore.dev/): keyless signing for software artifacts
- [in-toto](https://in-toto.io/): supply-chain integrity framework
- [OpenSSL dgst documentation](https://www.openssl.org/docs/man3.0/man1/openssl-dgst.html)
