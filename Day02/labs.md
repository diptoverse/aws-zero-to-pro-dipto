# Day 2 · IAM Labs

**Outcome:** a least-privilege CLI user, and proof of how AWS decides Allow vs Deny.

```text
s3-cli-user            CLI only, access key
  └─ S3Admins          group
       └─ S3-OneBucket-Allowed    customer managed policy
```

**Setup:** AWS CLI installed, and console access with an admin identity (I use IAM Identity Center).

---

## Lab 1 · Inspect the identity you already use

**Goal:** see that an SSO login is really a role.

**Steps:** Console → **IAM** (not *IAM Identity Center*) → **Roles**

**Expected:**

```text
AWSReservedSSO_AdministratorAccess_xxxx
Trusted entity: Identity Provider (SAML)
```

**Why:** an SSO login means assuming a role. STS issues the temporary credentials, and "Trusted entity" is the role's trust policy.

---

## Lab 2 · Create the group

**Goal:** permissions live on a group, not on users.

**Steps:** IAM → **User groups** → **Create group**

1. Name: `S3Admins`
2. Attach: `AmazonS3FullAccess`
3. Create

**Expected:** Permissions tab shows `AmazonS3FullAccess` with Type **AWS managed**.

---

## Lab 3 · Create the CLI user

**Goal:** a user with **no console password**.

**Steps:** IAM → **Users** → **Create user**

1. Name: `s3-cli-user`
2. Leave **"Provide user access to the AWS Management Console"** unchecked
3. Permissions → **Add user to group** → `S3Admins` → Create

**Expected:**

```text
ARN               arn:aws:iam::<account>:user/s3-cli-user
Attached via      Group S3Admins
Console access    Disabled
```

**Why:** no password means no login profile, so the console door is closed. Credentials decide *how* you get in; policies decide *what* you can do.

---

## Lab 4 · Create an access key

**Goal:** long-term credentials for the CLI.

**Steps:** `s3-cli-user` → **Security credentials** → **Create access key**

1. Use case: **Command Line Interface (CLI)** → confirm → Next
2. Tag: `laptop-cli` → Create
3. Save the ID and secret in a password manager (the secret is shown only once)

**Expected:** the key ID starts with `AKIA`, which means long-term. `ASIA` would mean temporary (STS).

---

## Lab 5 · Configure the CLI profile

**Goal:** store the key on the laptop under a name.

Run:

```bash
aws configure --profile s3-cli
```

Answer the prompts:

```text
AWS Access Key ID        AKIA...
AWS Secret Access Key    ********
Default region name      ap-south-1
Default output format    json
```

Verify (keys are masked):

```bash
aws configure list --profile s3-cli
```

**Why:** a profile is a named set of credentials, stored in `~/.aws/credentials` with settings in `~/.aws/config`. Before this step, `--profile s3-cli` fails with *"The config profile (s3-cli) could not be found"*.

---

## Lab 6 · Prove identity and implicit deny

**Goal:** the "done when" check.

**Who am I?**

```bash
aws sts get-caller-identity --profile s3-cli
```

```json
{
  "UserId": "AIDA...",
  "Account": "<account>",
  "Arn": "arn:aws:iam::<account>:user/s3-cli-user"
}
```

**Something not allowed:**

```bash
aws ec2 describe-instances --profile s3-cli
```

```text
UnauthorizedOperation
User: arn:aws:iam::<account>:user/s3-cli-user
is not authorized to perform: ec2:DescribeInstances
because no identity-based policy allows the ec2:DescribeInstances action
```

**Why:**

- `GetCallerIdentity` needs no permissions, so it only checks that the keys are valid.
- *"no identity-based policy allows"* means **implicit deny**.
- An explicit deny would read *"with an explicit deny in an identity-based policy"*.

---

## Lab 7 · Read `AmazonS3FullAccess`

**Steps:** IAM → **Policies** → `AmazonS3FullAccess` → **JSON**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*", "s3-object-lambda:*"],
      "Resource": "*"
    }
  ]
}
```

**Why it's not least privilege:** `Resource: "*"` covers **every** bucket, so the user can delete or expose any of them. Line-by-line breakdown: `notes.md` §8.

---

## Lab 8 · Stretch: allow one bucket only

### 8.1 Create two test buckets

Do this while the user still has full S3 access. Bucket names are globally unique, so add your name and the date.

```bash
aws s3 mb s3://dipto-iam-allowed-20261102 --profile s3-cli
```

```bash
aws s3 mb s3://dipto-iam-forbidden-20261102 --profile s3-cli
```

```bash
aws s3 ls --profile s3-cli
```

```text
2026-11-02 06:12:53  dipto-iam-allowed-20261102
2026-11-02 06:13:08  dipto-iam-forbidden-20261102
```

### 8.2 Write the policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTheBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::dipto-iam-allowed-20261102"
    },
    {
      "Sid": "ReadWriteDeleteFiles",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::dipto-iam-allowed-20261102/*"
    }
  ]
}
```

| Action type | Resource |
|---|---|
| Bucket: `ListBucket`, `DeleteBucket` | `arn:aws:s3:::bucket` |
| Object: `GetObject`, `PutObject`, `DeleteObject` | `arn:aws:s3:::bucket/*` |

No Deny is needed: anything not listed is implicitly denied.

**Common mistakes:**

| Wrong | Problem |
|---|---|
| `"Effect": "Deny"`, `"Action": "*"` | Blocks every AWS action |
| `"Resource": "arn:aws:s3:::s3"` | Points to a bucket named `s3` |
| `"Resource": "arn:aws:s3:::s3:*"` | Invalid object pattern, use `bucket/*` |

### 8.3 Create the policy

IAM → **Policies** → **Create policy** → **JSON** → paste → Next → name it `S3-OneBucket-Allowed` → Create

**Expected:** Type **Customer managed**, Used as **None**. A policy does nothing until it's attached.

### 8.4 Swap the policies on the group

IAM → **User groups** → `S3Admins` → **Permissions**

1. Attach `S3-OneBucket-Allowed`
2. Remove `AmazonS3FullAccess`

Attach first, then remove, so access never drops to zero mid-change.

### 8.5 Test every edge

IAM changes can take a few seconds to apply.

**A · List files in the allowed bucket** → ✅ allowed by `ListBucket` (blank output = empty bucket)

```bash
aws s3 ls s3://dipto-iam-allowed-20261102 --profile s3-cli
```

**B · Upload a file** → ✅ allowed by `PutObject`

```bash
echo "hello" > test.txt
aws s3 cp test.txt s3://dipto-iam-allowed-20261102/ --profile s3-cli
```

```text
upload: ./test.txt to s3://dipto-iam-allowed-20261102/test.txt
```

**C · List the forbidden bucket** → ❌ not in Resource, so implicit deny

```bash
aws s3 ls s3://dipto-iam-forbidden-20261102 --profile s3-cli
```

**D · List all buckets** → ❌ `ListAllMyBuckets` isn't allowed

```bash
aws s3 ls --profile s3-cli
```

```text
no identity-based policy allows the s3:ListAllMyBuckets action
```

**E · Delete the file** → ✅ allowed by `DeleteObject`

```bash
aws s3 rm s3://dipto-iam-allowed-20261102/test.txt --profile s3-cli
```

```text
delete: s3://dipto-iam-allowed-20261102/test.txt
```

**F · Delete the bucket** → ❌ `DeleteBucket` isn't allowed

```bash
aws s3 rb s3://dipto-iam-allowed-20261102 --profile s3-cli
```

```text
no identity-based policy allows the s3:DeleteBucket action
```

**Takeaways:**

- `ListBucket` (files in one bucket) is not `ListAllMyBuckets` (all bucket names, needs `Resource: "*"`)
- `DeleteObject` (a file) is not `DeleteBucket` (the bucket itself)
- To let a console user see bucket names, add this statement:

```json
{
  "Effect": "Allow",
  "Action": "s3:ListAllMyBuckets",
  "Resource": "*"
}
```

---

## Lab 9 · Cleanup

`s3-cli-user` can no longer delete buckets, so use the admin console session:

1. S3 → delete `dipto-iam-allowed-20261102`
2. S3 → delete `dipto-iam-forbidden-20261102`

Remove the local test file:

```bash
rm test.txt
```

**Kept for later days:** `s3-cli-user`, `S3Admins`, `S3-OneBucket-Allowed`, and the `s3-cli` profile.

---

## Command reference

| Command | Purpose |
|---|---|
| `aws configure --profile <p>` | Save keys and region |
| `aws configure list --profile <p>` | Show a profile (masked) |
| `aws configure list-profiles` | List all profiles |
| `aws sts get-caller-identity --profile <p>` | Who am I? |
| `aws s3 mb s3://<bucket>` | Make a bucket |
| `aws s3 ls` | List buckets |
| `aws s3 ls s3://<bucket>` | List files |
| `aws s3 cp <file> s3://<bucket>/` | Upload |
| `aws s3 rm s3://<bucket>/<key>` | Delete a file |
| `aws s3 rb s3://<bucket>` | Remove a bucket |
