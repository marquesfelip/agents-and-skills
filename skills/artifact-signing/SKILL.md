---
name: artifact-signing
description: >
  Artifact integrity verification and signing workflows for software supply chain security.
  Use when: artifact signing, code signing, container signing, Cosign, Sigstore, Notation,
  artifact integrity, supply chain security, SLSA provenance, keyless signing, Fulcio, Rekor,
  image signing, binary signing, release artifact signing, signature verification, attestation,
  in-toto attestation, GPG signing, checksum verification, SHA256 digest, tamper detection,
  artifact provenance, trusted build, signed release.
argument-hint: >
  Describe the artifact type and distribution channel (e.g., "Docker image pushed to ECR and
  distributed to Kubernetes clusters, plus Go binary releases on GitHub"), and specify the
  signing tool and trust model requirements.
---

# Artifact Signing Specialist

## When to Use

Invoke this skill when you need to:
- Sign container images, binaries, or release archives in a CI/CD pipeline
- Verify artifact integrity before deployment or installation
- Implement SLSA provenance attestations for supply chain transparency
- Configure keyless signing with Sigstore (no key management overhead)
- Enforce signature verification in deployment pipelines and admission controllers
- Meet supply chain security requirements (SLSA, EO 14028, SSDF)

---

## Step 1 — Understand Signing Models

| Model | Mechanism | Key Management | Trust Anchor |
|---|---|---|---|
| **Keyless (Sigstore)** | Ephemeral key + OIDC identity bound to a transparency log (Rekor) | None — identity is the CI job's OIDC token | Sigstore Fulcio CA + Rekor log |
| **Long-lived key pair** | GPG, RSA, EC key stored in a secret manager | Key rotation required; revocation complex | Public key distribution |
| **Cosign with KMS** | Cosign + AWS KMS / GCP KMS / HashiCorp Vault | Managed by cloud KMS | KMS access policy |
| **Notation (CNCF)** | OCI-native signing with pluggable trust policies | Notation trust store | Trust policy JSON |

**Recommendation:** Use **keyless Sigstore** for GitHub Actions — no key management, identity bound to the workflow, signatures are publicly verifiable against the Rekor transparency log.

Checklist:
- [ ] Signing model chosen based on infrastructure and key management capability
- [ ] Keyless signing used where OIDC token is available (GitHub Actions, GitLab CI)
- [ ] Long-lived keys stored in a KMS — never in CI secrets as raw key material
- [ ] Trust model documented: who can sign, what is verified, where is the trust root

---

## Step 2 — Sign Container Images with Cosign

**Install Cosign:**
```yaml
- uses: sigstore/cosign-installer@v3
```

**Sign an image (keyless — GitHub Actions OIDC):**
```yaml
- name: Sign image with Cosign
  env:
    DIGEST: ${{ steps.build.outputs.digest }}
    IMAGE: ghcr.io/myorg/myapp
  run: |
    cosign sign --yes "${IMAGE}@${DIGEST}"
```

**Sign with KMS (for environments without OIDC):**
```bash
cosign sign \
  --key awskms:///arn:aws:kms:us-east-1:123456789:key/my-key-id \
  myimage@sha256:<digest>
```

**Verify an image signature:**
```bash
# Keyless verification (verifies GitHub Actions identity):
cosign verify \
  --certificate-identity-regexp "https://github.com/myorg/myrepo/.github/workflows/release.yml" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:latest

# KMS key verification:
cosign verify \
  --key awskms:///arn:aws:kms:us-east-1:123456789:key/my-key-id \
  myimage:latest
```

Checklist:
- [ ] Image signed by digest (`@sha256:...`) — not by mutable tag
- [ ] Signature recorded in Rekor transparency log (default for keyless)
- [ ] `--yes` flag used in non-interactive CI — skips confirmation prompt
- [ ] Verification command tested and works before enforcing in deployment

---

## Step 3 — Sign Binary and Archive Releases

**Checksums file (minimum integrity — not authentication):**
```bash
sha256sum myapp-linux-amd64 myapp-darwin-arm64 myapp-windows-amd64.exe > checksums.txt
```

**Sign the checksums file with Cosign (keyless):**
```bash
cosign sign-blob \
  --yes \
  --bundle checksums.txt.bundle \
  checksums.txt
```

**Verify the signed blob:**
```bash
cosign verify-blob \
  --bundle checksums.txt.bundle \
  --certificate-identity-regexp "https://github.com/myorg/myrepo" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  checksums.txt
```

**GPG signing (traditional — for ecosystem compatibility):**
```bash
# Sign with GPG (requires key in CI secret)
gpg --detach-sign --armor myapp-linux-amd64
# Produces: myapp-linux-amd64.asc

# Verify
gpg --verify myapp-linux-amd64.asc myapp-linux-amd64
```

Checklist:
- [ ] Checksums file generated for all release artifacts
- [ ] Checksums file signed (Cosign blob or GPG) — not just the artifact itself
- [ ] Signatures and checksums uploaded alongside artifacts to GitHub Releases
- [ ] Installation documentation includes verification instructions for end users
- [ ] GPG public key published at a stable URL (keyserver or project website)

---

## Step 4 — SLSA Provenance Attestations

SLSA (Supply-chain Levels for Software Artifacts) provenance describes how an artifact was built:

**Generate SLSA provenance with GitHub Actions:**
```yaml
# Use the official SLSA generator for GitHub Actions
- uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2
  with:
    base64-subjects: "${{ steps.hash.outputs.hashes }}"
    upload-assets: true   # Attaches provenance to GitHub Release
```

**Generate provenance for Docker images:**
```yaml
- uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2
  with:
    image: ghcr.io/myorg/myapp
    digest: ${{ steps.build.outputs.digest }}
```

**Verify provenance:**
```bash
slsa-verifier verify-artifact myapp-linux-amd64 \
  --provenance-path myapp-linux-amd64.intoto.jsonl \
  --source-uri github.com/myorg/myrepo \
  --source-tag v1.2.3
```

Checklist:
- [ ] SLSA provenance generated at SLSA Level 3 (isolated, hermetic build)
- [ ] Provenance attached to GitHub Release as `.intoto.jsonl` file
- [ ] Provenance linked to the container image via Cosign attestation
- [ ] SLSA level chosen matches the threat model (L1: basic; L2: hosted build; L3: hardened)

---

## Step 5 — Enforce Signature Verification at Deploy Time

**Kubernetes — Sigstore Policy Controller:**
```yaml
# ClusterImagePolicy: require Cosign signature from the release workflow
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "ghcr.io/myorg/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "https://github.com/myorg/myrepo/.github/workflows/.*"
```

**OPA / Kyverno policy:**
```yaml
# Kyverno ClusterPolicy: block unsigned images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-signature
spec:
  rules:
    - name: verify-image
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - imageReferences: ["ghcr.io/myorg/*"]
          attestors:
            - count: 1
              entries:
                - keyless:
                    subject: "https://github.com/myorg/myrepo/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

**Pre-deploy verification script:**
```bash
#!/bin/bash
IMAGE="ghcr.io/myorg/myapp@${DIGEST}"
cosign verify \
  --certificate-identity-regexp "https://github.com/myorg/myrepo" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  "$IMAGE" || { echo "Image signature verification failed"; exit 1; }
```

Checklist:
- [ ] Signature verification runs before every deployment — not just at build time
- [ ] Kubernetes admission controller (Sigstore Policy Controller or Kyverno) enforces signed images cluster-wide
- [ ] Unsigned images are rejected at the admission layer — cannot be deployed even manually
- [ ] Verification policy version-controlled and applied via GitOps

---

## Step 6 — Key and Identity Management

Checklist:
- [ ] Keyless signing used where possible — eliminates key rotation burden
- [ ] Long-lived keys stored in KMS (AWS KMS, GCP KMS, HashiCorp Vault) — never in CI env vars as raw material
- [ ] Key rotation procedure documented and tested (for KMS-backed keys)
- [ ] Signing identity scoped to the specific workflow and repository — not a shared org-wide identity
- [ ] Rekor transparency log entries retained — they serve as the tamper-evident audit trail

---

## Output Report

### Critical
- No artifact signing — images or binaries can be tampered with between build and deployment
- Signature verification not enforced at deploy time — signed but unverified provides no protection
- Long-lived private key stored as a plain CI secret — key exposure compromises all past and future artifacts

### High
- Images signed by mutable tag instead of digest — tag can be repointed to a different (unsigned) image
- No SLSA provenance — consumers cannot verify the build environment or source commit
- Admission controller not configured — unsigned images can be deployed manually bypassing CI

### Medium
- GPG public key not published at a stable URL — end users cannot verify downloaded binaries
- Verification command not documented — security benefit is theoretical, not actionable
- Signing step runs after image push — a window exists where unsigned image is in the registry

### Low
- Checksums file present but not signed — integrity without authentication
- Rekor transparency log not consulted during verification — offline verification only
- SLSA provenance generated but not verified in deployment pipeline

### Passed
- Container images signed with Cosign (keyless) by digest on every build
- SLSA provenance generated and attached to all releases
- Signature verification enforced by Kubernetes admission controller before every deploy
- Binary releases include signed checksums file and verification instructions
- Signing identity scoped to the specific workflow — not a shared credential
