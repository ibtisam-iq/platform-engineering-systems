# Secure Static Hosting and Global Distribution (S3 + CloudFront + KMS + IAM + CloudTrail)

A production-grade, globally distributed static site deployment on AWS: private S3 origin served through CloudFront with Origin Access Control, KMS encryption at rest, Cross-Region Replication for disaster recovery, and full audit logging via CloudTrail.

The [portfolio-site](https://github.com/ibtisam-iq/portfolio-site) served as the static content for this deployment. The site is live at **[portfolio.ibtisam-iq.com](https://portfolio.ibtisam-iq.com)**.

---

## Architecture Overview

```
Browser
   |
   v
Cloudflare DNS  -->  portfolio.ibtisam-iq.com
   |                 (CNAME to CloudFront distribution domain)
   v
CloudFront Distribution (HTTPS, OAC, custom domain)
   |   ^ ACM certificate (us-east-1) for portfolio.ibtisam-iq.com
   |   ^ KMS key (us-east-1): CloudFront decrypts on read
   v
S3 Primary Bucket  (us-east-1, private, KMS-SSE, Bucket Key ON, versioning ON)
   |   portfolio-site-primary-<account-id>
   |
   |-- CloudTrail  -->  S3 logging bucket  (API audit trail)
   |-- S3 Server Access Logs  -->  S3 logging bucket
   |
   v  Cross-Region Replication (CRR)
S3 Replica Bucket  (us-west-2, private, KMS-SSE, Bucket Key ON, versioning ON)
       portfolio-site-replica-<account-id>
       ^ KMS key (us-west-2): re-encrypts replicated objects
```

### AWS Services Used

| Service | Role |
|---|---|
| S3 (primary, us-east-1) | Origin bucket: stores all static site files, private, versioned |
| S3 (replica, us-west-2) | Disaster recovery / Cross-Region Replication target |
| KMS (us-east-1) | Encrypts objects at rest in the primary bucket; used by CloudFront (OAC) and CRR |
| KMS (us-west-2) | Encrypts replicated objects at rest in the replica bucket |
| CloudFront | Global CDN: serves content from S3 via OAC, handles TLS termination |
| ACM | TLS certificate for `portfolio.ibtisam-iq.com` (must be issued in `us-east-1` for CloudFront) |
| IAM | Replication role granting S3 cross-region copy permissions; CloudFront OAC service principal |
| CloudTrail | Audit log of every API call against both buckets (management + data events) |
| S3 Server Access Logs | Per-request HTTP-level access log on the origin bucket |
| Cloudflare DNS | CNAME record pointing `portfolio.ibtisam-iq.com` to CloudFront distribution domain |
| Lifecycle Policy | Transitions older object versions to Glacier after 30 days |

---

## Stage 1: Storage and Encryption

### Phase 1: S3 Buckets (Primary + Replica)

Created two buckets: one in `us-east-1` as the CloudFront origin, one in `us-west-2` as the CRR target. Both buckets are fully private; no public access is ever granted directly.

S3 bucket names are globally unique across all AWS accounts. Appending the AWS Account ID as a suffix guarantees uniqueness without guessing.

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export PRIMARY_BUCKET="portfolio-site-primary-${ACCOUNT_ID}"
export REPLICA_BUCKET="portfolio-site-replica-${ACCOUNT_ID}"
export LOG_BUCKET="portfolio-site-logs-${ACCOUNT_ID}"

# Primary bucket (us-east-1)
aws s3 mb s3://$PRIMARY_BUCKET --region us-east-1

# Replica bucket (us-west-2)
aws s3 mb s3://$REPLICA_BUCKET --region us-west-2

# Block all public access on both buckets
for BUCKET in $PRIMARY_BUCKET $REPLICA_BUCKET; do
  aws s3api put-public-access-block --bucket $BUCKET \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
done

# Enable versioning on both (required for CRR)
for BUCKET in $PRIMARY_BUCKET $REPLICA_BUCKET; do
  aws s3api put-bucket-versioning \
    --bucket $BUCKET \
    --versioning-configuration Status=Enabled
done
```

> **Why versioning?** Cross-Region Replication only works when versioning is enabled on both source and destination buckets. It also enables lifecycle policies to transition old versions to Glacier, and protects against accidental overwrites or deletes.

---

### Phase 2: KMS Encryption Keys

KMS keys are regional: a key in `us-east-1` cannot encrypt or decrypt objects in `us-west-2`. Two separate keys were required.

#### Key 1: Primary Bucket (us-east-1)

This key protects objects stored in the primary bucket. CloudFront uses this key to decrypt objects when serving them via OAC. During CRR, S3 uses this key to decrypt the object at the source before replicating.

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

**Key policy for the primary bucket:**

```bash
aws kms put-key-policy \
  --key-id $KMS_KEY_ID1 \
  --region us-east-1 \
  --policy-name default \
  --policy "$(cat <<'POLICY'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID_PLACEHOLDER:root" },
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
          "aws:SourceAccount": "ACCOUNT_ID_PLACEHOLDER"
        }
      }
    }
  ]
}
POLICY
)" | sed "s/ACCOUNT_ID_PLACEHOLDER/${ACCOUNT_ID}/g"
```

Wait, that sed approach won't work with the heredoc piped to aws kms. Let me use the standard heredoc with variable expansion:

```bash
aws kms put-key-policy \
  --key-id $KMS_KEY_ID1 \
  --region us-east-1 \
  --policy-name default \
  --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${ACCOUNT_ID}:root" },
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
          "aws:SourceAccount": "${ACCOUNT_ID}"
        }
      }
    }
  ]
}
EOF
)"
```

> **What this policy does:** Root retains full control. CloudFront can decrypt for delivery. S3 (scoped to the account) can decrypt at the CRR source side.
>
> The heredoc (`<<EOF`) causes the shell to interpolate `${ACCOUNT_ID}` from the current session before the JSON is passed to the AWS CLI.

#### Key 2: Replica Bucket (us-west-2)

S3 uses this key to re-encrypt the replicated data at the destination. CloudFront never reads from the replica, so no CloudFront statement is needed here.

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

**Key policy for the replica bucket:**

```bash
aws kms put-key-policy \
  --key-id $KMS_KEY_ID2 \
  --region us-west-2 \
  --policy-name default \
  --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${ACCOUNT_ID}:root" },
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
          "aws:SourceAccount": "${ACCOUNT_ID}"
        }
      }
    }
  ]
}
EOF
)"
```

#### Apply Default Bucket Encryption with Bucket Key Enabled

The S3 Bucket Key is a performance and cost optimization that sits between S3 and KMS. Without it, S3 makes one `GenerateDataKey` KMS API call per object on every PUT and GET, meaning 1,000 uploads = 1,000 KMS calls. With `BucketKeyEnabled: true`, KMS generates a single short-lived bucket-level key that S3 reuses locally to derive per-object keys, reducing KMS API calls by up to 99% and cutting KMS costs proportionally. The security model is identical either way.

> **Why set it here and not in Phase 1?** The Bucket Key is part of the SSE-KMS encryption configuration (`put-bucket-encryption`), which requires the KMS key ID to be known. Phase 1 only creates the buckets. This command must run after `KMS_KEY_ID1` and `KMS_KEY_ID2` are exported.

```bash
# Primary bucket: SSE-KMS with Bucket Key enabled
aws s3api put-bucket-encryption \
  --bucket $PRIMARY_BUCKET \
  --server-side-encryption-configuration "$(cat <<EOF
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "${KMS_KEY_ID1}"
    },
    "BucketKeyEnabled": true
  }]
}
EOF
)"

# Replica bucket: SSE-KMS with Bucket Key enabled
aws s3api put-bucket-encryption \
  --bucket $REPLICA_BUCKET \
  --server-side-encryption-configuration "$(cat <<EOF
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "${KMS_KEY_ID2}"
    },
    "BucketKeyEnabled": true
  }]
}
EOF
)"
```

Verified Bucket Key status on both buckets:

```bash
aws s3api get-bucket-encryption --bucket $PRIMARY_BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].BucketKeyEnabled'
# Expected: true

aws s3api get-bucket-encryption --bucket $REPLICA_BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].BucketKeyEnabled'
# Expected: true
```

> **Bucket Key and CRR:** When the source bucket has a Bucket Key enabled, replicated objects at the destination also inherit the Bucket Key behaviour, provided `BucketKeyEnabled: true` is set on the replica bucket encryption config as well (done above).

---

## Stage 2: IAM Replication Role and CRR

### Phase 3: IAM Role for Cross-Region Replication

S3 needs an IAM role to assume when copying objects from the primary bucket to the replica. The role must allow S3 to read from the source and write to the destination, and must have access to both KMS keys for the decrypt/re-encrypt operation.

> **Why CRR before upload?** CRR only replicates objects uploaded after the replication rule is active. Setting up replication first ensures the initial content sync reaches the replica automatically. Uploading first and then enabling CRR would leave the replica empty until the next update.

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
  --policy-document "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSourceBucketRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::${PRIMARY_BUCKET}"
    },
    {
      "Sid": "AllowSourceObjectRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::${PRIMARY_BUCKET}/*"
    },
    {
      "Sid": "AllowDestinationWrite",
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::${REPLICA_BUCKET}/*"
    },
    {
      "Sid": "AllowKMSDecryptSource",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
      "Resource": "${KMS_KEY_ARN1}"
    },
    {
      "Sid": "AllowKMSEncryptDestination",
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
      "Resource": "${KMS_KEY_ARN2}"
    }
  ]
}
EOF
)"
```

#### Enable Replication on the Primary Bucket

```bash
export CRR_ROLE_ARN=$(aws iam get-role \
  --role-name s3-crr-portfolio-site \
  --query Role.Arn --output text)

aws s3api put-bucket-replication \
  --bucket $PRIMARY_BUCKET \
  --replication-configuration "$(cat <<EOF
{
  "Role": "${CRR_ROLE_ARN}",
  "Rules": [{
    "ID": "ReplicateAll",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::${REPLICA_BUCKET}",
      "EncryptionConfiguration": {
        "ReplicaKmsKeyID": "${KMS_KEY_ARN2}"
      }
    },
    "SourceSelectionCriteria": {
      "SseKmsEncryptedObjects": { "Status": "Enabled" }
    },
    "DeleteMarkerReplication": { "Status": "Enabled" }
  }]
}
EOF
)"
```

> **Why `SourceSelectionCriteria.SseKmsEncryptedObjects`?** Without this, S3 silently skips KMS-encrypted objects during replication. This flag is mandatory when the source bucket uses SSE-KMS.
>
> **Why `Priority`?** Required for any replication rule that uses `Filter` (even an empty one). Without it, `put-bucket-replication` returns a `MalformedXML` error.

---

### Phase 4: Upload Site Content

> **Source directory:** Only the `dist/` folder was synced. It contains exactly the production build artefacts (HTML, CSS, JS, assets). No `.git/`, no source files, no config files exist in `dist/`, so no `--exclude` flags were needed.

```bash
# Run from the root of the portfolio-site repository
aws s3 sync dist/ s3://$PRIMARY_BUCKET \
  --region us-east-1 \
  --sse aws:kms \
  --sse-kms-key-id $KMS_KEY_ID1 \
  --delete
```

> **`--sse aws:kms`** explicitly enforces KMS encryption on every uploaded object, even if the bucket default is already set. This prevents any object being silently uploaded with SSE-S3 if a caller omits the header.
>
> **`--sse-kms-key-id $KMS_KEY_ID1`** pins each object to the specific CMK so the Bucket Key on the upload side is engaged correctly. Without this flag, AWS falls back to the bucket default key but the per-request encryption header may not carry the key ID, which can cause CloudFront OAC `kms:Decrypt` failures.
>
> **`--delete`** removes any S3 objects that no longer exist in `dist/`. Keeps the bucket in sync with the exact build output and prevents stale files from being served.

Verified the upload and confirmed replication reached the replica:

```bash
aws s3 ls s3://$PRIMARY_BUCKET --recursive --human-readable
aws s3 ls s3://$REPLICA_BUCKET --recursive --human-readable

# Check replication status on a specific object
aws s3api head-object \
  --bucket $PRIMARY_BUCKET \
  --key index.html \
  --query ReplicationStatus
# Expected: "COMPLETED"
```

---

## Stage 3: CloudFront Distribution

### Phase 5: ACM Certificate (us-east-1)

CloudFront requires its TLS certificate to be issued in `us-east-1` regardless of where other resources are. This is a hard AWS constraint.

```bash
export ACM_CERT_ARN=$(aws acm request-certificate \
  --domain-name portfolio.ibtisam-iq.com \
  --validation-method DNS \
  --region us-east-1 \
  --query CertificateArn --output text)

echo "ACM_CERT_ARN=$ACM_CERT_ARN"
```

Extracted the DNS validation CNAME record that ACM requires:

```bash
# Wait a few seconds for ACM to generate the validation record
sleep 5

aws acm describe-certificate \
  --certificate-arn $ACM_CERT_ARN \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord.{Name:Name,Value:Value}' \
  --output table
```

Added the CNAME name and value as a DNS record in Cloudflare (proxy status: DNS only / grey cloud). Then waited for the certificate status to change to `ISSUED`:

```bash
# Poll until issued (typically 1 to 5 minutes after DNS propagation)
aws acm wait certificate-validated \
  --certificate-arn $ACM_CERT_ARN \
  --region us-east-1

aws acm describe-certificate \
  --certificate-arn $ACM_CERT_ARN \
  --region us-east-1 \
  --query 'Certificate.Status'
# Expected: "ISSUED"
```

---

### Phase 6: CloudFront Origin Access Control (OAC)

OAC is the modern replacement for Origin Access Identity (OAI). It signs requests to S3 using SigV4, works with SSE-KMS encrypted buckets, and does not require public S3 access.

```bash
export OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "portfolio-site-oac",
    "Description": "OAC for portfolio-site S3 origin",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' \
  --query 'OriginAccessControl.Id' \
  --output text)

echo "OAC_ID=$OAC_ID"
```

---

### Phase 7: Create the CloudFront Distribution

Created the distribution with the S3 REST API endpoint as origin, OAC signing, HTTPS redirect, the ACM certificate for the custom domain, and `CachingOptimized` as the managed cache policy.

> **`CachingOptimized` policy ID:** `658327ea-f89d-4fab-a63d-7e88639e58f6` is the AWS-managed CachingOptimized cache policy. It sets a default TTL of 86400s (24h), enables Gzip and Brotli compression, and forwards no headers, cookies, or query strings to the origin. This is the recommended policy for static site origins.

```bash
ORIGIN_DOMAIN="${PRIMARY_BUCKET}.s3.us-east-1.amazonaws.com"
CALLER_REF=$(date +%s)

CF_OUTPUT=$(aws cloudfront create-distribution \
  --distribution-config "$(cat <<EOF
{
  "CallerReference": "${CALLER_REF}",
  "Comment": "portfolio-site CDN",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "Aliases": {
    "Quantity": 1,
    "Items": ["portfolio.ibtisam-iq.com"]
  },
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3Origin",
      "DomainName": "${ORIGIN_DOMAIN}",
      "OriginAccessControlId": "${OAC_ID}",
      "S3OriginConfig": {
        "OriginAccessIdentity": ""
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "${ACM_CERT_ARN}",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [{
      "ErrorCode": 403,
      "ResponsePagePath": "/index.html",
      "ResponseCode": "200",
      "ErrorCachingMinTTL": 0
    }]
  },
  "HttpVersion": "http2and3",
  "PriceClass": "PriceClass_100"
}
EOF
)" --output json)

export CF_DISTRIBUTION_ID=$(echo $CF_OUTPUT | jq -r '.Distribution.Id')
export CF_DOMAIN=$(echo $CF_OUTPUT | jq -r '.Distribution.DomainName')
export CF_ETAG=$(echo $CF_OUTPUT | jq -r '.ETag')

echo "CF_DISTRIBUTION_ID=$CF_DISTRIBUTION_ID"
echo "CF_DOMAIN=$CF_DOMAIN"
```

> **`S3OriginConfig.OriginAccessIdentity: ""`** is required even when using OAC. It tells CloudFront this is an S3 REST API origin (not a custom origin) but that OAI is not in use. Omitting this field causes a validation error.
>
> **`CustomErrorResponses` for 403:** A single-page application (SPA) with client-side routing returns 403 from S3 for any path other than `index.html`, because no such S3 key exists. This error response maps 403 back to `index.html` with HTTP 200 so the client router handles the path.
>
> **`PriceClass_100`:** Limits edge locations to North America and Europe, the cheapest tier. Sufficient for a portfolio site; avoids charges from Asia/South America edge locations with minimal traffic.
>
> **`HttpVersion: http2and3`:** Enables HTTP/3 (QUIC) for clients that support it, reducing connection latency on mobile and lossy networks.

---

### Phase 8: S3 Bucket Policy (Allow CloudFront OAC)

After creating the distribution, applied the bucket policy that grants the CloudFront service principal access to the bucket, scoped to the specific distribution ARN.

```bash
aws s3api put-bucket-policy \
  --bucket $PRIMARY_BUCKET \
  --policy "$(cat <<EOF
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
      "Resource": "arn:aws:s3:::${PRIMARY_BUCKET}/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${CF_DISTRIBUTION_ID}"
        }
      }
    }
  ]
}
EOF
)"
```

> **Critical:** The `AWS:SourceArn` condition scopes this permission to one specific CloudFront distribution. Without this condition, any CloudFront distribution in any AWS account could read the bucket.

---

## Stage 4: DNS and Custom Domain

### Phase 9: Cloudflare DNS

Added a CNAME record in the Cloudflare dashboard:

| Type | Name | Content | Proxy status |
|---|---|---|---|
| CNAME | `portfolio` | `$CF_DOMAIN` (e.g., `d1abc123xyz.cloudfront.net`) | DNS only (grey cloud) |

> **Why "DNS only" and not proxied?** When Cloudflare proxies the request, it terminates the TLS connection and CloudFront sees Cloudflare's IP instead of the client's. This can break CloudFront's SNI-based certificate matching and geo-restriction features. DNS-only is required for CloudFront origins.

Verified propagation:

```bash
dig portfolio.ibtisam-iq.com CNAME +short
# Expected: d1abc123xyz.cloudfront.net.
```

---

## Stage 5: Observability, Audit, and Lifecycle

### Phase 10: CloudTrail Audit Logging

Created a dedicated logging bucket before enabling CloudTrail or S3 access logging. This bucket grants write access to both the CloudTrail service and the S3 log delivery service.

```bash
aws s3 mb s3://$LOG_BUCKET --region us-east-1

aws s3api put-public-access-block --bucket $LOG_BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**Bucket policy for the logging bucket** (grants both CloudTrail and S3 Server Access Logging):

```bash
aws s3api put-bucket-policy \
  --bucket $LOG_BUCKET \
  --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${LOG_BUCKET}"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${LOG_BUCKET}/AWSLogs/${ACCOUNT_ID}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Sid": "S3ServerAccessLogsWrite",
      "Effect": "Allow",
      "Principal": { "Service": "logging.s3.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${LOG_BUCKET}/s3-access-logs/*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "${ACCOUNT_ID}"
        }
      }
    }
  ]
}
EOF
)"
```

> **Why the `S3ServerAccessLogsWrite` statement?** The original runbook only granted CloudTrail permissions. The `put-bucket-logging` call in Phase 10 would succeed, but S3 would silently fail to deliver access logs because the target bucket never authorized the `logging.s3.amazonaws.com` service principal.

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

By default, CloudTrail only logs management events (bucket creates, policy updates). Enabled data events to also log every `GetObject`, `PutObject`, and `DeleteObject` call on the primary bucket:

```bash
aws cloudtrail put-event-selectors \
  --trail-name portfolio-site-trail \
  --event-selectors "$(cat <<EOF
[{
  "ReadWriteType": "All",
  "IncludeManagementEvents": true,
  "DataResources": [{
    "Type": "AWS::S3::Object",
    "Values": ["arn:aws:s3:::${PRIMARY_BUCKET}/"]
  }]
}]
EOF
)" \
  --region us-east-1
```

#### Enable S3 Server Access Logging

CloudTrail logs API calls. S3 Server Access Logs capture the raw HTTP request log, useful for debugging cache misses and access patterns.

```bash
aws s3api put-bucket-logging \
  --bucket $PRIMARY_BUCKET \
  --bucket-logging-status "$(cat <<EOF
{
  "LoggingEnabled": {
    "TargetBucket": "${LOG_BUCKET}",
    "TargetPrefix": "s3-access-logs/"
  }
}
EOF
)"
```

---

### Phase 10B: Lifecycle Policy (Glacier Tiering)

Transitioned non-current object versions (old deploys) to Glacier after 30 days. This prevents storage costs from accumulating across iterations of the site.

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

## Stage 6: Verification

### Phase 11: End-to-End Checks

#### 1. HTTPS via Custom Domain

```bash
curl -I https://portfolio.ibtisam-iq.com
# Expected: HTTP/2 200, x-cache: Hit from cloudfront (after warm-up)
```

#### 2. Direct S3 Access Must Return 403

```bash
curl -I https://$PRIMARY_BUCKET.s3.us-east-1.amazonaws.com/index.html
# Expected: 403 Forbidden, confirms no public S3 access
```

#### 3. CloudTrail: Verify Events Are Flowing

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=$PRIMARY_BUCKET \
  --region us-east-1 \
  --max-results 5
```

#### 4. Replication: Confirm Object Count Matches

```bash
echo "Primary: $(aws s3 ls s3://$PRIMARY_BUCKET --recursive | wc -l) objects"
echo "Replica: $(aws s3 ls s3://$REPLICA_BUCKET --recursive | wc -l) objects"
# Counts should be equal (allow a few minutes for CRR to complete)
```

#### 5. Bucket Key: Confirm Enabled on Both Buckets

```bash
aws s3api get-bucket-encryption --bucket $PRIMARY_BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].BucketKeyEnabled'
# Expected: true

aws s3api get-bucket-encryption --bucket $REPLICA_BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].BucketKeyEnabled'
# Expected: true
```

#### 6. Pre-signed URL (Demonstrates Private Bucket Access)

```bash
aws s3 presign s3://$PRIMARY_BUCKET/index.html \
  --region us-east-1 \
  --expires-in 300
# Returns a time-limited URL valid for 5 minutes
```

---

## Stage 6B: Troubleshooting

### CRR Not Replicating (Most Common Issue)

**Symptom:** Objects exist in the primary bucket but the replica is empty or incomplete after several minutes.

**Root cause:** The IAM replication role's KMS permissions are missing or the KMS key policies do not include the `s3.amazonaws.com` service principal.

**Fix:**

1. Check the KMS key policy on the primary bucket's key: confirm the `AllowS3ReplicationUse` statement is present with the correct `aws:SourceAccount` condition.
2. Check the KMS key policy on the replica bucket's key: confirm `AllowS3ReplicationUseReplica` is present.
3. Check the IAM replication role policy: confirm both `kms:Decrypt` (source key ARN) and `kms:Encrypt` (destination key ARN) are present, and both include `kms:DescribeKey`.
4. Verify `SourceSelectionCriteria.SseKmsEncryptedObjects` is set to `Enabled` in the replication rule. Without this, S3 silently skips KMS objects.

```bash
aws s3api get-bucket-replication --bucket $PRIMARY_BUCKET

aws s3api head-object \
  --bucket $PRIMARY_BUCKET \
  --key index.html \
  --query ReplicationStatus
# Should return: "COMPLETED"
```

---

### CloudFront Returns 403 on Root URL

**Symptom:** `https://portfolio.ibtisam-iq.com` returns `403 Forbidden` but direct object URLs work.

**Root cause:** The CloudFront distribution's Default root object is not set to `index.html`.

**Fix:**

```bash
# Get current config
aws cloudfront get-distribution-config --id $CF_DISTRIBUTION_ID > /tmp/cf-config.json
ETAG=$(jq -r '.ETag' /tmp/cf-config.json)

# Edit DefaultRootObject and update
jq '.DistributionConfig.DefaultRootObject = "index.html" | .DistributionConfig' /tmp/cf-config.json > /tmp/cf-update.json
aws cloudfront update-distribution --id $CF_DISTRIBUTION_ID --if-match $ETAG --distribution-config file:///tmp/cf-update.json
```

---

### ACM Certificate Stuck in PENDING_VALIDATION

**Symptom:** Certificate status remains `PENDING_VALIDATION` for more than 10 minutes.

**Root cause:** The DNS validation CNAME record was not added to Cloudflare, or Cloudflare is proxying it (orange cloud) which can interfere with ACM's DNS lookup.

**Fix:**

1. In Cloudflare DNS, confirm the ACM validation CNAME record exists with proxy status = DNS only (grey cloud).
2. Verify the record:

```bash
dig _<acm-token>.portfolio.ibtisam-iq.com CNAME +short
```

3. Wait up to 5 minutes after `dig` confirms propagation.

---

### CloudFront 403 After OAC Setup

**Symptom:** Distribution is deployed but all requests return 403.

**Root cause:** The S3 bucket policy was not updated after creating the OAC-based distribution.

**Fix:** Re-apply the bucket policy from Phase 8 with the correct `CF_DISTRIBUTION_ID`. Also confirm the KMS key policy includes `AllowCloudFrontToDecrypt`.

---

## Stage 7: Teardown

Delete all resources in reverse dependency order. CloudFront distributions must be disabled before deletion, and S3 buckets must be emptied before removal.

> **KMS keys:** KMS does not allow immediate deletion. The minimum scheduling window is 7 days. The keys cost nothing while pending deletion.

### Step 1: Disable and Delete the CloudFront Distribution

```bash
# Get current config and ETag
aws cloudfront get-distribution-config --id $CF_DISTRIBUTION_ID > /tmp/cf-config.json
ETAG=$(jq -r '.ETag' /tmp/cf-config.json)

# Disable the distribution
jq '.DistributionConfig.Enabled = false | .DistributionConfig' /tmp/cf-config.json > /tmp/cf-disable.json
aws cloudfront update-distribution \
  --id $CF_DISTRIBUTION_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/cf-disable.json

echo "Waiting for distribution to reach Deployed state (this takes several minutes)..."
aws cloudfront wait distribution-deployed --id $CF_DISTRIBUTION_ID

# Get the updated ETag after disable
ETAG=$(aws cloudfront get-distribution-config --id $CF_DISTRIBUTION_ID --query 'ETag' --output text)

# Delete the distribution
aws cloudfront delete-distribution --id $CF_DISTRIBUTION_ID --if-match $ETAG
```

### Step 2: Delete the OAC

```bash
OAC_ETAG=$(aws cloudfront get-origin-access-control --id $OAC_ID --query 'ETag' --output text)
aws cloudfront delete-origin-access-control --id $OAC_ID --if-match $OAC_ETAG
```

### Step 3: Delete the ACM Certificate

```bash
aws acm delete-certificate --certificate-arn $ACM_CERT_ARN --region us-east-1
```

### Step 4: Stop and Delete CloudTrail

```bash
aws cloudtrail stop-logging --name portfolio-site-trail --region us-east-1
aws cloudtrail delete-trail --name portfolio-site-trail --region us-east-1
```

### Step 5: Remove Replication Configuration

```bash
aws s3api delete-bucket-replication --bucket $PRIMARY_BUCKET
```

### Step 6: Empty and Delete All S3 Buckets

```bash
# Empty all three buckets (including all versions and delete markers)
for BUCKET in $PRIMARY_BUCKET $REPLICA_BUCKET $LOG_BUCKET; do
  echo "Emptying $BUCKET..."
  aws s3api list-object-versions --bucket $BUCKET --output json \
    | jq -r '.Versions[]? | "aws s3api delete-object --bucket '"$BUCKET"' --key \"\(.Key)\" --version-id \(.VersionId)"' \
    | bash 2>/dev/null
  aws s3api list-object-versions --bucket $BUCKET --output json \
    | jq -r '.DeleteMarkers[]? | "aws s3api delete-object --bucket '"$BUCKET"' --key \"\(.Key)\" --version-id \(.VersionId)"' \
    | bash 2>/dev/null
  aws s3 rb s3://$BUCKET
done
```

### Step 7: Delete the IAM Replication Role

```bash
aws iam delete-role-policy --role-name s3-crr-portfolio-site --policy-name crr-replication-policy
aws iam delete-role --role-name s3-crr-portfolio-site
```

### Step 8: Schedule KMS Key Deletion

```bash
aws kms schedule-key-deletion --key-id $KMS_KEY_ID1 --pending-window-in-days 7 --region us-east-1
aws kms schedule-key-deletion --key-id $KMS_KEY_ID2 --pending-window-in-days 7 --region us-west-2
```

### Step 9: Remove Cloudflare DNS Records

Manually remove from the Cloudflare dashboard:

1. The CNAME record for `portfolio` pointing to the CloudFront domain.
2. The ACM validation CNAME record (the `_<token>.portfolio.ibtisam-iq.com` entry).

---

## Deployment Summary

| Stage | Phase | What was done |
|---|---|---|
| **Stage 1**: Storage | Phase 1 | Created private versioned S3 primary + replica buckets (names suffixed with Account ID) |
| **Stage 1**: Encryption | Phase 2 | Created two regional KMS keys with scoped key policies; applied SSE-KMS with Bucket Key enabled on both buckets |
| **Stage 2**: IAM / CRR | Phase 3 | Created CRR IAM role + replication policy; enabled CRR with KMS re-encryption |
| **Stage 2**: Upload | Phase 4 | Synced `dist/` to S3 with explicit SSE-KMS key ID after CRR was active |
| **Stage 3**: CDN | Phase 5 | Issued ACM certificate in us-east-1 with Cloudflare DNS validation |
| **Stage 3**: CDN | Phase 6 | Created CloudFront OAC (SigV4 signing) via CLI |
| **Stage 3**: CDN | Phase 7 | Created CloudFront distribution (HTTPS, OAC, custom domain, SPA error handling, HTTP/3) |
| **Stage 3**: CDN | Phase 8 | Applied S3 bucket policy scoped to CloudFront distribution ARN |
| **Stage 4**: DNS | Phase 9 | Added Cloudflare CNAME (DNS-only) pointing to CloudFront domain |
| **Stage 5**: Observability | Phase 10 | Created CloudTrail trail with data events + S3 Server Access Logs |
| **Stage 5**: Lifecycle | Phase 10B | Added lifecycle policy to Glacier-tier old object versions after 30 days |
| **Stage 6**: Verification | Phase 11 | HTTPS check, S3 403 confirm, CloudTrail events, Bucket Key state, CRR count, pre-signed URL |
| **Stage 6B**: Troubleshooting | - | CRR failure (KMS policies), CloudFront 403 (OAC/bucket policy), ACM pending, root 403 |
| **Stage 7**: Teardown | - | Full resource cleanup in reverse dependency order |