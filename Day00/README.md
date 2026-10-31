# **Day 0: AWS Account Setup, Security & Tools**
### Topic: Secure your account and prepare your environment before Day 1
#### Guidelines:
- Understand Root user vs IAM user vs IAM role vs IAM Identity Center.
- Understand AWS account plans (Free vs Paid), credits and Free Tier limits.
- Learn why budgets and alerts are warnings, not spending limits.
- Use temporary credentials for the CLI instead of long-lived access keys.

#### Challenge (step by step):
1. **Secure the root user**: enable MFA on root and don't use it for daily work.
   - *Why:* root has unlimited access that no policy can restrict. Use it only for account, billing and recovery tasks.
2. **Check your account plan and credits**: Billing and Cost Management → Home and → Credits.
   - *Why:* a Free plan can't be charged but closes after 6 months or once credits run out. A Paid plan bills your card for anything beyond credits and free limits.
3. **Review your bills and look for leftover resources**: Billing → Bills (check past months), then **EC2 Global View**.
   - *Why:* start from a known $0 baseline. The EC2 console shows one Region at a time, so forgotten resources elsewhere are easy to miss.
4. **Enable Free Tier alerts and CloudWatch billing alerts**: Billing → Billing preferences → Alert preferences.
   - *Why:* you get an email near a free limit. CloudWatch billing metrics (us-east-1 only) are needed for billing alarms later.
5. **Create a monthly cost budget** (for example $5) with email alerts at 50% / 80% / 100% of actual cost (Budgets → Create budget → Customize).
   - *Why:* an early warning for any charge. ⚠️ A budget only alerts you. It does **not** stop resources or cap your bill.
6. **Enable IAM Identity Center** in a Region close to you, as a **Single-Region** instance with an **AWS owned key**.
   - *Why:* the default Multi-Region option uses customer managed KMS keys, which cost about $1/key/month. Single-Region is free.
   - ⚠️ **Free plan users:** enabling it creates an AWS Organization, which **upgrades the account to the Paid plan and ends your Free Tier credits**. Decide before you click.
7. **Create a user, an `AdministratorAccess` permission set and an account assignment**, then accept the email invite and register MFA (a passkey is best).
   - *Why:* daily work uses temporary role credentials from the access portal instead of root.
8. **Activate IAM user/role access to Billing**: as root, Account page → *IAM user and role access to Billing information* → Activate.
   - *Why:* until this setting is on, even admins get Access Denied on billing pages.
9. **Set the Console's default Region**: ⚙️ Settings → Localization and default Region.
   - *Why:* resources are created in the selected Region. The wrong Region means resources you forget about.
10. **Install AWS CLI v2** from the official installer and verify it.
    - *Why:* AWS supports v2, and checking the installer's signature confirms it's genuine before running it with `sudo`.
11. **Configure the CLI with SSO** instead of access keys.
    - *Why:* the credentials expire automatically, and no `~/.aws/credentials` file can leak.
12. **Fork this repo, clone it, and add a `.gitignore`** before your first commit.
    - *Why:* keep keys and secrets out of a public repo.

#### Commands (macOS):
```zsh
# Install and verify the CLI
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
pkgutil --check-signature AWSCLIV2.pkg      # expect Team ID 94KV3E626L (Amazon)
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version                               # expect aws-cli/2.x

# Connect the CLI to IAM Identity Center
aws configure sso                           # portal URL, SSO Region, profile name
echo 'export AWS_PROFILE=<profile-name>' >> ~/.zshrc && source ~/.zshrc

# Verify
aws sts get-caller-identity                 # ARN should contain assumed-role/AWSReservedSSO_...
aws configure list                          # credential type should be "sso"
```

#### Minimum `.gitignore`:
```gitignore
.aws/
credentials
*.pem
*.key
.env
.env.*
!.env.example
*.tfstate
*.tfstate.*
*.tfvars
.DS_Store
```

#### Daily routine:
```zsh
aws sso login                  # start (or when "Token has expired")
aws sts get-caller-identity    # who am I?
aws configure get region       # where am I?
# ... work, then delete billable resources ...
aws sso logout                 # finish
```

#### Key lessons:
- A default setting isn't always free. Read the cost notes before you click **Enable**.
- Budgets and alerts only warn you. Cleaning up is what saves money.
- Before creating anything, check the **identity** and **Region** in the top-right of the Console.
- Never commit or share passwords, MFA codes, access keys or unmasked account IDs.

#### Reference Links:
- [Choosing an AWS Free Tier plan](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Configure the CLI with IAM Identity Center](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
