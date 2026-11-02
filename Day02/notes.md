# Day 2 · IAM Basics

> **Goal:** a least-privilege identity for the CLI, and knowing exactly how AWS decides allow or deny.

Hands-on steps, commands and outputs: [`labs.md`](labs.md)

---

## IAM: Identity and Access Management

IAM is the security desk of an AWS account. Every request, from a console click to a CLI command or an app's API call, has to pass two checks:

1. **Who are you?** (authentication)
2. **Are you allowed to do this?** (authorization)

### Why IAM matters

- **The root user can do everything**, including closing the account. Using it daily is like handing out the building's master key. IAM gives you safer identities for daily work.
- **Every action is traceable.** Each person or app has its own identity, so logs show exactly who did what.
- **Damage stays small.** Each identity gets only what it needs, so a leaked credential can't take down the whole account.
- **Access is easy to change.** Add or revoke one person's access without touching anyone else.
- **There's no way around it.** Every action in AWS is an API call, and every API call must be signed with credentials and permitted by a policy.

IAM is a **global** service: users, groups, roles and policies apply in every Region.

---

## The four building blocks

### 👤 User

A **permanent identity** for one person or one app.

- Logs in with a **password** (console) and/or **access keys** (CLI, SDK).
- Credentials are **long-lived**. They work until you rotate or delete them, so a leaked key stays dangerous.
- A brand-new user with no policies **can do nothing**.
- No password means **no console access**. A CLI-only user needs only access keys.

> **Example:** developer Rahim gets the IAM user `rahim`. When a file gets deleted, the logs show *"rahim deleted it"*.

### 👥 Group

A **collection of users** who need the same permissions.

- Attach a policy to the group once, and every member inherits it.
- A group is **not an identity**. Nobody logs in as a group.
- A user can be in many groups. Groups can't contain other groups or roles.

> **Example:** five developers need S3. Put them in the `S3Admins` group with one policy. A new developer joins? Add them. Someone leaves? Remove them. Nobody else is affected.

### 🎭 Role

A set of permissions that someone **borrows for a short time**.

Think of it as a **hotel key card**: reception hands it out, it opens only your room, and it stops working at checkout. A user's access key is more like a house key, which works until you change the lock.

**Why roles exist:** every API call needs credentials. An EC2 server or Lambda function isn't a person, so it can't have a password, and putting access keys in code is risky because they leak and never expire. A role gives it **temporary credentials** that expire on their own, with no keys stored anywhere.

**A role has two policies:**

| Policy | Question it answers | Example |
|---|---|---|
| Trust policy | **Who** can assume this role? | "Only the EC2 service" |
| Permission policy | **What** can it do? | "Read files from `myapp-photos`" |

**Who assumes roles:**

| Who | Example |
|---|---|
| AWS services | An app on EC2 reads from S3; a Lambda writes to DynamoDB |
| IAM users | Rahim is read-only, then switches to an admin role for one hour, like `sudo` |
| Other AWS accounts | The Dev account's team gets read access to Production |
| SSO logins | An employee signs in with their company account and gets a role, with no IAM user needed |

> **Example:** a photo app on EC2 needs to read from S3. Instead of storing an access key on the server, you attach the role `PhotoApp-EC2-Role`. The app gets one-hour credentials automatically. If the server is hacked, those credentials expire soon and only reach one bucket.

> **Real life:** my console login `AdministratorAccess / dipto builder` comes from IAM Identity Center. Behind the scenes, I'm assuming the role `AWSReservedSSO_AdministratorAccess_...`.

### 📜 Policy

A **JSON document** that says what is allowed or denied, and on what.

- It's the only source of permissions. Users, groups and roles have **no power without one**.
- It does nothing until it's attached.
- It comes in three kinds:

| Kind | Created by | Reusable |
|---|---|---|
| AWS managed | AWS, e.g. `AmazonS3FullAccess` | ✅ |
| Customer managed | You, e.g. `S3-OneBucket-Allowed` | ✅ |
| Inline | You, embedded in one identity | ❌ deleted with it |

- **Identity-based** policies attach to a user, group or role. **Resource-based** policies attach to a resource, such as an S3 bucket policy or a role's trust policy, and must name **who** (a `Principal`).

---

## How they connect

```text
Policy  ── attached to ──►  User · Group · Role
User    ── member of   ──►  Group
Service · User · Account · SSO  ── assumes ──►  Role
Group   ✗ cannot assume or hold a role
```

- Policies give permissions to users, groups and roles. Nothing has power without a policy.
- Groups contain users only, never roles or other groups.
- Roles are *assumed*, never attached to a group.

**How a user assumes a role.** Both sides must agree:

1. The user's policy (often from a group) allows `sts:AssumeRole` on that role's ARN.
2. The role's trust policy trusts that user or its account.

Then the user switches role in the console, or uses a CLI profile with `role_arn`, and works with the role's permissions until the session expires. A trust policy can't name a group, so group-wide access is granted through step 1.

**Rule of thumb:** humans go into groups (or better, SSO). AWS services use roles.

---

## Reading a policy

Every statement answers up to four questions.

### Effect: allow or deny?

Only two values: `Allow` or `Deny`. A Deny is used as a guardrail for things that must never happen, such as deleting production backups.

### Action: which API call?

Written as `service:Operation`, and each one matches a real API call.

- `s3:GetObject` downloads a file, `s3:PutObject` uploads one.
- `s3:*` means every S3 action. `s3:Get*` means every action starting with *Get*.
- You can list several: `["s3:GetObject", "s3:PutObject"]`.

### Resource: on which thing?

Written as an ARN (see below).

- `arn:aws:s3:::my-bucket` is the bucket itself.
- `arn:aws:s3:::my-bucket/*` is every file inside it.
- `*` means every resource. Use it only when you truly mean everything.

Action says *"can open doors"*. Resource says *which doors*.

### Condition: under what circumstances? (optional)

Extra rules about *when* the statement applies, such as only from a specific IP range, only when MFA is used, or only in one Region.

> **Example:** allow deleting buckets only if the user signed in with MFA.

### The shape, for reference

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadPhotosWithMFA",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "Bool": { "aws:MultiFactorAuthPresent": "true" }
      }
    }
  ]
}
```

- `Version` is always `2012-10-17`. It's the policy language version, not a date you choose.
- `Statement` is a list, so one policy can hold many rules.
- `Sid` is an optional label for humans.

---

## ARN: Amazon Resource Name

The **unique address** of anything in AWS. Like a postal address, it points to exactly one thing.

```text
arn : partition : service : region : account-id : resource
```

| Resource | ARN |
|---|---|
| IAM user | `arn:aws:iam::111122223333:user/s3-cli-user` |
| IAM role | `arn:aws:iam::111122223333:role/Prod-Admin-Role` |
| EC2 instance | `arn:aws:ec2:ap-south-1:111122223333:instance/i-0abc123` |
| S3 bucket | `arn:aws:s3:::myapp-photos` |
| S3 files | `arn:aws:s3:::myapp-photos/*` |

Some parts are empty for a reason:

- **IAM** has no Region because it's global.
- **S3** has no Region and no account because bucket names are unique across all of AWS. That's why S3 ARNs start with `arn:aws:s3:::`.

---

## STS: Security Token Service

**STS issues temporary credentials.** If a role is the hotel key card, STS is the reception desk that hands it out and sets when it expires.

Every time a role is assumed, whether by EC2, Lambda, a user switching roles or an SSO login, STS is called behind the scenes. For EC2 and Lambda it happens automatically, and you never handle the keys.

### Long-lived vs temporary credentials

| | IAM user keys | STS credentials |
|---|---|---|
| Starts with | `AKIA` | `ASIA` |
| Parts | Key ID + secret | Key ID + secret + **session token** |
| Expires | Never, unless rotated | Automatically, minutes to hours |
| If leaked | Works until someone deletes it | Stops working soon |

### Key STS calls

| Call | Used for |
|---|---|
| `AssumeRole` | Switching roles, cross-account access |
| `AssumeRoleWithWebIdentity` | Web or OIDC logins, e.g. GitHub Actions deploying to AWS |
| `AssumeRoleWithSAML` | Corporate SSO |
| `GetSessionToken` | Temporary credentials for yourself, often with MFA |
| `GetCallerIdentity` | "Who am I?" It needs no permissions and can't be denied |

### Credentials vs permissions

- **Credentials** decide *how* you get in: a password for the console, access keys for the CLI.
- **Policies** decide *what* you can do once you're in.

---

## How AWS decides

All applicable policies are combined (the user's own plus every group it belongs to, or the role's), then three rules apply:

| Priority | Rule |
|---|---|
| 1 | An **explicit Deny** anywhere wins, always |
| 2 | Otherwise, an **explicit Allow** permits the request |
| 3 | Otherwise, it's **implicitly denied** (the default) |

```text
                 Request arrives
                        │
                        ▼
        Does any policy explicitly Deny it?
             │                     │
            yes                    no
             │                     ▼
             ▼        Does any policy explicitly Allow it?
          DENIED            │                  │
                           yes                 no
                            │                  │
                            ▼                  ▼
                         ALLOWED     DENIED (implicit, by default)
```

| | Implicit deny | Explicit deny |
|---|---|---|
| Happens when | No policy allows the action | A policy says `"Effect": "Deny"` |
| Can an Allow fix it? | Yes | No |
| Use it for | The normal default | Guardrails |

AWS error messages tell you which one you hit:

- *"no identity-based policy allows…"* means implicit deny
- *"…with an explicit deny…"* means something actively blocked it

In larger setups, other layers can also say no: Service Control Policies (AWS Organizations), permission boundaries, and resource-based policies. An explicit Deny in any of them still wins.

---

## Least privilege

> **Grant only what the job needs.** If a credential leaks, the damage stays small.

A hotel cleaner gets keys to the floors they clean, not to the safe. AWS identities should work the same way.

### Protect the root user

Turn on MFA, never create access keys for it, and use it only for the few tasks that require it.

### Organize with groups

Attach policies to groups, not individual users. There's one place to change permissions, onboarding and offboarding are just adding or removing a member, and audits are simple.

### Use roles for apps and services

Apps on EC2, Lambda or containers get a role, never a stored access key.

### Be specific

Name the exact actions and resources the job needs.

| Too broad | Least privilege |
|---|---|
| `s3:*` on `*` | `s3:GetObject` on `arn:aws:s3:::my-bucket/*` |

### Give only the credentials needed

A CLI-only user gets an access key and no console password.

### Start small, then review

Begin with the minimum and add a permission only when something fails. Use IAM Access Analyzer and "last accessed" data to remove unused permissions, delete unused keys, and rotate the rest.

---

## AmazonS3FullAccess, line by line

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

- `Version` is the policy language version.
- `Statement` holds one rule.
- `Effect: Allow` explicitly allows the actions below.
- `s3:*` allows every S3 action: files, buckets, bucket policies.
- `s3-object-lambda:*` allows every S3 Object Lambda action. That's a separate service where a Lambda changes files as they're read, such as resizing an image on download. `s3:*` doesn't cover it.
- `Resource: *` covers every bucket and file in the account, including future ones.
- There's no `Condition`, so it always applies, and no `Principal`, because it's identity-based.

**Verdict:** it's limited to S3, so EC2 is implicitly denied, but it's too broad inside S3. It can delete or expose any bucket. The stretch policy fixes this by naming one bucket.

---

## Things that trip people up

- **IAM vs IAM Identity Center.** They're different consoles. Identity Center handles SSO; IAM handles users, access keys, roles and policies.
- **Bucket vs files.** `arn:aws:s3:::bucket` is the bucket; `arn:aws:s3:::bucket/*` is the files inside it. Actions target one or the other.
- **Similar names, different actions.** `ListBucket` lists files in one bucket. `ListAllMyBuckets` lists every bucket name. `DeleteObject` deletes a file. `DeleteBucket` deletes the bucket itself.
- **AWS managed policies are often too broad.** `AmazonS3FullAccess` covers every bucket in the account.
- **A policy does nothing until it's attached.**
- **New users can do nothing.** The default is deny.
- **No Deny needed to block something.** If it isn't allowed, it's already denied.

---

## What was built

```text
s3-cli-user          no console access, one access key
  └─ S3Admins        group
       └─ S3-OneBucket-Allowed    one bucket: list, get, put, delete
```

**Done when:**

- `aws sts get-caller-identity --profile s3-cli` shows the user's ARN
- `aws ec2 describe-instances --profile s3-cli` fails with `UnauthorizedOperation`

Both passed. The stretch policy was tested against every edge case. See [`labs.md`](labs.md).

---

## Exam warm-up

**One policy allows `s3:*`, another denies `s3:DeleteBucket`. Can the user delete a bucket?**
No. An explicit Deny overrides any Allow.

**Why attach a policy to a group instead of a user?**
One place to manage permissions. Adding or removing a user from the group grants or revokes access, and changing the policy updates everyone at once.

**When should an app on EC2 use a role instead of an access key?**
Always. The role provides temporary credentials through STS that rotate automatically, so no long-lived key sits on the server or in code.

**Practice exam:** 10 / 10 ✅
