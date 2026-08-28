---
layout: default
title: "Cloud Interview — AWS Core Services, Networking & IAM"
---

# Cloud Interview — AWS Core Services, Networking & IAM

Twenty questions on the AWS services and concepts that actually gate access and connectivity in a real account: VPC networking (subnets, routing, NAT, security groups vs. NACLs), IAM (policy evaluation, roles, SCPs), and S3 (storage classes, durability, performance) — the fundamentals every AWS interview circles back to, distinct from file 01's general scalability/resilience patterns.

### Q1. What's the difference between a public and a private subnet in a VPC, and what actually makes a subnet "public"? {#q1}

**Question:**
What's the difference between a public and a private subnet in a VPC, and what actually makes a subnet "public"?

**Good answer:**
A subnet itself has no "public" or "private" flag — what makes it public is that its route table has a route sending `0.0.0.0/0` (or a specific range) to an Internet Gateway (IGW), and its instances have public IPs to actually receive return traffic. A private subnet's route table instead sends outbound internet-bound traffic to a NAT gateway (or has no internet route at all), so instances can't be reached directly from the internet. The distinction is entirely about routing configuration, not some inherent subnet property — you could take a "private" subnet and make it public just by editing its route table.

**Code example:**
```text
# Public subnet route table
Destination       Target
10.0.0.0/16        local
0.0.0.0/0          igw-0abc123

# Private subnet route table
Destination       Target
10.0.0.0/16        local
0.0.0.0/0          nat-0def456
```

**Follow-up question:**
If an EC2 instance in a "public" subnet has a public IP but its security group has no inbound rules at all, is it reachable from the internet?

**Follow-up good answer:**
No. Being in a public subnet only means the *routing path* exists for internet traffic to reach the ENI — it doesn't override the security group, which is a separate, mandatory layer. With zero inbound rules, nothing (public IP or not) can initiate a connection to that instance; the security group's implicit default-deny still applies. Public subnet placement and public IP assignment are necessary but not sufficient — the security group (and NACL) must also allow the traffic.

**Glossary:**
- **Internet Gateway (IGW)** — a horizontally scaled, redundant VPC component that provides a target in route tables for internet-routable traffic and performs NAT for instances with public IPs.
- **Route table** — a set of rules (routes) used to determine where network traffic from a subnet is directed.

**Mental model:**
Tests whether the candidate understands that AWS networking constructs are compositional (routing + security groups + NACLs all apply independently) rather than treating "public subnet" as a single magic checkbox.

**TL;DR:**
A subnet is "public" only because its route table points internet-bound traffic at an IGW — reachability still requires a public IP and permissive security group rules on top of that.

**References:**
- [Route tables for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)

---

### Q2. Security groups and network ACLs both filter traffic in a VPC — what's the actual mechanical difference between them? {#q2}

**Question:**
Security groups and network ACLs both filter traffic in a VPC — what's the actual mechanical difference between them?

**Good answer:**
Security groups operate at the instance/ENI level and are **stateful**: if you allow inbound traffic, the response is automatically allowed out regardless of outbound rules, and they only support "allow" rules (no explicit deny) — traffic not matching any rule is implicitly denied. Network ACLs operate at the subnet level and are **stateless**: they evaluate inbound and outbound traffic independently, so an ACL that allows inbound traffic on a port does *not* automatically allow the response back out — you must add a matching outbound rule yourself. NACLs also support explicit deny rules and evaluate numbered rules in order, stopping at the first match, whereas security groups evaluate every rule and allow if any rule matches.

**Follow-up question:**
Your app's inbound traffic reaches the instance fine, but responses never make it back to the client, and the security group is wide open. What's the most likely culprit?

**Follow-up good answer:**
A network ACL missing the outbound rule for the ephemeral port range the client is using for its return traffic. Because NACLs are stateless, allowing inbound traffic on, say, port 443 doesn't implicitly allow the outbound response — you need a separate outbound rule permitting traffic back to the client's IP range on the ephemeral ports (typically 1024–65535) it initiated the connection from. Since the security group is stateful and already open, the NACL is the layer most likely to be silently dropping the un-declared return leg.

**Glossary:**
- **Ephemeral port** — a temporary port a client OS assigns for the source side of an outbound connection, used to route the response back.
- **Stateless firewall** — a filter that evaluates each packet/direction independently, with no memory of prior traffic.

**Mental model:**
This is one of the most common real AWS networking debugging scenarios — testing whether the candidate actually internalized stateful-vs-stateless rather than just being able to recite the definition.

**TL;DR:**
Security groups are stateful and instance-level (allow-only); NACLs are stateless and subnet-level (allow+deny, ordered, and you must explicitly permit return traffic).

**References:**
- [Control traffic to your AWS resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Control subnet traffic with network access control lists](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)

---

### Q3. Walk through exactly how AWS decides to allow or deny a request when multiple IAM policies apply. {#q3}

**Question:**
Walk through exactly how AWS decides to allow or deny a request when multiple IAM policies apply.

**Good answer:**
The default is implicit deny. AWS evaluates all applicable policies (identity-based, resource-based, permissions boundaries, SCPs, session policies) and combines them with specific rules: identity-based and resource-based policies on the *same account* are combined with a logical **union** (either one allowing is enough); identity-based policies combined with a permissions boundary or with SCPs are combined with a logical **intersection** (both must allow). Across all of this, **an explicit `Deny` anywhere always wins**, overriding any `Allow` no matter where it comes from. If nothing explicitly allows the action after all applicable policies are combined, the request is denied by default (implicit deny) — there's no "default allow."

**Follow-up question:**
A developer has `AdministratorAccess` attached directly to their IAM user, but a specific S3 `PutObject` call still fails. What are the possible causes given the evaluation logic above?

**Follow-up good answer:**
Since identity-based policy allows everything, the denial must be coming from a layer that intersects with or overrides it: (1) an SCP at the OU/account level in AWS Organizations that doesn't allow `s3:PutObject` (SCPs intersect with identity-based policies and affect even admins in member accounts), (2) an explicit `Deny` in a resource-based policy (e.g., the S3 bucket policy) targeting that principal, (3) a permissions boundary attached to the user that doesn't include `s3:PutObject`, or (4) a VPC endpoint policy if the request is going through one, since it also gates access independently. `AdministratorAccess` alone is never sufficient to guarantee access — it's just one input among several that must all agree.

**Glossary:**
- **Permissions boundary** — an advanced IAM feature that sets the maximum permissions an identity-based policy can grant to a user or role.
- **Implicit deny** — the default outcome when no policy explicitly allows an action.

**Mental model:**
Tests whether the candidate has memorized the union/intersection/explicit-deny rules precisely enough to debug a real multi-policy access problem, not just recite "least privilege" as a buzzword.

**TL;DR:**
Combine same-account identity+resource policies as a union, intersect identity policies with boundaries/SCPs, and an explicit deny anywhere always overrides any allow; absent any allow, the default is deny.

**References:**
- [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

---

### Q4. What are the different types of IAM policies (identity-based, resource-based, SCPs, permissions boundaries), and when would you reach for each? {#q4}

**Question:**
What are the different types of IAM policies (identity-based, resource-based, SCPs, permissions boundaries), and when would you reach for each?

**Good answer:**
Identity-based policies attach to a user, group, or role and grant that identity permissions — the default, everyday tool. Resource-based policies attach directly to a resource (like an S3 bucket policy or an SNS topic policy) and grant access to specified principals, which is the only way to grant *cross-account* access without the other account assuming a role. Service Control Policies (SCPs) apply at the AWS Organizations level to member accounts and set a permissions ceiling — they never grant anything themselves, only restrict what identity-based/resource-based policies in that account can allow. Permissions boundaries attach to a specific user or role and similarly cap what that principal's identity-based policies can grant, useful for letting a team self-manage IAM roles within a guardrail without risking privilege escalation.

**Follow-up question:**
You want to let a contractor account access one specific S3 bucket in your account without creating an IAM user for them in your account. Which policy type do you use, and why not the others?

**Follow-up good answer:**
A resource-based policy (an S3 bucket policy) naming the contractor's AWS account (or a specific role ARN in it) as the principal. Identity-based policies can't grant cross-account access on their own — you'd have to create an IAM user or role for them in your account, which is exactly what you're trying to avoid. SCPs and permissions boundaries only restrict permissions within your own organization/account; they can't grant access to an external account. The bucket policy directly authorizes the external principal without requiring any identity to exist on your side.

**Glossary:**
- **Principal** — the entity (user, role, account, or service) making a request; the "who" a policy grants or denies access to.
- **Cross-account access** — access to a resource in one AWS account by a principal from a different AWS account.

**Mental model:**
Checks whether the candidate can map each policy type to the *specific* problem it uniquely solves, rather than treating them as interchangeable ways to write "Allow" statements.

**TL;DR:**
Identity-based policies are the default per-principal grant; resource-based policies are the only way to grant cross-account access directly; SCPs and permissions boundaries only restrict, at the org and per-principal level respectively.

**References:**
- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [Service control policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)

---

### Q5. How do you choose between S3 storage classes for a given workload? {#q5}

**Question:**
How do you choose between S3 storage classes for a given workload?

**Good answer:**
Match access pattern to cost profile: **S3 Standard** for frequently accessed, latency-sensitive data (no retrieval fee, millisecond access). **S3 Standard-IA** or **One Zone-IA** for data accessed roughly monthly that still needs millisecond retrieval, accepting a per-GB retrieval fee and a 30-day minimum storage duration — One Zone-IA trades resilience (single AZ) for a lower price, appropriate for reproducible data like CRR replicas. **S3 Intelligent-Tiering** for data with unpredictable or changing access patterns, since it auto-moves objects between tiers with no retrieval fee, at the cost of a small monitoring fee. For archival, **S3 Glacier Instant Retrieval** (quarterly access, millisecond reads), **Glacier Flexible Retrieval** (yearly access, minutes-to-hours restore), and **Glacier Deep Archive** (rarely accessed, hours restore, cheapest) form a ladder of decreasing cost and increasing retrieval latency.

**Code example:**
```bash
aws s3api put-object --bucket my-bucket --key backups/db.sql \
  --storage-class STANDARD_IA
```

**Follow-up question:**
You have millions of small (under 100 KB) log files you rarely read but must retain for compliance for 7 years. Why is Intelligent-Tiering likely a poor fit here, and what would you pick instead?

**Follow-up good answer:**
S3 Intelligent-Tiering charges a small monitoring/automation fee *per object*, and its archive tiers only activate for objects that stay untouched for 90+ days — for millions of tiny, rarely-touched objects, that per-object monitoring fee can dominate the cost, and objects under 128 KB aren't eligible for auto-tiering at all (they stay in the Frequent Access tier). A better fit is **S3 Glacier Deep Archive**, which is priced for exactly this profile (rarely accessed, long retention, tolerant of hours-long restore times) without per-object monitoring overhead — though for many small objects you'd also want to batch/zip them first to reduce the fixed per-object metadata overhead Glacier classes carry.

**Glossary:**
- **Retrieval fee** — a per-GB charge for reading data out of an infrequent-access or archive storage class.
- **Minimum storage duration** — the minimum time you're billed for regardless of when you delete or transition an object (e.g., 30 days for IA classes).

**Mental model:**
Tests real cost-optimization judgment — can the candidate reason about access-pattern-to-cost trade-offs instead of just naming the storage classes that exist.

**TL;DR:**
Pick by access frequency and predictability: Standard for hot data, IA classes for predictable-infrequent, Intelligent-Tiering for unpredictable, and the Glacier ladder for archival by expected retrieval urgency.

**References:**
- [Understanding and managing Amazon S3 storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

---

### Q6. How does S3 achieve its durability, and what's its consistency model? {#q6}

**Question:**
How does S3 achieve its durability, and what's its consistency model?

**Good answer:**
S3 Standard and most other classes redundantly store objects across a minimum of three geographically separated Availability Zones within a Region, designed for 99.999999999% ("eleven nines") durability and 99.99% availability over a given year — it's designed to sustain the loss of an entire AZ without data loss (S3 One Zone-IA is the exception, trading that AZ-loss resilience for lower cost). On consistency, S3 provides strong read-after-write consistency for all operations: as soon as a PUT succeeds, any subsequent GET or LIST reflects that write — there's no eventual-consistency window to design around, which historically was not the case and is a common outdated assumption candidates carry over from older material.

**Follow-up question:**
Given strong read-after-write consistency, is it still possible for two concurrent requests to see different results for the same key?

**Follow-up good answer:**
Yes — strong consistency guarantees that *after* a write completes, all subsequent reads see it; it says nothing about the ordering of two writes racing each other, or a read that's concurrent with an in-flight write (which may return either the old or new version, but never garbage/partial data). If two PUTs to the same key race, the last one to complete wins, and reads before that completion may still see the prior version. So consistency here means "no stale reads after a completed write," not "global agreement on the order of concurrent operations."

**Glossary:**
- **Read-after-write consistency** — a guarantee that a read immediately following a completed write will return that write's data, not a stale prior version.
- **Availability Zone (AZ)** — one or more discrete, physically separated data centers within an AWS Region.

**Mental model:**
Checks whether the candidate has current, accurate knowledge (S3's consistency model changed materially in Dec 2020) rather than repeating outdated "eventual consistency" folklore about S3.

**TL;DR:**
S3 is designed for eleven-nines durability via multi-AZ redundancy and provides strong read-after-write consistency for all operations, not eventual consistency.

**References:**
- [Data protection in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DataDurability.html)

---

### Q7. What is IMDSv2, and why does it matter for security compared to IMDSv1? {#q7}

**Question:**
What is IMDSv2, and why does it matter for security compared to IMDSv1?

**Good answer:**
The Instance Metadata Service (IMDS) lets code running on an EC2 instance retrieve instance metadata, including temporary IAM credentials for the instance's role, at `169.254.169.254`. IMDSv1 allows a simple `GET` request with no token — meaning a Server-Side Request Forgery (SSRF) vulnerability in an application (e.g., an app that fetches an attacker-supplied URL) can trivially be tricked into hitting the metadata endpoint and exfiltrating the instance's IAM credentials. IMDSv2 requires a session first: a `PUT` request to `/latest/api/token` returns a token that must then be included as a header on subsequent `GET` requests. This matters because most SSRF exploitation is via simple GET-based redirects/proxies — requiring an initial PUT with a custom header is a much harder primitive for an attacker to smuggle through a vulnerable app, closing off a whole common exploitation path.

**Code example:**
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Follow-up question:**
An instance is configured to allow both IMDSv1 and IMDSv2 ("token optional"). Is it actually protected against the SSRF-to-credential-theft attack?

**Follow-up good answer:**
No — if IMDSv1 is still allowed, an attacker's SSRF payload can simply skip the token dance and issue the same old unauthenticated GET request IMDSv1 always supported, bypassing IMDSv2 entirely. The protection only takes effect when the instance (or the account) is configured with `HttpTokens=required`, which rejects any request that doesn't present a valid IMDSv2 token — effectively disabling IMDSv1. "Optional" mode is a compatibility bridge for migration, not a security boundary; you have to explicitly enforce v2-only to get the SSRF mitigation.

**Glossary:**
- **SSRF (Server-Side Request Forgery)** — a vulnerability where an attacker tricks a server into making an HTTP request to an unintended destination, often internal infrastructure.
- **Hop limit** — the number of network hops IMDSv2's PUT response is allowed to traverse, relevant for containerized environments.

**Mental model:**
This tests awareness of a real, high-profile vulnerability class (this exact SSRF-to-IMDS-credential-theft chain was behind a well-known 2019 cloud breach) and whether the candidate knows the fix requires *enforcement*, not just availability.

**TL;DR:**
IMDSv2's PUT-then-token-GET flow closes the classic SSRF-to-credential-theft path that IMDSv1's plain GET allowed, but only when configured as token-required, not merely token-optional.

**References:**
- [Configure the Instance Metadata Service options](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html)

---

### Q8. How does a NAT gateway let private-subnet instances reach the internet without being reachable from it? {#q8}

**Question:**
How does a NAT gateway let private-subnet instances reach the internet without being reachable from it?

**Good answer:**
A public NAT gateway sits in a public subnet with an Elastic IP, and the private subnet's route table sends its `0.0.0.0/0` traffic to the NAT gateway instead of directly to an Internet Gateway. The NAT gateway performs source address translation: it rewrites the private instance's source IP to its own Elastic IP for outbound requests, and when the response comes back, it translates the destination back to the originating private IP and forwards it inward. Critically, connections can only be *initiated from within the VPC* — the NAT gateway has no mechanism for an external host to initiate a new inbound connection to a private instance, since there's no rule mapping an unsolicited external request to any specific private IP.

**Follow-up question:**
When would you choose a NAT instance over a NAT gateway despite AWS recommending the gateway?

**Follow-up good answer:**
Rarely, but two legitimate cases: needing port forwarding (NAT gateways don't support it) or wanting to use the box as a bastion host as well — both features NAT gateways explicitly don't support. You'd also consider it if you need to attach a security group directly to the NAT device itself (NAT gateways can't have security groups attached, only the resources behind them can), for fine-grained outbound filtering at the NAT layer. Otherwise, NAT gateway wins on almost every axis: it's managed, highly available per-AZ, scales to 100 Gbps automatically, and requires no patching, versus a NAT instance requiring you to manage failover, patch the OS, and size the instance for your bandwidth.

**Glossary:**
- **Elastic IP** — a static, public IPv4 address you can allocate and associate with AWS resources.
- **Source NAT (SNAT)** — rewriting the source IP address of outbound packets so responses route back correctly.

**Mental model:**
Tests understanding of asymmetric connectivity (outbound-only) as a security property, not just "NAT gateway = internet access," plus whether the candidate can articulate the narrow real cases for the "wrong" answer (NAT instance).

**TL;DR:**
A NAT gateway translates private instances' outbound source IPs to its own Elastic IP and only forwards return traffic for connections the VPC itself initiated, so external hosts have no path to initiate inbound connections.

**References:**
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Compare NAT gateways and NAT instances](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)

---

### Q9. An EC2 instance in a private subnet can't reach an external API. Walk through your diagnostic methodology. {#q9}

**Question:**
An EC2 instance in a private subnet can't reach an external API. Walk through your diagnostic methodology.

**Good answer:**
Work outward from the instance, layer by layer: (1) check the instance's **security group** outbound rules allow the destination port/protocol; (2) check the subnet's **NACL** for both outbound (to the destination) and inbound (for the ephemeral-port return traffic) rules, remembering NACLs are stateless; (3) check the **route table** for the subnet — is there a route to a NAT gateway (or IGW, if it's meant to be public) for the destination CIDR, and is that NAT gateway actually healthy and in a public subnet with its own valid route to an IGW; (4) check DNS resolution is working if the failure looks like a name-resolution issue rather than a connect timeout; (5) if all of that looks right, use **VPC Flow Logs** on the ENI to see whether packets are being ACCEPTed or REJECTed and at which point, which tells you definitively whether it's a security-group/NACL block versus a routing black hole (flow logs show REJECT for SG/NACL denies, or simply no return traffic for a routing issue).

**Follow-up question:**
VPC Flow Logs show the outbound packet as ACCEPTed, but the application still times out. What do you check next?

**Follow-up good answer:**
If flow logs confirm the packet left the ENI cleanly, the block is likely *outside* the VPC's control plane — check whether the NAT gateway's route to an Internet Gateway is intact and the IGW is attached to the VPC, whether the destination itself is reachable (an external outage or an IP allow-list on their end), or whether a stateful inspection appliance/firewall in a more complex topology (e.g., centralized egress via Transit Gateway) is silently dropping the traffic downstream of the ENI. It's also worth checking whether the timeout is actually a DNS resolution failure being misreported as a connection timeout — flow logs on the ENI won't capture a DNS resolver failure the same way.

**Glossary:**
- **VPC Flow Logs** — a feature that captures IP traffic metadata (accept/reject, source/destination, ports) going to and from network interfaces in a VPC.

**Mental model:**
This is the single most common "debug my AWS network" interview scenario — testing whether the candidate has an actual systematic layer-by-layer method versus guessing at random components.

**TL;DR:**
Diagnose VPC connectivity outward from the instance: security group, then NACL (both directions, stateless), then route table/NAT health, then DNS, using VPC Flow Logs to definitively distinguish a security block from a routing gap.

**References:**
- [Logging IP traffic using VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

---

### Q10. How would you find out exactly why a specific IAM principal got an access-denied error? {#q10}

**Question:**
How would you find out exactly why a specific IAM principal got an access-denied error?

**Good answer:**
First, read the error message itself carefully — modern AWS access-denied errors often name the *policy type* responsible (e.g., "with an explicit deny in a service control policy") and sometimes the exact policy ARN, which can resolve the question immediately. If it's ambiguous, check **AWS CloudTrail** event history for the exact API call: CloudTrail records the request, the identity that made it, and — critically — whether it was denied, which lets you correlate the timestamp/action/resource precisely rather than guessing. For request-signing or condition-key issues, the **IAM Policy Simulator** lets you test a specific principal against a specific action/resource without making a real API call. Work through the evaluation order: identity-based policy, resource-based policy (if applicable), SCPs, permissions boundary, session policies — checking each for either a missing `Allow` (implicit deny) or a present `Deny` (explicit deny).

**Follow-up question:**
CloudTrail shows the call was denied by an SCP, but you're the account admin with `AdministratorAccess` and can't find any SCP restricting that action in the console. What's the likely explanation?

**Follow-up good answer:**
SCPs are managed at the AWS Organizations level, not within the individual member account — an account admin with full IAM permissions in their own account has no visibility into or control over SCPs unless they also have Organizations permissions and are looking in the *management account* or delegated administrator's console. The SCP could be attached at a parent Organizational Unit level rather than directly to the account, which is easy to miss if you're only checking the account's direct attachments. You'd need to ask the Organizations administrator, or if you have the `organizations:Describe*`/`organizations:List*` permissions, trace the OU hierarchy yourself to find where the restricting SCP is actually attached.

**Glossary:**
- **CloudTrail** — an AWS service that logs API calls made within an account, including who made them and whether they were allowed or denied.
- **IAM Policy Simulator** — a tool for testing the effect of IAM policies on specific actions/resources without making live API calls.

**Mental model:**
Tests real operational debugging skill for one of the most common and frustrating AWS support scenarios, and whether the candidate understands the organizational boundary between account-level and Organizations-level policy control.

**TL;DR:**
Read the error message for the policy type/ARN first, then correlate with CloudTrail for the exact denied call, and use the Policy Simulator to test hypotheses — remembering SCPs live at the Organizations level, invisible from inside a member account's own IAM console.

**References:**
- [Troubleshoot access denied error messages](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_access-denied.html)

---

### Q11. S3 used to recommend randomizing key prefixes for high-throughput workloads. Is that still necessary? {#q11}

**Question:**
S3 used to recommend randomizing key prefixes for high-throughput workloads. Is that still necessary?

**Good answer:**
No, not since a 2018 architecture change. S3 now automatically scales request rate per prefix, supporting at least 3,500 PUT/COPY/POST/DELETE or 5,500 GET/HEAD requests per second *per prefix*, with no limit on the number of prefixes in a bucket — so you can scale linearly just by using more prefixes (e.g., 10 prefixes could sustain roughly 55,000 GET/s), regardless of whether those prefixes look "random." The older guidance to reverse or randomize key names (e.g., hash-prefixing) was a workaround for a partitioning scheme that no longer exists. Scaling to a new higher request rate happens gradually rather than instantaneously, though, so a sudden spike can still produce transient 503 "Slow Down" errors while S3 catches up.

**Follow-up question:**
Your application still gets occasional 503 SlowDown errors from S3 even after distributing keys across several prefixes. What would you check or change?

**Follow-up good answer:**
First confirm the spike is sudden rather than sustained — since S3's scaling per prefix is gradual, not instantaneous, a request rate that jumps sharply can outrun that scaling even with good prefix distribution, and the fix there is often just retrying with exponential backoff (which the AWS SDKs do by default) rather than architectural changes. If the sustained rate is genuinely exceeding the per-prefix limits, check whether requests are actually distributed across distinct prefixes as intended (a bug funneling everything through one "hot" prefix is a common cause) or consider using S3 Transfer Acceleration or CloudFront/ElastiCache in front of S3 if the workload is latency- or throughput-bound rather than request-count-bound.

**Glossary:**
- **Prefix (S3)** — the portion of an object key up to the last `/`, historically relevant to S3's internal partitioning scheme.
- **503 Slow Down** — an S3 error returned transiently while it's in the process of automatically scaling partition capacity to a new, higher request rate.

**Mental model:**
Tests whether the candidate has current knowledge rather than repeating stale, pre-2018 S3 performance folklore that's still widely circulated in interview prep material.

**TL;DR:**
S3 auto-scales request rate per prefix (3,500 write / 5,500 read req/s each, unlimited prefixes) since 2018 — manual key randomization for performance is obsolete guidance.

**References:**
- [Best practices design patterns: optimizing Amazon S3 performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)

---

### Q12. What does "least privilege" mean in concrete IAM terms, and how do you actually apply it rather than just stating it as a principle? {#q12}

**Question:**
What does "least privilege" mean in concrete IAM terms, and how do you actually apply it rather than just stating it as a principle?

**Good answer:**
Least privilege means granting only the specific actions on the specific resources a principal actually needs, for only as long as it needs them — not "Action: *" on "Resource: *" because it's convenient. Concretely: scope IAM policy statements to specific actions (`s3:GetObject` not `s3:*`) and specific resource ARNs (a specific bucket/prefix, not `*`); use IAM Access Analyzer and the service-last-accessed data (visible per-user/role) to find and strip unused permissions after the fact, since starting minimal and adding on request is more reliable than guessing upfront; prefer temporary credentials via role assumption over long-lived access keys, since a scoped, expiring credential limits blast radius even if compromised; and use condition keys (like source IP or MFA-present) to further narrow when a grant applies.

**Follow-up question:**
A team insists they need `s3:*` on `*` because they "don't know exactly which S3 actions their app will need." How would you push back constructively?

**Follow-up good answer:**
Point them at **IAM Access Analyzer's policy generation** feature, or the **service-last-accessed data**: deploy with a broad policy in a non-production environment first, let CloudTrail/Access Analyzer observe the actual API calls the application makes over a representative period, then generate a scoped policy from that observed usage rather than either guessing narrowly (risking outages) or granting everything (risking breach blast radius). This turns "we don't know what we need" from a justification for over-permissioning into a data-gathering step with a concrete, low-cost path to a correctly-scoped policy.

**Glossary:**
- **IAM Access Analyzer** — a service that can generate least-privilege policies based on observed CloudTrail activity and identify unused permissions.
- **Blast radius** — the scope of damage possible if a given credential or component is compromised.

**Mental model:**
Distinguishes candidates who can only recite "least privilege" as a slogan from those who have an actual operational process for achieving it under real uncertainty about requirements.

**TL;DR:**
Least privilege means scoping actions/resources/conditions to exactly what's needed, achieved practically via Access Analyzer/service-last-accessed data and temporary credentials rather than upfront guessing.

**References:**
- [Grant least privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)

---

### Q13. What is "defense in depth" and how do security groups, NACLs, and IAM concretely form layers of it in an AWS architecture? {#q13}

**Question:**
What is "defense in depth" and how do security groups, NACLs, and IAM concretely form layers of it in an AWS architecture?

**Good answer:**
Defense in depth means no single control is trusted as the sole barrier — if one layer is misconfigured or bypassed, others still constrain the blast radius. Concretely in AWS: NACLs provide a coarse, subnet-wide backstop (e.g., blocking a known-bad CIDR range entirely, independent of any instance-level misconfiguration); security groups provide fine-grained, per-instance/per-service filtering (the primary day-to-day control); and IAM constrains what an authenticated principal — including one that *did* get network access — can actually do to AWS APIs and data. A network-layer compromise (e.g., an attacker reaching an instance) doesn't automatically grant IAM permissions, and an over-permissioned IAM role doesn't help an attacker who can't reach the network path to use it. Each layer independently narrows the attack surface the others leave open.

**Follow-up question:**
If security groups already default-deny and are stateful and sufficient for most traffic control, why bother adding NACLs at all?

**Follow-up good answer:**
Because NACLs protect against misconfiguration *of the security groups themselves* — if someone accidentally opens a security group too broadly, a correctly configured NACL at the subnet level can still block the truly dangerous traffic (e.g., a blanket deny on a known-malicious IP range, or blocking all traffic to a subnet that should never be internet-reachable at all, regardless of what any individual instance's security group says). It's specifically valuable as an independent, harder-to-accidentally-loosen backstop, precisely because it's rarely touched day-to-day, whereas security groups are frequently modified per-application and thus more prone to human error.

**Glossary:**
- **Attack surface** — the total set of points where an unauthorized user could try to enter or extract data.

**Mental model:**
Tests whether the candidate understands defense in depth as independent, non-redundant layers rather than believing one sufficiently strict control makes the others unnecessary.

**TL;DR:**
Defense in depth means NACLs, security groups, and IAM each independently narrow the attack surface, so a failure or misconfiguration in one layer doesn't automatically compromise the whole system.

**References:**
- [Security best practices for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)

---

### Q14. You need to connect 20 VPCs so they can all communicate. Would you use VPC peering or Transit Gateway, and why? {#q14}

**Question:**
You need to connect 20 VPCs so they can all communicate. Would you use VPC peering or Transit Gateway, and why?

**Good answer:**
Transit Gateway. VPC peering connections are point-to-point and explicitly **not transitive** — if A is peered with B and B is peered with C, A cannot reach C through B; you'd need a direct A–C peering too, meaning full mesh connectivity for N VPCs requires on the order of N(N-1)/2 peering connections, which becomes unmanageable well before 20 VPCs. Transit Gateway acts as a central hub: each VPC attaches to it once, and the Transit Gateway handles routing between all attached VPCs (and on-premises networks via VPN/Direct Connect) transitively, so connectivity scales roughly linearly (N attachments) instead of quadratically. The trade-off is Transit Gateway has per-attachment and per-GB data processing charges that peering doesn't, and peering has marginally lower latency since traffic goes directly point-to-point rather than through a hub.

**Follow-up question:**
Two VPCs attached to the same Transit Gateway aren't able to reach each other even though both attachments show as "available." What would you check?

**Follow-up good answer:**
The Transit Gateway route table associated with each attachment — Transit Gateway doesn't automatically route between all attached VPCs by default in every configuration; each attachment is associated with a TGW route table, and that route table needs propagated or static routes covering the other VPC's CIDR. It's a common gotcha: people assume "attached to the same TGW" implies full connectivity, but if the attachments are associated with different TGW route tables (a common pattern for network segmentation), traffic between them won't route unless you deliberately configure routes/associations to allow it. Also verify the VPCs' own route tables have routes pointing at the Transit Gateway for the relevant CIDRs.

**Glossary:**
- **Transitive routing** — the ability for traffic to flow through an intermediate hop (e.g., A→hub→C) without a direct connection between the endpoints.
- **TGW route table** — a routing table specific to a Transit Gateway attachment, separate from a VPC's own route tables.

**Mental model:**
Tests whether the candidate understands the specific scaling failure mode of peering (non-transitivity, quadratic growth) that makes it the wrong tool past a small number of VPCs, not just "Transit Gateway is newer so it's better."

**TL;DR:**
VPC peering is non-transitive and scales quadratically with VPC count; Transit Gateway provides transitive hub-and-spoke routing that scales linearly, making it the right choice for connecting many VPCs.

**References:**
- [What is a transit gateway?](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [VPC peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)

---

### Q15. What's wrong with an IAM policy that grants `"Action": "s3:*"` on `"Resource": "*"`, and how did policies like this cause real S3 data breaches? {#q15}

**Question:**
What's wrong with an IAM policy that grants `"Action": "s3:*"` on `"Resource": "*"`, and how did policies like this cause real S3 data breaches?

**Good answer:**
This grants every S3 action (including `PutBucketPolicy`, `PutBucketAcl`, `DeleteBucket`) on every bucket in the account to whatever principal holds it — far beyond what almost any single workload legitimately needs, and it violates least privilege by construction. In practice, several well-publicized breaches trace back to exactly this pattern combined with a bucket policy or ACL that was *also* misconfigured to allow public access: an overly broad IAM policy meant an application or compromised credential could modify bucket-level permissions (not just read/write objects), and a mistake or compromise then flipped a bucket's access controls to public, exposing its contents to the internet. The fix is scoping the policy to only the specific actions needed (typically just `s3:GetObject`/`s3:PutObject` on a specific bucket ARN) so that even a fully compromised credential can't touch bucket-level access configuration at all.

**Follow-up question:**
Beyond IAM policy scoping, what account-level control specifically prevents S3 buckets from being made public even if someone tries?

**Follow-up good answer:**
**S3 Block Public Access**, which can be set at the bucket or account level and overrides any bucket policy or ACL that would otherwise grant public access — even a bucket policy explicitly allowing `Principal: "*"` is blocked from taking effect if Block Public Access is enabled. AWS enables this by default for new buckets and accounts specifically because misconfigured bucket policies/ACLs were such a common and high-impact source of breaches; disabling it now requires a deliberate, auditable action, which raises the bar against an accidental public-exposure misconfiguration.

**Glossary:**
- **Bucket policy** — a resource-based JSON policy attached directly to an S3 bucket controlling who can access it.
- **S3 Block Public Access** — an account/bucket-level setting that overrides any policy or ACL attempting to grant public access.

**Mental model:**
Connects an abstract IAM anti-pattern to concrete, real-world incident causation, testing whether the candidate can reason about *why* over-permissioning is dangerous in practice, not just that it's "bad practice."

**TL;DR:**
Wildcard `s3:*`/`*` policies let a compromised credential alter bucket-level access controls, not just object data — real breaches followed this pattern; S3 Block Public Access is the specific guardrail against the resulting exposure.

**References:**
- [Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)

---

### Q16. How do you grant a Lambda function in Account A temporary access to resources in Account B without sharing long-lived credentials? {#q16}

**Question:**
How do you grant a Lambda function in Account A temporary access to resources in Account B without sharing long-lived credentials?

**Good answer:**
Create an IAM role in Account B whose trust policy names Account A's Lambda execution role as a trusted principal, then grant that Account B role the permissions it needs. Account A's Lambda function calls STS `AssumeRole` against that role's ARN, receiving short-lived temporary credentials (an access key, secret key, and session token, defaulting to a 1-hour session by default and configurable up to the role's max session duration setting). The function uses those temporary credentials for subsequent calls into Account B. No static access keys are ever created or shared between accounts — the trust relationship is entirely policy-based, and the credentials automatically expire, limiting exposure if they somehow leaked.

**Code example:**
```json
// Trust policy on the role in Account B
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::AAAA_ACCOUNT_A:role/my-lambda-role"},
    "Action": "sts:AssumeRole"
  }]
}
```

**Follow-up question:**
The Lambda function needs to assume a role in Account B, which itself needs to assume a further role in Account C to complete the operation. Is there anything different about the session limits in this chained scenario?

**Follow-up good answer:**
Yes — this is "role chaining," and using the temporary credentials from one assumed role to assume a *further* role caps that second session's duration at a maximum of 1 hour, regardless of what the target role's configured maximum session duration setting normally allows. This limitation specifically applies to chaining (assuming a role using already-temporary credentials from a previous assumption); it doesn't apply to the *initial* assumption from a Lambda execution role's own credentials or an EC2 instance profile. If a workflow needs longer-lived access than that across a multi-hop chain, the architecture typically needs rethinking (e.g., avoiding the chain, or re-assuming periodically) rather than trying to extend the 1-hour chained-session cap.

**Glossary:**
- **Trust policy** — the resource-based policy attached to an IAM role specifying which principals are allowed to assume it.
- **Role chaining** — using credentials obtained from assuming one role to then assume a second, different role.

**Mental model:**
Tests understanding of the standard, correct pattern for cross-account access (as opposed to the anti-pattern of sharing static keys) and awareness of a specific, easy-to-miss limitation (the 1-hour role-chaining cap).

**TL;DR:**
Cross-account access should go through STS AssumeRole against a trust-policy-scoped role, yielding short-lived credentials — chained role assumptions are additionally capped at a 1-hour session regardless of the target role's normal max.

**References:**
- [Methods to assume a role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html)

---

### Q17. What's the difference between a gateway VPC endpoint and an interface VPC endpoint? {#q17}

**Question:**
What's the difference between a gateway VPC endpoint and an interface VPC endpoint?

**Good answer:**
A **gateway endpoint** is a route-table target — you add a route in your subnet's route table pointing traffic destined for the service to the endpoint, and it works only for **S3 and DynamoDB**; it doesn't use AWS PrivateLink or consume an IP address in your subnet. An **interface endpoint** provisions an actual Elastic Network Interface (ENI) with a private IP in your subnet, and works via AWS PrivateLink for a much broader range of AWS services (and some third-party/partner services) — traffic reaches it via DNS resolution to that private IP rather than a route table entry. Both let traffic to the AWS service stay entirely on the AWS network instead of egressing through a NAT gateway or Internet Gateway, but interface endpoints incur an hourly charge plus data processing fees, while gateway endpoints are free.

**Follow-up question:**
After adding an S3 gateway endpoint to your VPC, some instances still show S3 traffic going out through the NAT gateway in VPC Flow Logs. What would you check?

**Follow-up good answer:**
Whether the gateway endpoint's route was actually added to the *specific route table* associated with those instances' subnets — gateway endpoints require an explicit route table association, and if a subnet uses a different route table than the one you edited (common in multi-tier VPC designs with per-subnet-group route tables), its traffic will keep taking the NAT gateway's default route instead. You'd also check the endpoint policy isn't restricting the specific bucket/actions those instances need, which would cause a different failure (explicit denial) but is worth ruling out alongside the routing check.

**Glossary:**
- **AWS PrivateLink** — the underlying technology that lets interface endpoints expose services privately within a VPC via ENIs.
- **Endpoint policy** — an IAM resource policy attached to a VPC endpoint controlling which principals/actions can use it.

**Mental model:**
Tests precise knowledge of a frequently-confused pair of AWS constructs, and whether the candidate can debug the specific, common gateway-endpoint gotcha (per-route-table association) rather than just defining the terms.

**TL;DR:**
Gateway endpoints are free route-table targets limited to S3/DynamoDB; interface endpoints are ENI-based PrivateLink connections (with hourly/data charges) supporting a much broader set of services.

**References:**
- [AWS PrivateLink concepts](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [Gateway endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)

---

### Q18. What exactly can and can't a Service Control Policy (SCP) do? {#q18}

**Question:**
What exactly can and can't a Service Control Policy (SCP) do?

**Good answer:**
An SCP can only **restrict** — it sets the maximum available permissions for IAM users and roles in member accounts of an AWS Organization, and never grants anything by itself. Even with an SCP that "allows" an action, a user still needs an actual identity-based (or resource-based) policy granting that permission; the SCP is a ceiling, not a source of permissions. SCPs apply to every user and role in an affected member account, including that account's root user, and the effective permission for any action is the intersection of what the SCP(s), any RCPs, and the identity/resource-based policies all separately allow. Critically, SCPs **do not affect the Organization's management account** at all, and they don't restrict AWS service-linked roles.

**Follow-up question:**
You attach an SCP that denies `s3:DeleteBucket` to an OU, but users in an account under that OU still successfully delete buckets. What are the possible explanations?

**Follow-up good answer:**
Most likely, the account in question is the **management account** of the organization, which SCPs never affect regardless of where they're attached in the OU hierarchy. Other possibilities: the account isn't actually under that OU (check the OU hierarchy/inheritance — SCPs apply based on actual attachment location, not assumption), the SCP wasn't actually saved/attached correctly, or the deletion is happening via a service-linked role, which SCPs explicitly cannot restrict. It's also worth confirming SCPs are enabled as a policy type for that Organizations root at all — they must be explicitly enabled, they're not on by default in every organization.

**Glossary:**
- **Organizational Unit (OU)** — a grouping of accounts within an AWS Organization used to apply policies collectively.
- **Management account** — the account that created an AWS Organization; SCPs never restrict this account.

**Mental model:**
Tests the specific, commonly-missed exceptions (management account exemption, service-linked roles) that trip people up in real SCP troubleshooting, beyond the basic "SCPs restrict permissions" definition.

**TL;DR:**
SCPs only restrict (never grant), apply to all member-account principals including root, but never apply to the management account or service-linked roles.

**References:**
- [Service control policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)

---

### Q19. What's the practical difference between attaching a role's permissions via a permissions boundary versus just writing a tighter identity-based policy? {#q19}

**Question:**
What's the practical difference between attaching a role's permissions via a permissions boundary versus just writing a tighter identity-based policy?

**Good answer:**
A permissions boundary caps what an identity-based policy attached to that specific principal can grant — it's most useful when you don't control what identity-based policy will eventually be attached (or want to allow *someone else*, like a developer, to attach their own policies within a safe ceiling) rather than when you already know the exact tight policy to write yourself. If you fully control and know the desired permissions upfront, just writing a precisely-scoped identity-based policy is simpler and sufficient — no boundary needed. Boundaries earn their complexity specifically in delegation scenarios: e.g., letting an application team create and attach IAM policies to roles they manage, while guaranteeing via the boundary that they can never grant themselves permissions beyond a fixed ceiling, even by mistake or by attaching an overly broad policy.

**Follow-up question:**
A developer with permission to create IAM roles attaches `AdministratorAccess` to a new role they create, but the role still can't do most administrative actions. Why?

**Follow-up good answer:**
The developer's own IAM permissions to create roles were likely themselves scoped with a requirement that any role they create must have a specific permissions boundary attached (a common delegation pattern, often enforced via a condition on `iam:CreateRole` requiring `iam:PermissionsBoundary` to be set to a specific policy ARN). Even though the attached identity-based policy is `AdministratorAccess`, the boundary intersects with it and caps the effective permissions to whatever the boundary allows — demonstrating exactly the delegation safety boundaries are designed for: the developer can attach any policy they want, but can never exceed the ceiling an administrator predefined.

**Glossary:**
- **Delegation** — allowing another principal to manage IAM resources (like creating roles) within constraints you control.

**Mental model:**
Tests whether the candidate understands permissions boundaries as a *delegation* tool specifically, not just a redundant way to restrict permissions when writing your own tight policy would be simpler and sufficient.

**TL;DR:**
Permissions boundaries matter specifically for delegation — capping what someone else's self-attached policies can grant — not as a substitute for simply writing a tight policy yourself when you already know the exact permissions needed.

**References:**
- [Permissions boundaries for IAM entities](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)

---

### Q20. Your company is migrating from a flat, single-account AWS setup to AWS Organizations with multiple accounts. What's the security argument for doing this, given that you could apply the same IAM policies within one account? {#q20}

**Question:**
Your company is migrating from a flat, single-account AWS setup to AWS Organizations with multiple accounts. What's the security argument for doing this, given that you could apply the same IAM policies within one account?

**Good answer:**
Account boundaries are a fundamentally stronger isolation unit than IAM policy boundaries within a single account: resource-level isolation (a compromised credential in one account has zero standing access to resources in a sibling account, versus needing to be perfectly correct across every IAM policy and resource policy in a shared account), blast-radius containment for both security incidents and operational mistakes (a misconfigured service quota, a runaway Lambda, or a compromised credential in a "dev" account can't directly touch "prod" resources), and cleaner billing/compliance boundaries. It also unlocks SCPs as an *organization-wide, un-bypassable-from-within-the-account* guardrail — something no identity-based policy can achieve, since even an account's own administrator can't remove an SCP applied from above by the Organizations management account. This is why multi-account is now AWS's own recommended default architecture (via AWS Control Tower/Landing Zone) rather than a large single account carefully partitioned with IAM alone.

**Follow-up question:**
Isn't this over-engineering for a small startup with a five-person engineering team? What's the actual cost of adopting multi-account this early versus waiting?

**Follow-up good answer:**
The main cost is operational overhead: managing cross-account IAM roles, centralized logging/CloudTrail aggregation, and Organizations/SCP setup takes real upfront effort disproportionate to a five-person team's immediate needs, and tools like AWS Control Tower reduce but don't eliminate this. The counter-argument is that retrofitting account separation onto a mature, entangled single-account setup later — untangling resource dependencies, migrating data, re-wiring CI/CD — is significantly more expensive than starting with at least a minimal split (e.g., separate accounts for prod vs. non-prod) from day one, even if a fuller per-team/per-environment account structure is deferred until the org actually grows into needing it. The right call is usually "start with 2-3 accounts, not 1, and not 30."

**Glossary:**
- **AWS Control Tower** — an AWS service that automates setting up a secure, multi-account AWS environment based on best-practice blueprints.
- **Blast radius** — the scope of impact a security incident or operational failure can have, ideally contained by isolation boundaries.

**Mental model:**
This is a judgment/trade-off question testing whether the candidate can argue both for and against an architectural decision pragmatically, rather than dogmatically insisting multi-account is always correct regardless of context.

**TL;DR:**
Multi-account isolation via AWS Organizations provides a stronger, un-bypassable-from-within blast-radius boundary than IAM alone can within one account, at the cost of real operational overhead best justified even minimally (e.g., prod/non-prod split) from early on rather than retrofitted later.

**References:**
- [AWS Organizations terminology and concepts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=cloud&tags=aws-core-services-networking-and-iam&autostart=1" | relative_url }})
