# **Day 1: Introduction to AWS — Notes**
### Topic: How AWS is built, how it charges, and how you use it

#### What we'll learn:
Regions · Availability Zones · Opt-in Regions · Latency · Services by Region · Global vs regional services · Choosing a Region · On-premises / Cloud / Hybrid · IaaS / PaaS / SaaS · Pricing models · Edge locations · Console / CLI / SDK

---

### 1. Regions
A **Region** is a geographic area where AWS runs data centers, for example **Mumbai (`ap-south-1`)**. Regions are **independent**: a server created in Mumbai exists **only** in Mumbai. If you switch Region in the Console, it seems to "disappear". That's the #1 reason people lose track of resources and get surprise bills.
**Why they exist:** to be close to users, to keep data inside a country when the law requires it, and to keep a failure in one Region from affecting the others.

### 2. Availability Zones (AZs)
Each Region has several **AZs**: separate data centers with their **own power and network**, connected by fast links. If the Region is a city, the AZs are **separate buildings** in it. When one goes down, the others keep running.
**Why it matters:** run your app in **2+ AZs** and it survives an AZ failure (**high availability**).
**Note:** AZ names like `ap-south-1a` map to different physical AZs in each account. The **AZ ID** (e.g. `aps1-az1`) is the real physical location.

### 3. Opt-in Regions
Regions launched after 2019 (e.g. **Hyderabad**) are **off by default** and must be enabled first.
**Why:** security. Nobody can create resources (e.g. an attacker running crypto-mining) in a Region you never turned on.

### 4. Latency
**Latency** is the time data takes to reach a server and come back, in milliseconds. Lower means a faster app.
**The closest Region on a map isn't always the fastest**, because traffic follows cables and ISP routes. So **measure, don't guess**: time a request to each Region's public AWS endpoint several times and take the **middle value**. The first attempt is usually slower because it includes the DNS lookup.

### 5. Why services differ between Regions
AWS launches new services in large, older Regions first. Newer Regions may be **missing services or features**. Check the AWS *Services by Region* page before choosing.

### 6. Global vs regional services
- **Regional:** EC2, RDS, Lambda, VPC, DynamoDB live in **one Region**.
- **Global:** IAM, Billing, Route 53, CloudFront, Organizations apply to the **whole account**. The Console shows "Global" for them.
- **S3:** the console lists **all** buckets, but **each bucket lives in one Region**.
**Habit:** check the Region in the top-right corner before creating anything.

### 7. How to choose a Region
Weigh these in order: **Compliance → Latency → Services → Price.**

**Worked example from Dhaka:**

| | Mumbai | Hyderabad | Singapore |
|---|---|---|---|
| Latency | ~50 ms | ~49 ms | ~57 ms |
| AZs | 3 | 3 | 3 |
| Services needed | ✅ | ✅ | ✅ |
| `t3.micro`/month | $8.18 | $8.18 | $9.64 |
| Enabled by default | ✅ | ❌ | ✅ |

**Decision: Mumbai (`ap-south-1`).** Latency, AZs and services are tied. Mumbai wins on **price**, **maturity**, and being **enabled by default**.

### 8. On-premises vs cloud vs hybrid
- **On-premises:** you **buy your own servers**. Full control, but high upfront cost, slow to expand, and you must buy enough for **peak** traffic.
- **Cloud:** you **rent from AWS** on demand. Ready in minutes, pay only for what you use, scale up and down (**elasticity**). Spending shifts from **CapEx → OpEx**.
- **Hybrid:** **both, connected** through VPN or Direct Connect. E.g. a bank keeps records on-premises and runs its app on AWS.

**Example:** an Eid sale brings 10× traffic for 2 weeks. On-premises, those extra servers sit idle the other 50 weeks. In the cloud, you scale up and back down and pay only for those 2 weeks.

### 9. IaaS vs PaaS vs SaaS
The difference is **how much the provider manages**.
- **IaaS:** you get a **virtual server** and manage everything on it → **EC2** *(an empty flat)*
- **PaaS:** you bring **only code and data** → **Lambda, RDS, Elastic Beanstalk** *(a furnished flat)*
- **SaaS:** you just **use the software** → **Gmail, Zoom** *(a hotel room)*

**Shared Responsibility:** AWS secures **the cloud itself** (hardware, data centers). You secure **what you put in the cloud** (data, IAM, the OS on EC2).

### 10. Pricing models
**Why it matters:** estimate the cost **before** you build. The same server can cost up to **4× more or less** depending on how you pay.

**Example:** one `t3.micro` in Mumbai, running 24/7:

| Model | What it means | Monthly | Best for |
|---|---|---|---|
| **On-Demand** | Pay per second, no commitment | $8.18 | Learning, testing, new apps |
| **1-Year Reserved** | Commit for 12 months | ~$5.18 (−37%) | Steady production |
| **3-Year Reserved** | Commit for 36 months | ~$3.00 (−63%) | Long-term core systems |
| **Spot** | Spare capacity; **can be taken back with 2 min warning** | ~$2.04 (−75%) | Batch jobs, CI. **Never live apps** |

Also: **Savings Plans** give a Reserved-like discount with more flexibility. A **Dedicated Host** is a whole physical server, used for licensing or compliance.

**How to choose:** Can it be interrupted? → Spot. Running only a few hours a day? → On-Demand. Running 24/7 for years? → Reserved / Savings Plans.
**Hidden cost:** data **going out** to the internet (100 GB/month free, then ~$0.11/GB) can cost more than the server itself.

### 11. Edge locations
**Edge locations store temporary copies of content close to users, so nearby users don't have to fetch it from the main Region.**

**How it works:** a user in Dhaka requests an image → the request goes to the **nearest edge**. The first time, the edge fetches it from the origin in Mumbai and **keeps a copy**. After that, nearby users get the copy instantly. The copy **expires** after a set time (TTL) and is then refreshed.

**What's stored there:** ✅ temporary copies of images, videos, CSS/JS. ❌ **Not** user accounts, orders or database records, which stay in the Region.
**Used by:** CloudFront (CDN), Route 53 (DNS), WAF/Shield (blocking attacks at the edge).
⚠️ Never cache personal pages, or one user may see another user's data.

### 12. Access methods: Console, CLI, SDK
All three call the **same AWS API** with the **same login and permissions**, and they're **all free**. You pay for the resources you create, not for how you created them.
- **Console:** click through the website. Good for **learning and exploring**. *CloudShell* (`>_` in the Console) gives you a ready-to-use, already-logged-in Terminal in the browser.
- **CLI:** `aws <service> <action>` commands in Terminal. Good for **repeated tasks and scripts**. Log in with `aws sso login`.
- **SDK:** AWS libraries for code (e.g. **boto3** for Python). Good for **apps that use AWS**. The same API calls, written as functions.

**Security rule:** use SSO or IAM roles. **Never put access keys in code.**

