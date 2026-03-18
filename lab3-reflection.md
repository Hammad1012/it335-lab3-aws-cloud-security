# Lab 3 Reflection – AWS Cloud Security Fundamentals
## Capital One Breach Case Study Connection


## Summary of Configurations

### 1. Identity and Access Management (IAM)
I verified that MFA is enabled on the root account which stops the risk of
root credential compromise via password alone. So then I created a restricted IAM user
`lab-student-user` and group `LabUsers` with a custom policy `LabStudentPolicy`
granting only read only access to S3, EC2, and IAM list operations. I verified
the restrictions by signing in as the IAM user and confirming that privileged
actions were denied as seen in the screenshots.

### 2. Secure Storage (S3)
I created an S3 bucket `it335-lab3-hammad-secure` with "Block all public access"
enabled across all four settings. No bucket policy was added, meaning no
external entity can access bucket contents. The bucket exists solely as a
private, internal storage resource.

### 3. Secured Compute (EC2 / Security Groups / IMDS)
I launched a Free Tier t2.micro Amazon Linux instance with a custom security
group `lab3-restricted-sg` that restricts SSH (port 22) access exclusively
to my personal IP address (/32 CIDR). I configured the instance to use IMDSv2
only, requiring session tokens for all metadata requests.

### 4. Visibility and Governance (Logging / Billing)
I navigated to the Billing & Cost Management dashboard to confirm Free Tier
usage and verify no unexpected charges as you can see in the screenshot submitted. I located and reviewed the CloudTrail
console, confirming API activity logging is active across the account.


## Explicit Capital One Breach Connections

### IAM - Overly Permissive Credentials
We talked about this in class, so in the Capital One breach, a misconfigured WAF
allowed SSRF requests to reach the EC2 Instance Metadata Service, which handed
back temporary IAM role credentials. Those credentials had a way too broad IAM
policy, giving the attacker ListBuckets and GetObject permissions across
basically all S3 resources. What I did in my lab directly addresses this by
applying least privilege IAM: the LabStudentPolicy only grants the minimum
actions needed. Even if credentials were stolen, the damage would be limited
to read only access on a small set of services, not the ability to steal
100 million customer records.

### S3 - Public Access Block Prevents Data Exfiltration
Capital One's customer data was stored in S3 buckets that the attacker's
stolen credentials could reach at scale. The buckets weren't publicly exposed,
but the role credentials had GetObject rights. Enabling "Block all public
access" adds an important safety layer, even if a bucket policy was
accidentally set up wrong to allow public access, this account-level setting
would still block those requests. This control, combined with tighter IAM
policies, would have made it much harder for the attacker to pull out that
much data.

### EC2 - SSRF and IMDSv2
The whole credential theft chain in the Capital One incident relied on IMDSv1,
which gives back credentials in response to any basic HTTP GET request,
including forged requests coming through the WAF. IMDSv2 stops this completely, so this basically
requires a session token that has to be requested using a PUT request with
a time limited header, which SSRF attacks can't get through a normal HTTP
proxy. So by setting my EC2 instance to use IMDSv2 only, I cut off that
whole attack path. On top of that, locking down the security group to just
my personal IP (/32) means the instance can't even be reached from the
public internet at all including from anything an attacker controls.

### Logging - Reducing the Window of Opportunity
Capital One's attacker went unnoticed for around three months. CloudTrail was
running but the alerts weren't set up to catch unusual API activity like a
huge number of S3 GetObject calls coming from a strange IP. My lab confirmed
CloudTrail is active, meaning all API calls are being recorded. If that's
paired with CloudWatch Alarms or AWS GuardDuty, unusual patterns like thousands
of GetObject calls in just a few minutes would set off alerts right away.
Keeping an eye on billing also helps, a sudden spike in S3 data transfer
costs can be an early sign that something is wrong, even before anyone looks
at the security logs.


## Critical Analysis

The easiest control to get right was S3 Block Public Access, AWS now has it
turned on by default, so the secure option is already the easy option. The
control most likely to go wrong was Security Groups, it's really easy to just
set SSH to 0.0.0.0/0, especially when you're testing something quickly, and
a lot of tutorials online actually tell you to do that. This kind of shortcut
is exactly how real world breaches happen not because of some complex attack,
but because someone chose convenience over security.

But honestly the most important takeaway is that the Capital One incident
wasn't just a technical failure. The tools to prevent it were already there,
IAM leastprivilege policies, IMDSv2, CloudTrail alerting. The real failure
was organizational no culture of regularly checking security settings, no
automatic enforcement of IAM boundaries, and no real time alerts for unusual
API behavior. Technology by itself can't fully protect an organization and without
processes that require regular IAM audits, automated Config rules to catch
setting changes, and a security culture that takes misconfiguration just as
seriously as malware, even good tools will eventually fail. This lab showed me
that foundational controls only actually work when they're part of everyday
habits, not just switched on once and left alone.
