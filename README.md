# Migrating a Production Secrets Platform to OpenBao

## Document status

* **Audience:** platform engineering, SRE, security, and application teams
* **Scope:** implementation experience from a multi-cluster Kubernetes platform
* **Purpose:** explain why OpenBao was selected, how it was deployed, what was tested, what was learned, and what is required for production operation
* **Sensitivity:** organization names, internal URLs, account identifiers, repository names, and personal names have been removed

## Executive summary

The organization selected OpenBao after architecture discussions and proof-of-concept work. The first delivery phase was not a simple binary replacement: it replaced encrypted-file secret handling used by deployment automation with a Kubernetes-native secret-management platform. The design also preserved a Vault-compatible operating model so that existing concepts—authentication methods, policies, secret engines, leases, TTLs, and audit logging—could be retained.

The implementation uses one OpenBao deployment per Kubernetes cluster rather than one shared global cluster. Applications declare the properties of their secrets through a custom Secret Definition resource. OpenBao generates or retrieves the value and renders a normal Kubernetes Secret in the application namespace. Existing applications therefore continue consuming Kubernetes Secrets without application-code changes, while engineers no longer need to know the secret values.

The project included:

* a security and compliance review;
* a real-application pilot;
* OpenBao deployment and hardening work;
* testing of static and generated secrets;
* detailed testing of token TTL and dynamic-secret leases;
* dynamic security testing of OpenBao 2.5.1; and
* production-readiness work for audit logging, backup, monitoring, and upgrades.

## 1. Why the change was needed

### 1.1 Encrypted files were not enough

At the time of the assessment, more than 230 secrets were used by Kubernetes-deployed services to connect to other services, cloud providers, and data stores. Many were stored in encrypted files committed to source control.

Encryption protected the files at rest, but the model had important limitations:

* access to the decryption key could expose many secrets at once;
* production and non-production access were not sufficiently separated;
* the file-based workflow did not provide enterprise-level authorization;
* access auditing was limited or absent;
* some secrets existed outside the encrypted-file workflow; and
* secret generation, rotation, and revocation were not consistently managed.

The key problem was therefore not only where the ciphertext was stored. It was how access to the plaintext was granted, audited, rotated, and revoked.

### 1.2 Why OpenBao was selected

After discussions and proof-of-concept work, OpenBao was selected because it provided a community-driven, open-source implementation based on the Vault model while supporting the capabilities required by the platform:

* secret engines;
* authentication methods;
* fine-grained policies;
* renewable tokens and leases;
* dynamic secrets;
* Kubernetes integration;
* audit devices; and
* high availability with integrated storage.

The choice was also practical. The platform could retain familiar Vault concepts and client patterns while moving to an independently governed open-source project.

### 1.3 Scope was intentionally phased

The initial phase focused on Kubernetes workloads and deployment automation. Future phases were expected to extend the model to additional secret producers and consumers, including CI workflows, other automation systems, external integrations, and controlled human access.

This phased approach avoided attempting to migrate every secret and every access pattern at the same time.

## 2. Decisions made by the implementation team

The following decisions were specific to the implementation rather than generic migration advice.

| Area | Decision | Reason |
|---|---|---|
| Deployment topology | Deploy a separate OpenBao cluster in each Kubernetes cluster | Reduce blast radius and simplify isolation |
| Secret interface | Use a custom Secret Definition resource | Store secret requirements in repositories, not secret values |
| Application compatibility | Render standard Kubernetes Secrets | Keep existing applications unchanged |
| High availability | Use integrated storage with Raft and active/standby nodes | Provide quorum-based durability and failover |
| Unsealing | Use cloud KMS auto-unseal | Avoid manually reconstructingunseal material during normal restarts |
| Human access | Use an identity-aware access gateway and group-to-policy mapping | Avoid exposing OpenBao directly and enforce least privilege |
| Workload access | Use Kubernetes/OIDC-based authentication | Bind access to workload identity rather than shared credentials |
| Image integrity | Pin images and verify the external KMS plugin checksum | Prevent a mismatched or modified seal plugin from starting |
| Scheduling | Use a dedicated worker pool | Reduce co-tenancy and side-channel exposure |
| Backup | Use integrated-storage snapshots sent to object storage | Provide a recoverable copy outside the cluster |
| Audit | Start with reliable local log collection and design for a second audit destination | Preserve availability while improving forensic retention |
| Upgrade strategy | Roll standby nodes before the active leader | Avoid failover to an older version |

## 3. Target architecture

### 3.1 Components

The deployed platform consists of:

* **OpenBao server:** API, authentication, policy evaluation, secret engines, token store, lease management, and audit brokering.
* **Integrated storage:** local durable storage with Raft consensus, quorum writes, active/standby behavior, and snapshot/restore.
* **Configuration operator:** applies declarative authentication, policy, secret-engine, and secret-synchronization resources.
* **Secret Definition interface:** custom Kubernetes resources that describe how a secret should be generated or retrieved.
* **KMS seal plugin:** allows OpenBao to auto-unseal using a cloud KMS.
* **Certificate management:** issues and distributes the internal CA and API certificates.
* **Access gateway sidecar:** exchanges an identity-provider JWT for an OpenBao token and forwards requests to the active node.
* **Snapshot agent:** periodically exports integrated-storage snapshots to an object-storage bucket.
* **Monitoring and log collection:** captures health, audit events, failures, latency, seal state, and storage status.

### 3.2 Secret Definition workflow

Teams define secret requirements instead of storing plaintext values. A definition can specify properties such as:

* secret name and destination;
* length and complexity;
* rotation period;
* static versus generated behavior; and
* the owning workload or service.

During deployment:

1. the configuration operator authenticates to OpenBao;
2. OpenBao generates a new value or retrieves an existing value;
3. the value is stored in OpenBao or fetched from the configured engine;
4. the operator renders a standard Kubernetes Secret; and
5. the application consumes the Kubernetes Secret using its existing configuration.

The intended result is that engineers can review the definition and policy without seeing the resulting secret.

## 4. Deployment procedure used in practice

### 4.1 Provision dependencies first

The deployment required the following resources before installing the OpenBao chart:

* certificate management and trust-distribution components;
* dedicated worker capacity;
* persistent storage;
* cloud KMS resources and IAM permissions;
* an object-storage bucket for snapshots;
* image-pull credentials; and
* monitoring and log-collection paths.

The deployment tool in use could not reliably create all namespaces during installation. The implementation therefore created the OpenBao, configuration-operator, and certificate-reloader namespaces in advance, with the Helm ownership labels and annotations required for subsequent releases.

Image-pull credentials were created in the required namespaces and replicated where appropriate.

### 4.2 Configure KMS auto-unseal

Each node starts sealed. It can find its storage, but it cannot use the stored data until the barrier encryption key is unwrapped.

The implementation used a cloud KMS seal. The KMS infrastructure supplied:

* the KMS key or alias;
* the service-account IAM role;
* the region or provider configuration; and
* the permissions required by the OpenBao pods.

A significant operational detail was introduced by newer OpenBao versions: cloud KMS seals are delivered as external plugins. The plugin binary was included in the custom OpenBao image, and its SHA-256 checksum was supplied to the chart.

The image tag and checksum must be changed together. If the image contains a different plugin than the configured checksum, OpenBao refuses to configure the seal and the cluster remains sealed.

### 4.3 Prepare nodes and storage

OpenBao was scheduled onto a dedicated worker pool with matching tolerations and node selectors. The nodes were checked for:

* disabled swap;
* synchronized clocks;
* restricted network access;
* restricted persistent-volume access; and
* appropriate operating-system and container security settings.

Swap was explicitly checked because sensitive data must remain in memory and must not be paged to disk in cleartext.

### 4.4 Deploy in dependency order

The deployment order was:

1. **OpenBao chart** — server, storage, TLS, auto-unseal, sidecars, and snapshot agent.
2. **Configuration operator** — controller that depends on the CA bundle produced by the OpenBao deployment.
3. **OpenBao configuration** — declarative policies, authentication methods, secret engines, audit configuration, and snapshot-agent permissions.

The order matters. The snapshot agent is deployed with the OpenBao chart, but its Kubernetes authentication role, policy, and object-storage credential are created by the later configuration step. Consequently, the first snapshot job may temporarily enter `CreateContainerConfigError` until the configuration deployment completes. Subsequent runs succeed without manual intervention.

### 4.5 Initialize the cluster

The chart included an initialization job that:

* initialized the cluster after the first node became available;
* persisted the root token and recovery keys in a Kubernetes Secret;
* enabled Kubernetes authentication; and
* created the minimum policies and roles required by the configuration operator.

Bootstrap policies were intentionally limited. They were not treated as ordinary continuously reconciled resources because doing so would require long-lived root-level credentials. When bootstrap policies change, the implementation requires:

1. updating the initialization script and image;
2. deploying the change to new environments; and
3. applying the corresponding policy update to existing environments through an authenticated administrative procedure.

Root tokens and recovery material are break-glass credentials only. They must not be placed in repositories, CI/CD variables, or long-lived workload Secrets.

### 4.6 Configure high availability

The reference design used integrated storage with Raft. One node is active and the remaining nodes are hot standbys. If the active node fails, a standby can promote itself.

The reference cluster used five replicas. A new node joined by discovering the current leader, completing a seal-based challenge, receiving the cluster certificate, applying a Raft snapshot, contacting the KMS, and auto-unsealing.

The implementation also required DNS-based API and cluster addresses. Using pod IP addresses would not match the certificate subject-alternative names and could break TLS validation between clients and nodes.

### 4.7 Configure human access

Human access was routed through an identity-aware gateway. The gateway sidecar:

1. received a JWT from the access gateway;
2. exchanged the JWT for an OpenBao token using JWT authentication;
3. selected a role based on identity-provider group membership;
4. cached the token for its lease; and
5. forwarded requests to the active OpenBao node.

The gateway was not treated as the authorization boundary. Each OpenBao role also had to bind the allowed groups through JWT claims. Otherwise, a caller with a valid JWT could attempt to request a more privileged role directly.

### 4.8 Configure snapshots

Integrated storage was backed up through snapshots written to object storage. The process required:

* a bucket created before deployment;
* restricted bucket access;
* an IAM identity for the snapshot agent;
* storage of the snapshot-agent credential in OpenBao; and
* a Kubernetes authentication role and policy for the snapshot agent.

The backup is not considered complete until a restore has been tested in an isolated environment.

## 5. Security hardening applied

The production hardening checklist included the following controls:

* run OpenBao as a non-root user;
* use a read-only root filesystem where possible;
* allow writes only to the persistent data volume and required temporary directories;
* use TLS for client, server, and cluster communication;
* prefer TLS 1.3;
* disable swap;
* disable core dumps because they can expose encryption keys;
* use a dedicated worker pool;
* apply Kubernetes network policies;
* restrict persistent-volume and cloud-volume access;
* keep clocks synchronized;
* disable command history in sensitive container contexts;
* disable `sys/raw` in normal operation;
* avoid root tokens outside bootstrap and break-glass recovery;
* use least-privilege policies for the configuration operator;
* audit API requests; and
* define offboarding procedures for access revocation and credential rotation.

The hardening review also recorded that OpenBao is not itself FIPS-certified, although a deployment may be configured with FIPS-oriented components or images where required.

## 6. Testing performed

### 6.1 Real-application pilot

The pilot was designed around real application scenarios rather than only synthetic API checks. It included:

* deploying the secret manager in a representative environment;
* consuming a static secret from a real application;
* consuming both static and generated secrets from a real application; and
* verifying that applications continued to use ordinary Kubernetes Secrets.

The pilot was tracked as a completed engineering initiative, with no test-type gap recorded.

### 6.2 Token and lease lifecycle investigation

A production-readiness investigation found an important interaction between configuration-operator tokens and dynamic-secret leases:

* a dynamic secret receives a lease associated with the token that created it;
* when the parent token expires, dependent leases may be revoked even if their own TTL is longer; and
* a secret created close to the parent token's maximum TTL can therefore expire earlier than expected.

Two alternatives were tested in a proof-of-concept environment:

1. create a new token for each request with a TTL exceeding the longest secret TTL plus a safety buffer; or
2. cache and renew the operator token, while ensuring it stops using the token early enough that no active secret lease depends on it when the token finally expires.

Both test scenarios confirmed the expected behavior. The selected approach was token caching with a sufficiently large TTL. The design constraint was recorded explicitly: if renewal occurs at roughly 73% of the token lifetime, the remaining lifetime must still exceed the longest dynamic-secret TTL. In simplified form:

```text
remaining token lifetime after the final renewal > longest dynamic-secret TTL
```

The trade-off was also documented: caching reduces unwanted lease revocation but can keep one token alive for months if the operator is not restarted. That token lifetime must therefore be balanced against security requirements and operational restart behavior.

### 6.3 Dynamic security testing

The security assessment used dynamic testing across reconnaissance, authentication, privilege escalation, policy evaluation, SSRF/lateral movement, persistence, and data-exposure scenarios. The assessment targeted OpenBao 2.5.1.

The main results were:

| Test area | Result |
|---|---|
| Seal status and OIDC discovery endpoints | Information was limited and considered intended for health or standards-compliant OIDC behavior |
| Token decoding and malformed JWT handling | Invalid input failed safely; no sensitive information was disclosed |
| Mixed-case user names and password changes | The old password failed; no phantom user or authentication bypass was observed |
| Case-sensitive policy paths | Reads and writes to both case variants were denied; no policy bypass was observed |
| Identity-group privilege escalation | Not exploitable by a non-root token in the tested version; identity-group write access remains highly privileged |
| `sys/raw` privilege escalation | Requires explicitly insecure configuration and authorization; endpoint kept disabled |
| Certificate and JWKS URL requests | Arbitrary URLs are part of the supported configuration model; outbound restrictions should be enforced at the network layer |
| PKI `sign-verbatim` testing | Incorrect PoC parameters returned HTTP 400; the endpoint is intentionally privileged and unsafe |
| TOTP cross-key reuse | Deterministic TOTP behavior was expected; unique secrets per account or key are required |

The overall conclusion was that no exploitable critical finding was confirmed under the default supported configuration. Several test cases still produced important operational requirements: restrict identity-management permissions, keep `sys/raw` disabled, limit outbound network access, and use unique TOTP secrets.

### 6.4 Audit logging tests and design

The security review required audit logging and recommended at least two audit devices because OpenBao refuses to service requests if it cannot write to an enabled audit device.

The implementation discussion made a pragmatic first choice:

* use standard log collection from stdout as the reliable primary path;
* avoid a directly connected network audit device that could block OpenBao operations during a socket problem;
* consider a second file- or object-storage-backed audit device for forensic retention; and
* add checks and dashboard panels for audit failures and latency.

The pilot audit rollout was temporarily blocked by an implementation bug that had recently been fixed. This was tracked as a readiness issue rather than treated as evidence that audit logging could be omitted.

### 6.5 Deployment and Kubernetes tests

Deployment testing covered details that are easy to miss in a generic migration:

* namespace ownership labels and annotations required for Helm adoption;
* image-pull secret availability in every dependent namespace;
* certificate distribution and reload behavior;
* KMS plugin checksum verification;
* auto-unseal after startup;
* DNS-based TLS addresses;
* dedicated worker-pool scheduling;
* the expected temporary snapshot-agent error before configuration completion; and
* the configuration operator's minimum policy permissions.

Certificate rotation used a sidecar to send a reload signal to OpenBao after certificate renewal. The certificate duration was made configurable, and the deployment used a dedicated internal CA distributed through the cluster trust mechanism.

## 7. Results and current conclusions

The implementation produced the following results:

* OpenBao was selected after proof-of-concept evaluation rather than installed as an untested replacement.
* Real applications consumed both static and generated secrets through the Kubernetes integration.
* The Secret Definition model reduced direct human exposure to secret values.
* The token/lease PoC identified a subtle production risk before broad adoption and led to a concrete caching and TTL decision.
* Dynamic security testing did not confirm an exploitable critical finding under the default supported configuration.
* The hardening work identified concrete deployment controls: dedicated nodes, TLS, disabled swap, restricted policies, KMS auto-unseal, disabled `sys/raw`, and controlled root-token use.
* The deployment order and bootstrap limitations were documented, including the need to apply some policy changes manually to existing clusters.
* Snapshot backup and restore became explicit production requirements rather than implicit assumptions.
* Audit logging was treated as a reliability-sensitive dependency: if audit writes block or fail, OpenBao can stop serving requests.

The result was not a claim that OpenBao is secure by default in every environment. The security outcome depends on the surrounding deployment, policy, identity, network, backup, and monitoring configuration.

## 8. Production rollout approach

The planning record described an initial release target in the second quarter of 2026, marked as tentative, with broader secret-management coverage expected later in 2026. The rollout was phased rather than treated as a single cutover.

The production calendar should contain the following stages:

1. development deployment and chart validation;
2. representative staging deployment;
3. pilot applications using static and generated secrets;
4. security and production-readiness review;
5. first production canary or production group;
6. observation period with audit, latency, error, and application-health checks;
7. additional production groups; and
8. legacy-secret cleanup and final ownership review.

Each stage should record:

* responsible team;
* change window;
* approval requirements;
* freeze-window checks;
* success criteria;
* rollback method;
* evidence location; and
* decision to proceed, pause, or revert.

## 9. Upgrade and rollback experience

OpenBao upgrades were not treated as true zero-downtime changes. With the documented procedure, expected downtime should be short, but it is still a change-window concern.

The upgrade procedure has several important constraints:

* review the release notes for all intervening versions;
* take and verify a snapshot before the upgrade;
* avoid failing over to an older leader;
* update standby nodes before the leader;
* use an `OnDelete` strategy or equivalent controlled rollout; and
* use a post-upgrade job or runbook to automate the correct order.

OpenBao does not guarantee that every storage-format change is backward-compatible. A rollback after a storage change may therefore require restoring the storage from a snapshot rather than deploying the previous image.

## 10. Ongoing expectations

The operating team is expected to:

* review and apply OpenBao security updates through the vulnerability-management process;
* update deployments at least quarterly or according to the approved maintenance policy;
* keep image provenance and digests recorded;
* maintain KMS, IAM, TLS, and certificate-rotation procedures;
* maintain at least two tested audit and log-consumption paths where required by the security model;
* monitor audit failures, request latency, authentication failures, seal state, Raft quorum, storage, and certificate expiry;
* test backup restoration at least twice per year;
* review configuration-operator policies with the security team;
* rotate break-glass credentials when authorized personnel change;
* review secret ownership and policy boundaries when workloads change; and
* keep incident, upgrade, recovery, and secret-rotation runbooks current.

## 11. Lessons learned

### Treat secret management as a platform, not a package

Installing the server is only one part of the work. Identity, policy, KMS, TLS, storage, backups, audit behavior, monitoring, and recovery must be designed together.

### Test token lifetimes with real lease behavior

Token TTLs and dynamic-secret TTLs are coupled. A configuration that looks correct from the token perspective can revoke application secrets earlier than intended.

### Keep privileged functionality disabled

Several apparent vulnerabilities required explicitly unsafe configuration or privileged permissions. The practical mitigation was not to grant those permissions and not to enable `sys/raw` during normal operation.

### Make bootstrap behavior explicit

The initialization job creates important resources only once. A later edit to a Kubernetes ConfigMap does not automatically change the policies already stored in OpenBao. Existing clusters require an explicit update procedure.

### Design audit logging for failure behavior

Audit logging is part of the request path. An audit destination that blocks can block OpenBao itself. Reliability, retention, and forensic requirements must be balanced when choosing audit devices.

### Do not assume backup means recoverability

A configured snapshot job is not proof of recovery. Restore into an isolated environment and verify that policies, authentication, secret data, and application access all recover correctly.

## 12. Final assessment

The migration experience supports OpenBao as a viable secrets-management platform when it is deployed and operated as a security-critical distributed service.

The most important conclusion is that the migration was successful because it combined:

* a clear reason to move away from encrypted-file secret handling;
* a phased and compatible secret interface;
* per-cluster isolation;
* KMS-backed auto-unseal;
* controlled bootstrap and least-privilege policies;
* real-application validation;
* token and lease testing;
* targeted security testing;
* documented backup and upgrade procedures; and
* explicit operational ownership.

OpenBao should not be presented as a drop-in replacement that eliminates operational responsibility. It provides the building blocks; the security and reliability outcome comes from the surrounding design and the discipline of operating it.

## References

* [OpenBao documentation](https://openbao.org/docs/)
* [OpenBao source repository](https://github.com/openbao/openbao)
* [OpenBao security model](https://openbao.org/docs/internals/security/)
* [OpenBao integrated storage](https://openbao.org/docs/internals/integrated-storage/)
* [OpenBao audit devices](https://openbao.org/docs/audit/)
* [OpenBao seal concepts](https://openbao.org/docs/concepts/seal/)
* [OpenBao token concepts](https://openbao.org/docs/concepts/tokens/)
* [OpenBao production hardening guidance](https://openbao.org/docs/concepts/)
