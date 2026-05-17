# AWS Static Website Hosting — Portfolio Site (S3 + CloudFront + KMS + IAM + CloudTrail)

A production-grade, globally distributed deployment of the [portfolio-site](https://github.com/ibtisam-iq/portfolio-site) on AWS — served securely from a private S3 bucket through CloudFront with Origin Access Control, KMS encryption at rest, Cross-Region Replication for disaster recovery, and full audit logging via CloudTrail.

> This is **Step 1A** of the overall deployment journey documented in the [portfolio-site](https://github.com/ibtisam-iq/portfolio-site) repository. The site made live at **[portfolio.ibtisam-iq.com](https://portfolio.ibtisam-iq.com)**.

---

## Architecture Overview

```
Browser
   │
   ▼
Cloudflare DNS  →  portfolio.ibtisam-iq.com
   │                (CNAME → CloudFront distribution domain)
   ▼
CloudFront Distribution (HTTPS, OAC, custom domain)
   │   ↑ ACM certificate (us-east-1) for portfolio.ibtisam-iq.com
   │   ↑ KMS key (us-east-1) — CloudFront decrypts on read
   ▼
S3 Primary Bucket  (us-east-1 · private · KMS-SSE · versioning ON)
   │   portfolio-site-primary
   │
   ├── CloudTrail  →  S3 logging bucket  (API audit trail)
   └── S3 Server Access Logs  →  S3 logging bucket
   │
   ▼  Cross-Region Replication (CRR)
S3 Replica Bucket  (us-west-2 · private · KMS-SSE · versioning ON)
       portfolio-site-replica
       ↑ KMS key (us-west-2) — re-encrypts replicated objects
```

### AWS Services Used

| Service | Role |
|---|---|
| S3 (primary — us-east-1) | Origin bucket — stores all static site files, private, versioned |
| S3 (replica — us-west-2) | Disaster recovery / Cross-Region Replication target |
| KMS (us-east-1) | Encrypts objects at rest in the primary bucket; used by CloudFront (OAC) and CRR |
| KMS (us-west-2) | Encrypts replicated objects at rest in the replica bucket |
| CloudFront | Global CDN — serves content from S3 via OAC; handles TLS termination |
| ACM | TLS certificate for `portfolio.ibtisam-iq.com` (must be issued in `us-east-1` for CloudFront) |
| IAM | Replication role granting S3 cross-region copy permissions; CloudFront OAC service principal |
| CloudTrail | Audit log of every API call against both buckets (management + data events) |
| S3 Server Access Logs | Per-request HTTP-level access log on the origin bucket |
| Cloudflare DNS | CNAME record pointing `portfolio.ibtisam-iq.com` → CloudFront distribution domain |
| Lifecycle Policy | Transitions older object versions to Glacier after 30 days |
| Pre-signed URLs | Time-limited direct access to private objects (optional / for demos) |

---

## Stage 1 — Storage, Encryption & Upload

### Phase 1 — S3 Buckets (Primary + Replica)

Two buckets were created — one in `us-east-1` as the CloudFront origin, one in `us-west-2` as the CRR target. Both buckets are **fully private**; no public access is ever granted directly.

```bash
export PRIMARY_BUCKET="portfolio-site-primary"
export REPLICA_BUCKET="portfolio-site-replica"
export ACCOUNT_ID="<your-aws-account-id>"

# Create primary bucket (us-east-1)
aws s3 mb s3://$PRIMARY_BUCKET --region us-east-1

# Create replica bucket (us-west-2)
aws s3 mb s3://$REPLICA_BUCKET --region us-west-2

# Block all public access on both buckets
aws s3api put-public-access-block --bucket $PRIMARY_BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

aws s3api put-public-access-block --bucket $REPLICA_BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Enable versioning on both (required for CRR)
aws s3api put-bucket-versioning \
  --bucket $PRIMARY_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning \
  --bucket $REPLICA_BUCKET \
  --versioning-configuration Status=Enabled
```

> **Why versioning?** Cross-Region Replication only works when versioning is enabled on both source and destination buckets. It also enables lifecycle policies to transition old versions to Glacier, and protects against accidental overwrites or deletes.

---

### Phase 2 — KMS Encryption Keys

KMS keys are **regional** — a key in `us-east-1` cannot be used to encrypt or decrypt objects in `us-west-2`. Two separate keys are required.

#### Key 1 — Primary Bucket (us-east-1)

This key protects objects stored in the primary bucket. CloudFront uses this key to **decrypt** objects when serving them via OAC. During CRR, S3 uses this key to **decrypt** the object at the source before replicating.

```bash
KMS_KEY_ID1=$(aws kms create-key \
  --description "S3 encryption key for portfolio-site primary bucket" \
  --region us-east-1 \
  --query KeyMetadata.KeyId --output text)

KMS_KEY_ARN1=$(aws kms describe-key \
  --key-id $KMS_KEY_ID1 --region us-east-1 \
  --query KeyMetadata.Arn --output text)

aws kms create-alias \
  --alias-name alias/portfolio-site-primary \
  --target-key-id $KMS_KEY_ID1 \
  --region us-east-1
```

**Key policy — Primary bucket** (apply via AWS Console → KMS → Key policy → Edit):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<your-account-id>:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowCloudFrontToDecrypt",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3ReplicationUse",
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": ["kms:Encrypt","kms:Decrypt","kms:ReEncrypt*","kms:GenerateDataKey*","kms:DescribeKey"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<your-account-id>",
          "aws:SourceArn": "arn:aws:s3:::portfolio-site-primary"
        }
      }
    }
  ]
}
```

> **What this policy does:** Root retains full control. CloudFront can decrypt for delivery. S3 (scoped to the primary bucket ARN) can decrypt at the CRR source side.

#### Key 2 — Replica Bucket (us-west-2)

S3 uses this key to **re-encrypt** the replicated data at the destination. CloudFront never reads from the replica, so no CloudFront statement is needed here.

```bash
KMS_KEY_ID2=$(aws kms create-key \
  --description "S3 encryption key for portfolio-site replica bucket" \
  --region us-west-2 \
  --query KeyMetadata.KeyId --output text)

KMS_KEY_ARN2=$(aws kms describe-key \
  --key-id $KMS_KEY_ID2 --region us-west-2 \
  --query KeyMetadata.Arn --output text)

aws kms create-alias \
  --alias-name alias/portfolio-site-replica \
  --target-key-id $KMS_KEY_ID2 \
  --region us-west-2
```

**Key policy — Replica bucket:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<your-account-id>:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowS3ReplicationUseReplica",
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": ["kms:Encrypt","kms:Decrypt","kms:ReEncrypt*","kms:GenerateDataKey*","kms:DescribeKey"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<your-account-id>",
          "aws:SourceArn": "arn:aws:s3:::portfolio-site-replica"
        }
      }
    }
  ]
}
```

#### Apply Default Bucket Encryption

```bash
# Primary bucket
aws s3api put-bucket-encryption \
  --bucket $PRIMARY_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "'"$KMS_KEY_ID1"'"
    }}]
  }'

# Replica bucket
aws s3api put-bucket-encryption \
  --bucket $REPLICA_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "'"$KMS_KEY_ID2"'"
    }}]
  }'
```

---

### Phase 3 — Upload Site Content

```bash
aws s3 sync . s3://$PRIMARY_BUCKET \
  --region us-east-1 \
  --sse aws:kms \
  --sse-kms-key-id $KMS_KEY_ID1 \
  --exclude ".git/*" \
  --exclude "*.md" \
  --delete
```

> `--delete` removes any S3 objects that no longer exist in the local source. `--exclude ".git/*"` prevents Git internals from being uploaded. `--sse` enforces KMS encryption on every uploaded object even if the bucket default is already set.

Verify the upload:

```bash
aws s3 ls s3://$PRIMARY_BUCKET --recursive --human-readable
```

---

## Stage 2 — IAM Replication Role

### Phase 4 — IAM Role for Cross-Region Replication

S3 needs an IAM role to assume when copying objects from the primary bucket to the replica. The role must allow S3 to read from the source and write to the destination, and must have access to both KMS keys for the decrypt/re-encrypt operation.

#### Create the Role

```bash
aws iam create-role \
  --role-name s3-crr-portfolio-site \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'
```

#### Attach the Replication Policy

```bash
aws iam put-role-policy \
  --role-name s3-crr-portfolio-site \
  --policy-name crr-replication-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowSourceBucketRead",
        "Effect": "Allow",
        "Action": [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ],
        "Resource": "arn:aws:s3:::portfolio-site-primary"
      },
      {
        "Sid": "AllowSourceObjectRead",
        "Effect": "Allow",
        "Action": [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ],
        "Resource": "arn:aws:s3:::portfolio-site-primary/*"
      },
      {
        "Sid": "AllowDestinationWrite",
        "Effect": "Allow",
        "Action": [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ],
        "Resource": "arn:aws:s3:::portfolio-site-replica/*"
      },
      {
        "Sid": "AllowKMSDecryptSource",
        "Effect": "Allow",
        "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
        "Resource": "'"$KMS_KEY_ARN1"'"
      },
      {
        "Sid": "AllowKMSEncryptDestination",
        "Effect": "Allow",
        "Action": ["kms:Encrypt", "kms:GenerateDataKey"],
        "Resource": "'"$KMS_KEY_ARN2"'"
      }
    ]
  }'
```

#### Enable Replication on the Primary Bucket

```bash
export CRR_ROLE_ARN=$(aws iam get-role \
  --role-name s3-crr-portfolio-site \
  --query Role.Arn --output text)

aws s3api put-bucket-replication \
  --bucket $PRIMARY_BUCKET \
  --replication-configuration '{
    "Role": "'"$CRR_ROLE_ARN"'",
    "Rules": [{
      "ID": "ReplicateAll",
      "Status": "Enabled",
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::portfolio-site-replica",
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "'"$KMS_KEY_ARN2"'"
        }
      },
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": { "Status": "Enabled" }
      },
      "DeleteMarkerReplication": { "Status": "Enabled" }
    }]
  }'
```

> **Why `SourceSelectionCriteria.SseKmsEncryptedObjects`?** Without this, S3 silently skips KMS-encrypted objects during replication. This flag is mandatory when the source bucket uses SSE-KMS.

#### Verify Replication After Upload

```bash
aws s3api get-bucket-replication --bucket $PRIMARY_BUCKET

# Confirm objects exist in replica
aws s3 ls s3://$REPLICA_BUCKET --recursive --human-readable
```

---

## Stage 3 — CloudFront Distribution

### Phase 5 — ACM Certificate (us-east-1)

CloudFront requires its TLS certificate to be issued in `us-east-1` regardless of where your resources are. This is a hard AWS constraint.

```bash
aws acm request-certificate \
  --domain-name portfolio.ibtisam-iq.com \
  --validation-method DNS \
  --region us-east-1
```

Go to **AWS Console → ACM → us-east-1** → find the pending certificate → copy the **CNAME name** and **CNAME value** → add them as a CNAME record in **Cloudflare DNS** for your domain.

Wait for status to change to **Issued** (usually 1-5 minutes after the DNS record propagates):

```bash
aws acm list-certificates --region us-east-1
```

Export the ARN:

```bash
export ACM_CERT_ARN="arn:aws:acm:us-east-1:<your-account-id>:certificate/<cert-id>"
```

---

### Phase 6 — CloudFront Origin Access Control (OAC)

OAC is the modern replacement for Origin Access Identity (OAI). It signs requests to S3 using SigV4, works with SSE-KMS encrypted buckets, and does not require public S3 access.

Create OAC via the **AWS Console → CloudFront → Origin access → Create control setting**:

| Field | Value |
|---|---|
| Name | `portfolio-site-oac` |
| Origin type | S3 |
| Signing behavior | Sign requests (recommended) |
| Signing protocol | SigV4 |

Copy the OAC ID — you will reference it in the distribution config.

---

### Phase 7 — Create the CloudFront Distribution

Create via **AWS Console → CloudFront → Create distribution** with the following settings:

| Setting | Value |
|---|---|
| Origin domain | `portfolio-site-primary.s3.us-east-1.amazonaws.com` |
| Origin access | Origin access control (OAC) — select `portfolio-site-oac` |
| Viewer protocol policy | Redirect HTTP to HTTPS |
| Allowed HTTP methods | GET, HEAD |
| Cache policy | `CachingOptimized` (managed) |
| Compress objects | Yes |
| Alternate domain name (CNAME) | `portfolio.ibtisam-iq.com` |
| Custom SSL certificate | Select the ACM cert issued above |
| Default root object | `index.html` |

After creation, copy the **Distribution domain name** (e.g., `d1abc123xyz.cloudfront.net`).

---

### Phase 8 — S3 Bucket Policy (Allow CloudFront OAC)

After creating the distribution, CloudFront will prompt you to **copy the bucket policy** — use it directly. It grants the CloudFront service principal access to the bucket, scoped to your specific distribution ARN.

Apply it via **AWS Console → S3 → Permissions → Bucket Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::portfolio-site-primary/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<your-account-id>:distribution/<distribution-id>"
        }
      }
    }
  ]
}
```

> **Critical:** The `AWS:SourceArn` condition scopes this permission to your specific CloudFront distribution only. Without this condition, any CloudFront distribution in any AWS account could read your bucket.

---

## Stage 4 — DNS & Custom Domain

### Phase 9 — Cloudflare DNS

In the **Cloudflare dashboard → DNS → Add record**:

| Type | Name | Content | Proxy status |
|---|---|---|---|
| CNAME | `portfolio` | `d1abc123xyz.cloudfront.net` | DNS only (grey cloud) |

> **Why "DNS only" and not proxied?** When Cloudflare proxies the request, it terminates the TLS connection and CloudFront sees Cloudflare's IP instead of the user's. This can break CloudFront's SNI-based certificate matching and geo-restriction features. Use DNS-only for CloudFront origins.

After adding the record, verify propagation:

```bash
dig portfolio.ibtisam-iq.com CNAME
# Should return: portfolio.ibtisam-iq.com → d1abc123xyz.cloudfront.net
```

---

## Stage 4B — Observability, Audit & Lifecycle

### Phase 10 — CloudTrail Audit Logging

Create a **dedicated logging bucket** before enabling CloudTrail. This bucket should be in the same region (`us-east-1`) and must have a bucket policy allowing CloudTrail to write to it.

```bash
export LOG_BUCKET="portfolio-site-logs"

aws s3 mb s3://$LOG_BUCKET --region us-east-1

aws s3api put-public-access-block --bucket $LOG_BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**Bucket policy for the logging bucket** (CloudTrail requires this exact format):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::portfolio-site-logs"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::portfolio-site-logs/AWSLogs/<your-account-id>/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

#### Create the Trail

```bash
aws cloudtrail create-trail \
  --name portfolio-site-trail \
  --s3-bucket-name $LOG_BUCKET \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --region us-east-1

aws cloudtrail start-logging \
  --name portfolio-site-trail \
  --region us-east-1
```

#### Enable S3 Data Events

By default, CloudTrail only logs management events (bucket creates, policy updates). To also log every `GetObject`, `PutObject`, and `DeleteObject` call on your primary bucket:

```bash
aws cloudtrail put-event-selectors \
  --trail-name portfolio-site-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::portfolio-site-primary/"]
    }]
  }]' \
  --region us-east-1
```

#### Enable S3 Server Access Logging

CloudTrail logs API calls. S3 Server Access Logs capture the raw HTTP request log — useful for debugging cache misses and access patterns.

```bash
aws s3api put-bucket-logging \
  --bucket $PRIMARY_BUCKET \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "portfolio-site-logs",
      "TargetPrefix": "s3-access-logs/"
    }
  }'
```

---

### Phase 10B — Lifecycle Policy (Glacier Tiering)

Transitions non-current object versions (old deploys) to Glacier after 30 days. This prevents storage costs from accumulating as you iterate on the site.

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket $PRIMARY_BUCKET \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionTransitions": [{
        "NoncurrentDays": 30,
        "StorageClass": "GLACIER"
      }],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 365
      }
    }]
  }'
```

---

## Stage 5 — Verification

### Phase 11 — End-to-End Checks

#### 1. HTTPS via Custom Domain

```bash
curl -I https://portfolio.ibtisam-iq.com
# Expected: HTTP/2 200, x-cache: Hit from cloudfront (after warm-up)
```

#### 2. Direct S3 Access Must Return 403

```bash
curl -I https://portfolio-site-primary.s3.us-east-1.amazonaws.com/index.html
# Expected: 403 Forbidden — confirms no public S3 access
```

#### 3. CloudTrail — Verify Events Are Flowing

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=portfolio-site-primary \
  --region us-east-1 \
  --max-results 5
```

#### 4. Replication — Confirm Object Count Matches

```bash
# Count objects in primary
aws s3 ls s3://$PRIMARY_BUCKET --recursive | wc -l

# Count objects in replica
aws s3 ls s3://$REPLICA_BUCKET --recursive | wc -l
# Should be equal (allow a few minutes for CRR to complete)
```

#### 5. Pre-signed URL (Optional — Demonstrates Private Bucket Access)

```bash
aws s3 presign s3://$PRIMARY_BUCKET/index.html \
  --region us-east-1 \
  --expires-in 300
# Returns a time-limited URL valid for 5 minutes
```

---

## Stage 6 — Troubleshooting

### CRR Not Replicating (Most Common Issue)

**Symptom:** Objects exist in the primary bucket but the replica is empty or incomplete after several minutes.

**Root cause:** The IAM replication role's KMS permissions are missing or the KMS key policies do not include the `s3.amazonaws.com` service principal.

**Fix:**
1. Check the KMS key policy on the **primary bucket's key** — confirm the `AllowS3ReplicationUse` statement is present with the correct `aws:SourceArn` condition.
2. Check the KMS key policy on the **replica bucket's key** — confirm `AllowS3ReplicationUseReplica` is present.
3. Check the IAM replication role policy — confirm both `kms:Decrypt` (for the source key ARN) and `kms:Encrypt` (for the destination key ARN) are present.
4. Verify `SourceSelectionCriteria.SseKmsEncryptedObjects` is set to `Enabled` in the replication rule — without this, S3 silently skips KMS objects.

```bash
# Re-check replication config
aws s3api get-bucket-replication --bucket $PRIMARY_BUCKET

# Check replication status on a specific object
aws s3api head-object \
  --bucket $PRIMARY_BUCKET \
  --key index.html \
  --query ReplicationStatus
# Should return: "COMPLETED"
```

---

### CloudFront Returns 403 on Root URL

**Symptom:** `https://portfolio.ibtisam-iq.com` returns `403 Forbidden` but direct object URLs work.

**Root cause:** The CloudFront distribution's **Default root object** is not set to `index.html`.

**Fix:** AWS Console → CloudFront → Distribution → Settings → Edit → set **Default root object** to `index.html` → Deploy.

---

### ACM Certificate Stuck in `PENDING_VALIDATION`

**Symptom:** Certificate status remains `PENDING_VALIDATION` for more than 10 minutes.

**Root cause:** The DNS validation CNAME record was not added to Cloudflare, or Cloudflare is proxying it (orange cloud) which may interfere with ACM's DNS lookup.

**Fix:**
1. In Cloudflare → DNS, confirm the ACM validation CNAME record exists with **proxy status = DNS only (grey cloud)**.
2. Verify the record using `dig`:
   ```bash
   dig _<acm-token>.portfolio.ibtisam-iq.com CNAME
   ```
3. Wait up to 5 minutes after the dig confirms propagation.

---

### CloudFront 403 After OAC Setup

**Symptom:** Distribution is deployed but all requests return 403.

**Root cause:** The S3 bucket policy was not updated after creating the OAC-based distribution.

**Fix:** Return to S3 → Bucket Policy → paste the CloudFront-generated bucket policy that scopes access to your specific distribution ARN (see Phase 9). Also confirm the KMS key policy includes `AllowCloudFrontToDecrypt`.

---

## Deployment Summary

| Stage | Phase | What was done |
|---|---|---|
| **Stage 1** — Source | Phase 1 | Cloned portfolio-site repo, confirmed static structure, local preview |
| **Stage 2** — Storage | Phase 2 | Created private versioned S3 primary + replica buckets |
| **Stage 2** — Encryption | Phase 3 | Created two regional KMS keys with scoped key policies |
| **Stage 2** — Upload | Phase 4 | Synced site files to S3 with SSE-KMS enforcement |
| **Stage 3** — IAM | Phase 5 | Created CRR IAM role + replication policy; enabled CRR with KMS re-encryption |
| **Stage 4** — CDN | Phase 6 | Issued ACM certificate in us-east-1 with Cloudflare DNS validation |
| **Stage 4** — CDN | Phase 7 | Created CloudFront OAC (SigV4 signing) |
| **Stage 4** — CDN | Phase 8 | Created CloudFront distribution (HTTPS, OAC, custom domain, index.html default) |
| **Stage 4** — CDN | Phase 9 | Applied S3 bucket policy scoped to CloudFront distribution ARN |
| **Stage 5** — DNS | Phase 10 | Added Cloudflare CNAME (DNS-only) pointing to CloudFront domain |
| **Stage 5B** — Observability | Phase 11 | Created CloudTrail trail with data events + S3 Server Access Logs |
| **Stage 5B** — Lifecycle | Phase 11B | Added lifecycle policy to Glacier-tier old object versions after 30 days |
| **Stage 6** — Verification | Phase 12 | HTTPS check, S3 403 confirm, CloudTrail event lookup, CRR object count, pre-signed URL |
| **Stage 7** — Troubleshooting | — | CRR 100% failure (KMS policies), CloudFront 403 (OAC/bucket policy), ACM pending, root 403 |
