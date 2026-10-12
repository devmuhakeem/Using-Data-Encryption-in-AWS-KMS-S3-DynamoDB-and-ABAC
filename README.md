# Using Data Encryption in AWS: KMS, S3, DynamoDB, and ABAC

A hands-on AWS Builder Lab where I protected a healthcare company's medical images and sales data using AWS KMS, S3 server-side encryption, attribute-based access control, and two forms of DynamoDB client-side encryption, plus the AWS Encryption SDK for local files.

## Scenario
As an administrator at a fictional healthcare startup, I needed to encrypt medical imaging data in S3, control who could decrypt it without editing IAM policies for every change, and explore both server-side and client-side encryption options for a DynamoDB-backed sales analytics workload.

## What I did

### 1. Created a KMS key
- Created a symmetric, single-Region customer-managed key aliased **ImageAccessKey**, tagged `ImageType: MRI`
- Assigned key **administration** permissions to my own console user, and key **usage** permissions to the `viewerInstanceRoleForMedicalImages` IAM role attached to the viewer EC2 instance
- Enabled automatic key rotation

### 2. Enabled S3 server-side encryption
- Set the default encryption on the `corp-image-store-` bucket to SSE-KMS using the ImageAccessKey ARN, with S3 Bucket Key enabled
- Confirmed this only affects new uploads, not the five existing (unencrypted) coloring-contest images already in the bucket

### 3. Uploaded and verified encrypted objects
- Used the lab's Image Transfer tool from a Windows EC2 instance (via Fleet Manager RDP) to upload new X-ray images to the bucket
- Verified in the object's Properties tab that the new images showed SSE-KMS, the correct key ARN, and Bucket Key enabled
- Confirmed encrypted and unencrypted objects can coexist in the same bucket

### 4. Hit an Access Denied error, then fixed it with ABAC
- The viewer instance could open the unencrypted coloring-contest images but got Access Denied on the new X-ray images
- Inspected the IAM role's policy and found a `StringNotEqualsIgnoreCase` condition requiring the KMS key's `ImageType` tag to equal `XRAY` before decryption was allowed
- Instead of editing the IAM policy or the EC2 role, I changed the KMS key's `ImageType` tag from `MRI` to `XRAY`
- Re-ran the Image Viewer tool and confirmed the X-ray images now opened successfully — a live demonstration of attribute-based access control

### 5. Verified DynamoDB server-side encryption
- Wrote Python (boto3) to create `Table1` and load 1,033 European office-supply sales records from a CSV
- Confirmed the table used the default AWS-owned KMS key and that items were fully readable in the console — DynamoDB transparently decrypts for authorized requests

### 6. Implemented client-side encryption with a direct KMS provider
- Created a second key, **DynamoDB_Direct_CMP_Key**
- Used the DynamoDB Encryption SDK's `AwsKmsCryptographicMaterialsProvider` and `EncryptedTable` wrapper to encrypt North America sales records (under 1,000 units sold) before writing them to `Table2`
- Inserted 93 records; confirmed only the two primary-key attributes stayed readable, with the SDK adding `amzn-ddb-map-desc` and `amzn-ddb-map-sig` metadata attributes

### 7. Explored a wrapped cryptographic materials provider
- Built `Table3` using a `WrappedCryptographicMaterialsProvider`, simulating wrapping and signing keys retrieved from an on-premises key management system (generated locally for the lab, not a real external connection)
- Encrypted 1,345 low-revenue vegetable sales records the same way, with the same primary-key-readable, rest-encrypted pattern

### 8. Encrypted local files with the AWS Encryption SDK
- Created a third key, **AWS_Encryption_SDK_Key**
- Wrote Python using `aws_encryption_sdk.EncryptionSDKClient` with `CommitmentPolicy.REQUIRE_ENCRYPT_REQUIRE_DECRYPT` to read plaintext recipe files and write encrypted versions prefixed `[ENCRYPTED]-`
- Verified the output files contained binary ciphertext instead of readable text

## Key takeaways
- Key **administration** and key **usage** are separate IAM responsibilities, and should usually be assigned to different principals
- Changing a bucket's default encryption never retroactively re-encrypts existing objects — that needs a separate remediation pass
- ABAC let me change an access decision by editing one resource tag, with zero changes to IAM policy or role assignment — powerful, but it means tag-editing permissions need to be governed as carefully as the policies that reference them
- DynamoDB server-side encryption is transparent to authorized callers; client-side encryption (direct KMS or wrapped providers) hides attribute values even from someone with direct table access, at the cost of extra application complexity and key-management responsibility
- Encrypted data is only as recoverable as the key protecting it — losing the key means losing the data permanently

## Tools
AWS KMS, Amazon S3, Amazon EC2, AWS IAM, Amazon DynamoDB, DynamoDB Encryption SDK, AWS Encryption SDK, Python (boto3)

---
*Completed as an AWS Builder Lab (Using Data Encryption in AWS). The on-premises key retrieval in Task 5 is simulated by the lab, not a real external system connection.*
