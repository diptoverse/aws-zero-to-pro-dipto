# **Day 1: Hands-on Labs**

### Re-runnable experiments for Day 1 concepts

All labs are free and read-only. Nothing is created in AWS.

---

## Before you start

Labs 3, 4 and 5 need an active AWS session. Log in once from Terminal (any folder):

```zsh
aws sso login
```

---

## Lab 1: Measure latency to Regions

**Where:** Mac Terminal, any folder. No AWS login needed.

**What:** Time a connection to each Region's public endpoint, 5 times per Region.

**Why:** Latency depends on the network route, not on distance on a map, so you have to measure it.

**Command:**

```zsh
for r in ap-south-1 ap-south-2 ap-southeast-1; do
  for i in 1 2 3 4 5; do
    printf "%s  " "$r"
    curl -o /dev/null -s -w "%{time_connect}\n" "https://dynamodb.$r.amazonaws.com/ping"
  done
done
```

**How to read it:** Values are in seconds (`0.050` = 50 ms). Take the median of the 5 runs. The first run is usually slower because it includes the DNS lookup.

**Result (from Dhaka):**

| Region | Median latency |
|---|---|
| Mumbai (`ap-south-1`) | ~50 ms |
| Hyderabad (`ap-south-2`) | ~49 ms |
| Singapore (`ap-southeast-1`) | ~57 ms |

Latency is effectively tied, so it doesn't decide the Region.

---

## Lab 2: List Availability Zones using the Console

**Where:** Browser, AWS Console. Sign in through the IAM Identity Center access portal.

**What:** Find the Mumbai AZs by clicking through the Console.

**Steps:**

1. Open the access portal and choose **AdministratorAccess**.
2. In the top-right corner, set the Region to **Asia Pacific (Mumbai)**.
3. Search for **EC2** and open it.
4. On the EC2 Dashboard, scroll to **Zones**.

**Result:**

| Zone name | Zone ID |
|---|---|
| ap-south-1a | aps1-az1 |
| ap-south-1b | aps1-az3 |
| ap-south-1c | aps1-az2 |

---

## Lab 3: List Availability Zones using the CLI

**Where:** Mac Terminal, any folder, after `aws sso login`.

**What:** Get the same data as Lab 2 with a single command.

**Command:**

```zsh
aws ec2 describe-availability-zones --query "AvailabilityZones[].[ZoneName,ZoneId]" --output table
```

**How to read it:** `--query` picks the fields to show, and `--output table` formats them as a table.

**Result:** The same 3 AZs and Zone IDs as the Console. Both use the same API.

---

## Lab 4: List Availability Zones using the SDK (Python / boto3)

**Where:** Mac Terminal, inside a project folder such as `sdk-test/`, after `aws sso login`. Keep this folder outside the git repository.

**What:** Make the same API call from Python code.

**Step 1: Create the folder and an isolated Python environment** (first time only)

```zsh
mkdir -p ~/engg/devops/bongodev/cloud-engg/sdk-test
cd ~/engg/devops/bongodev/cloud-engg/sdk-test
python3 -m venv .venv
source .venv/bin/activate
pip install boto3
```

When the environment is active, your prompt shows `(.venv)`. Next time, just go to the folder and run `source .venv/bin/activate`.

**Step 2: Create `azs.py` in that folder**

```python
import boto3

ec2 = boto3.client("ec2", region_name="ap-south-1")
response = ec2.describe_availability_zones()

for az in response["AvailabilityZones"]:
    print(az["ZoneName"], az["ZoneId"])
```

**Step 3: Run it, then deactivate the environment**

```zsh
python azs.py
deactivate
```

**How to read it:** boto3 finds your SSO login through the `AWS_PROFILE` variable, so the code contains no access keys.

**Result:** The same output as Labs 2 and 3. The Console, CLI and SDK are three ways into the same AWS API.

---

## Lab 5: Check who you are logged in as

**Where:** Either of these:

- Mac Terminal, any folder, after `aws sso login`
- CloudShell (the `>_` icon in the Console top bar), which is already logged in

**What:** Confirm your identity before doing any work.

**Command:**

```zsh
aws sts get-caller-identity
```

**Result:** The ARN contains `assumed-role/AWSReservedSSO_AdministratorAccess_.../<user>`. That means temporary credentials, not the root user.


