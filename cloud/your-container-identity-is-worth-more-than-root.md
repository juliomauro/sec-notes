# Your Container's Identity Is Worth More Than Root

> **Category:** Cloud · IAM · Containers  
> **Level:** Practitioner  

---

There is a persistent mental model in infrastructure security that frames privilege as a binary: you either have root or you don't. In cloud environments, this model is dangerously incomplete.

A container running as a non-root user, with no shell, no tools, and limited file read access, can still be the starting point for a full cross-account compromise — if the workload identity attached to it has the wrong permissions.

## The Metadata Endpoint

Every major cloud provider exposes a metadata service at a link-local address (typically `169.254.169.254`). This endpoint is reachable from any process running on the host or inside a container, unless explicitly blocked. It serves:

- Instance identity documents
- Temporary IAM credentials
- User data (often contains bootstrap secrets)

Querying it requires nothing more than an HTTP request:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Returns: my-app-role

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/my-app-role
# Returns: AccessKeyId, SecretAccessKey, Token, Expiration
```

If a vulnerability in the application allows any form of outbound HTTP request or file read, this endpoint is reachable. Server-side request forgery (SSRF), path traversal, dependency vulnerabilities — any of these can be the entry point.

## What Happens After

The credentials returned are temporary but fully functional AWS credentials tied to the IAM role attached to the workload. Their blast radius depends entirely on what that role is allowed to do.

In a real exercise, a role that could not list all buckets, could not create instances, and had no obvious administrative access still allowed:

- Listing images in a specific ECR repository
- Reading objects from a specific S3 bucket

That was enough. An old container image in the registry had a `.env` file baked into one of its layers. The `.env` had credentials for another service. That service had a cross-account trust relationship. The chain ended in a production account.

## The Layer History Problem

Container images are built in layers. Each `RUN`, `COPY`, and `ADD` instruction creates a new layer. When a secret is added in one layer and deleted in a later layer, it is **not removed from the image** — it is only hidden from the final filesystem view. The layer containing the secret still exists in the image manifest and can be extracted:

```bash
docker save my-image | tar -x
# Each layer is a tar archive — extract and inspect
find . -name "*.env" -o -name "*.key" -o -name "credentials"
```

Tools like `trufflehog`, `dive`, and `trivy` can scan image history automatically.

## Blocking the Path

**Restrict metadata access:**

For AWS, enforce IMDSv2 (token-required mode) and set hop limit to 1 on the instance. This prevents containers from reaching the metadata endpoint in most configurations:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxx \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

For containers that do not need instance metadata at all, block the endpoint via network policy or iptables.

**Apply least-privilege to workload roles:**

The IAM role attached to a container should have exactly the permissions the application needs — nothing more. If the app reads from one S3 bucket, the role policy should name that exact bucket and only allow `s3:GetObject`.

**Scan image layers before pushing:**

Make image scanning part of the CI pipeline. Any image with secrets in its history should fail the build, not just be flagged.

## The Core Insight

In cloud, the workload identity is often more valuable than root on the host. A container running as uid 1000 with no sudo and a read-only filesystem can still be the pivot point into a production account if the IAM role attached to it has been granted permissions that were never properly reviewed.

The question to ask about every workload is not "what user does this container run as?" but "what can the identity attached to this workload do — and what can it reach?"

---

*Related: [Case 02 — Container Metadata Abuse and IAM Escalation](https://github.com/juliomauro/sec-writeups/tree/main/cases/02-container-metadata)*
