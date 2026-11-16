# PodiumD Hosting Standards

This document sets out what Dimpact expects from software suppliers, so that their software can be deployed and operated reliably on the PodiumD platform hosted by SSC Hosting.

| Section | What you find there |
| --- | --- |
| [About this document](#about-this-document) | Purpose, roles, requirement levels and terms |
| [Priority requirements](#priority-requirements) | The three items to address first |
| [The hosting platform](#the-hosting-platform) | The environment that the software is deployed into |
| [Requirements](#requirements) | All requirements, with an index by ID |
| [Proposed additions](#proposed-additions) | Suggestions for new standards, not yet agreed |
| [References](#references) | Background reading |
| [Version history](#version-history) | Versions of this document |

## About this document

### Purpose

PodiumD is a collection of open-source, Common Ground-compliant components. External software suppliers build the components, funded by the municipalities. PodiumD replaces the E-suite workflow system that many municipalities use.

SSC Hosting runs PodiumD on Microsoft Azure, with development, test, acceptance and production (OTAP) environments for multiple Dutch municipalities.

This document formalizes what Dimpact, the contracting party, expects from software suppliers who publish software to the platform.

The same applications are deployed for many municipalities, in several environments each. Fully automated deployment is therefore a hard requirement for every application (see [COMP008.1-100percent-automated]).

The document lives in a Git repository and changes through pull requests.

### Who does what

| Party | Role |
| --- | --- |
| Dimpact | Contracting party. Curates and provides all Helm charts in one single repository. Provides examples of parameter naming for the umbrella Helm chart. Provides the URLs of public-facing applications, and agrees addresses, host names and CNAMEs with the municipalities. |
| SSC Hosting | Hosting provider. Builds and maintains the Azure infrastructure and the Kubernetes clusters. Copies container images from the supplier's registry to its own registry and scans them. Deploys the applications. |
| Suppliers | Build the components and deliver them as containers with Helm charts that meet the requirements in this document. |

### Requirement levels

Requirements use the key words of [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Key word | Meaning |
| --- | --- |
| MUST, MUST NOT | Required or forbidden. |
| SHOULD, SHOULD NOT | Recommended or discouraged. Deviate only for a good reason. |
| MAY | Optional. |

The level of a requirement can become stricter or looser in a later version of this document.

Each requirement has an ID, for example `COMP014.2-probe-config`. Use the ID to refer to a requirement.

### Terms

| Term | Meaning |
| --- | --- |
| OTAP | *Ontwikkel, Test, Acceptatie, Productie*: the development, test, acceptance and production environments. Short names: ONTW, TEST, ACCP, PROD. |
| Platform | A set of components that together provide a solution, for example Open Zaak and ZAC. |
| Installation (*installatie*) | Deploying a component, for example running the Helm chart so that Open Zaak and ZAC run technically. |
| Configuration (*configuratie*) | The parameters that let a component do what it has to do, for example the connection between ZAC and Open Zaak. |
| Functional setup (*inrichting*) | The functional content of a component, possibly with very specific configuration, for example case types (*zaaktypen*) and processes. |

The last four definitions come from supplier Maykin Media.

## Priority requirements

These three "hot items" are the minimum Kubernetes stability requirements for delivered software. They have the highest priority.

The components of all suppliers have been reviewed against them. Where a component does not comply, the supplier is asked to include the fix in its planned work. The goal is that all software complies by the end of Q1 2026.

| Hot item | What is expected | Details |
| --- | --- | --- |
| Health checks and probes | Clear health checks and probes on standard API endpoints. | [COMP014-healthchecks], especially [COMP014.1-health-states] and [COMP014.2-probe-config] |
| Upgradability | Applications must be upgradable with zero downtime. Use database techniques that allow a rolling upgrade of components, migration of the database, and rollback of the database when required. | [COMP006.2-upgrade-time-limit] |
| High availability | Critical components must survive a zone failure. Suppliers should also start thinking about deployment to multiple regions, and what that means for the database. | [COMP014.3-redundancy] |

## The hosting platform

This section describes the environment that the software is deployed into.

### Environments

Each municipality uses a set of Azure subscriptions, one per OTAP environment. This simplifies billing and separates environments, customers and suppliers from each other. Not every customer or supplier needs all four environments.

| Environment | Purpose |
| --- | --- |
| Development (*ontwikkel*) | Verify that the software works in a production-like environment. Developers are free to deploy container versions and new Helm charts here. SSC Hosting makes changes to Azure resources and the Kubernetes cluster, in an updated version of the environment. |
| Test | For municipalities to test new application features and to train staff. |
| Acceptance | Like-for-like with production: the same hardware resources, and copies of production data. |
| Production | The environment with maximum uptime, monitoring and resources. One per municipality. |

To save cloud costs, all development, test and acceptance environments are switched off in the evening and at weekends. An environment can stay on for a short period when there is a reason for it.

### Cloud provider

Dimpact currently uses Microsoft Azure. In the future it MAY move to another platform (Google, AWS, private cloud) or use several at once. Deployment work should therefore be generic and use open standards.

### Azure services

New Azure resources can be discussed. This is the current list.

| Function | Service | How it is used |
| --- | --- | --- |
| Kubernetes | AKS | Standard AKS with two autoscaling node groups: one for application workloads, one for management workloads. |
| Container registry | ACR | With vulnerability scanning. |
| SQL database | Azure Database for PostgreSQL flexible server | PostgreSQL 14 is the standard. One server per environment, with one database per application. The database time zone is UTC. |
| Edge load balancer | Azure Application Gateway | The internet-facing endpoint. It terminates TLS, redirects HTTP to HTTPS, rewrites URLs and routes traffic to the right cluster. There is one gateway for ONTW and TEST, and one for ACCP and PROD. The on-premises Palo Alto and gateway services may take over this function in the future. |
| Storage | Azure Storage Accounts | Blob storage and Azure File Shares (SMB and NFS). |
| Secrets and certificates | Azure Key Vault | The single source of truth for all certificates and secrets. |
| Identity | Entra ID | See [Identity](#identity). |
| Deployment | Azure DevOps | CI/CD for infrastructure, application deployment and management tasks. |

### Deployment

An Azure DevOps pipeline deploys applications to the Kubernetes cluster by installing a Helm chart. Other infrastructure is built and maintained with Terraform and with bash scripts that run `az` CLI commands.

### Infrastructure life cycle

- Most Azure resources, such as storage and databases, have very long life cycles.
- Kubernetes is updated every month with Azure Fleet Manager, in the order ONTW, TEST, ACCP, PROD, with one week between environment types. A non-critical Kubernetes patch has therefore run in ONTW for three weeks before it reaches production.
- Upgrades to other resources, such as the database server and storage, are announced well in advance. Suppliers can then test their applications against the new version and plan an upgrade path if needed.

### Infrastructure monitoring

The infrastructure (Kubernetes, edge gateway, databases and so on) MUST be monitored for errors, as exported to Azure. Other metrics, such as CPU capacity and disk IO, should be used for rightsizing and autoscaling of resources.

### Application Gateway settings

- One HTTP header rewrite is applied, for HSTS.
- HTTP/2 support is disabled.
- All other settings are the Azure defaults; see the Microsoft Azure documentation. Any change to a default will be recorded in this document.

### TLS certificates

- Public-facing services use certificates from Let's Encrypt. The cert-manager application in the cluster requests them automatically for every web-facing service. Certificates and keys are stored in Azure Key Vault and deployed to the listeners of the Application Gateway.
- Certificates used internally between applications and legacy systems are out of scope.

## Requirements

Suppliers are expected to deliver applications in a way that makes them secure, scalable, stable, manageable and easy to deploy.

### Index

**Core requirements**: application structure, deployment and configuration.

| ID | Summary |
| --- | --- |
| [COMP001-naming] | Naming conventions for services, components and resources |
| [COMP002-versioning] | How the PodiumD version is defined |
| [COMP002.1-version-api] | Standard version endpoint with application metadata |
| [COMP003.1-app-as-containers] | Applications are delivered as containers |
| [COMP003.3-container-registry] | Containers are available from a public container registry |
| [COMP003.3-semantic-versioning] | Container image tags use semantic versioning |
| [COMP004-timezone] | UTC for logging and monitoring |
| [COMP005-software-supply-chain] | Vulnerability scanning and mitigation |
| [COMP006.1-stop-start-safe] | Withstand sudden stops and starts |
| [COMP006.2-upgrade-time-limit] | Rolling upgrades with zero downtime |
| [COMP006.3-rollbacks] | Rollback of deployments |
| [COMP007.1-helm] | A Helm chart for every application |
| [COMP007.3-dependancy-apps] | Dependencies are deployed as Helm sub-charts |
| [COMP007.4-storage] | The chart supports SSC's storage choices |
| [COMP007.5-ingress] | The chart supports SSC's ingress choices |
| [COMP007.5-database] | The chart supports an external cloud database |
| [COMP008.1-100percent-automated] | Complete deployment from the Helm chart |
| [COMP009-internal-communication] | Components communicate inside the cluster, through APIs |
| [COMP010-sql-migration-scripts] | Built-in database migrations with rollback |
| [COMP011-no-local-storage] | No persistent data on local container storage |
| [COMP011-permanent-data] | Data that needs a backup goes to an external file share |
| [COMP012-secrets] | Secrets are kept in the Kubernetes secrets store |
| [COMP013-ssl] | Endpoints are plain HTTP; TLS ends at the gateway |

**Health checks and monitoring**: observability, health and metrics.

| ID | Summary |
| --- | --- |
| [COMP014-healthchecks] | Probe endpoints for container health |
| [COMP014.1-health-states] | Defined health states: healthy, degraded, unhealthy, starting, stopping |
| [COMP014.2-probe-config] | Liveness, readiness and startup probes |
| [COMP014.3-redundancy] | Multi-zone deployment, retries and load balancing |
| [COMP014.4-pdb] | Pod Disruption Budgets for critical services |
| [COMP015.1-logging-to-stdout] | All logging goes to STDOUT |
| [COMP016.2-utc-logs] | Log lines have UTC timestamps |
| [COMP017-golden-signals] | Metrics for the four golden signals |
| [COMP017.1-latency-metrics] | Request duration and response time |
| [COMP017.2-traffic-metrics] | Requests per second and concurrent connections |
| [COMP017.3-error-metrics] | Error rate and error types |
| [COMP017.4-saturation-metrics] | Resource utilization and queue depth |
| [COMP018-alerting] | Alert design principles |
| [COMP018.1-slo-alerting] | Alerts are tied to Service Level Objectives |
| [COMP019-slo] | Service Level Objectives and error budgets |

**Operations**: resources, recovery and releases.

| ID | Summary |
| --- | --- |
| [COMP020-resource-recommendations] | CPU and memory estimates, autoscaling configuration |
| [COMP021-disaster-recovery] | Recovery time and recovery point objectives, backup, recovery tests |
| [COMP022-release-notes] | Content of release notes |

**Security, compliance and performance**

| ID | Summary |
| --- | --- |
| [COMP023-security] | Container security, network security and secret management |
| [COMP024-sbom] | Software Bill of Materials in SPDX or CycloneDX format |
| [COMP025-api-practices] | API design, versioning and documentation |
| [COMP028-compliance] | Audit logging and GDPR |
| [COMP029-performance] | Response time targets, throughput and caching |
| [COMP030-cost-optimization] | Resource efficiency and cost visibility |

Three sections have no ID yet: [URLs and DNS](#urls-and-dns), [Identity](#identity) and [Documentation](#documentation).

Withdrawn IDs: COMP007.2-operators, COMP026-service-mesh, COMP027-documentation.

### Core requirements

#### Naming [COMP001-naming]

Uniform names for applications and parameters keep repeated configuration to a minimum.

- An application MUST have one single name that is used throughout the service, preferably between 5 and 10 characters.
- Service names MUST follow a consistent pattern: `{domain}-{function}-service`.
- Service names MUST use lowercase, with hyphens as separators.
- Service names MUST be descriptive and reflect the business domain.
- Service names MUST be the same in all OTAP environments.
- Helm chart names MUST match the service name.
- Container image names MUST use the service name as base.
- API endpoint paths SHOULD use the service domain name.

| Good | Avoid |
| --- | --- |
| `klant-management-service`, `zaak-api-service`, `document-storage-service` | `service1`, `api`, `app-x`, `temp-service` |

Dimpact will provide examples of parameter naming for the umbrella Helm chart.

#### Versioning [COMP002-versioning]

PodiumD is a suite of sub-applications. The PodiumD version is defined by the versions of those sub-applications.

#### Version endpoint [COMP002.1-version-api]

- Each application or component MUST expose a standardized version endpoint at `/api/v1/version` or `/health/version`.
- The endpoint MUST be accessible without authentication.
- The endpoint SHOULD be included in health check monitoring.
- The endpoint SHOULD return JSON with:
  - the application or component name
  - the semantic version number
  - the build date and time
  - the Git commit SHA (optional, but recommended)
  - the dependencies and their versions (optional, but recommended)

```json
{
  "name": "zaak-api-service",
  "version": "2.5.1",
  "buildDate": "2024-10-15T14:30:00Z",
  "gitCommit": "a3f2d1c",
  "dependencies": {
    "postgres": "14.5",
    "redis": "7.0.5"
  }
}
```

#### Containers [COMP003]

- **COMP003.1-app-as-containers**: Applications MUST be delivered as one container or a set of containers.
- **COMP003.3-container-registry**: The containers MUST be available from a public container registry such as Docker Hub. Containers that are not open source MAY be secured with a key or token.
- **COMP003.3-semantic-versioning**: Container image tags MUST be semantically versioned. Pre-release versions MAY be deployed as version plus keyword, for example `1.2.3-snapshot`.

SSC Hosting copies the containers from the supplier's registry to a container registry in the SSC Azure environment.

#### Time zone [COMP004-timezone]

- Applications SHOULD use UTC for logging and monitoring: all logs and metrics are in UTC. The default time zone of the database is also UTC.
- Anything shown to an end user should be in the appropriate local time zone.

See also [COMP016.2-utc-logs].

#### Software supply chain [COMP005-software-supply-chain]

- Delivered containers MUST already have been scanned for vulnerabilities by the supplier.
- A flagged vulnerability MUST be fixed, mitigated, or waived by the system architects of the supplier, Dimpact and SSC Hosting together, within a specified time limit. For high and critical CVEs the limit is in [COMP023-security].
- SSC Hosting also scans all containers regularly.

Examples of tools: [SonarQube](https://www.sonarsource.com/products/sonarqube/) for scanning code, and [GitHub container scanning](https://github.com/marketplace/actions/container-scan) for scanning containers.

#### Stops, starts and scaling [COMP006.1-stop-start-safe]

- **COMP006.1-stop-start-safe**: Applications MUST withstand sudden stops and starts, for example when a node fails or a container moves to another node.
- To survive a node failure without downtime, every component MUST run at least two instances that take over each other's work automatically, and MUST use shared storage.
- Applications SHOULD handle load-based scaling up and down of containers gracefully, except where scaling is inappropriate.
- Applications MAY be deployed with init containers, and with startup and shutdown scripts.

#### Upgrades [COMP006.2-upgrade-time-limit]

- Any upgrade of a component SHOULD be possible as a rolling upgrade with zero downtime, unless a waiver has been granted.
- Upgrades that involve database changes are currently exempt from this requirement.
- A deployment that requires any downtime requires an architectural review.

#### Rollbacks [COMP006.3-rollbacks]

- Deployments SHOULD provide a rollback function. This is especially needed when a migration stops on errors during a deployment (see [COMP010-sql-migration-scripts]).

#### Helm charts [COMP007]

- **COMP007.1-helm**: All applications MUST come with Helm charts. An application MAY use Helm sub-charts. It MUST itself be deployable as a sub-chart, so that it can be part of a complete "umbrella" installation of the PodiumD platform.
- **COMP007.3-dependancy-apps**: Applications that the application depends on MUST be deployed as Helm sub-charts.
- **COMP007.4-storage**: If file storage is required, the Helm chart MUST support SSC's storage choices.
- **COMP007.5-ingress**: If ingress is required, the Helm chart MUST support SSC's ingress choices.
- **COMP007.5-database**: If a database is required, the Helm chart MUST support the use of an external cloud database.
- Helm charts SHOULD be available from the same artifact repository as the containers, and SHOULD be in OCI format.
- Over time, all parties should work to harmonize variable naming in the Helm charts.

PodiumD as a whole is deployed in one single Kubernetes namespace, `podiumd`.

#### Automated deployment [COMP008.1-100percent-automated]

- PodiumD is deployed to many customers and environments. All components of an application MUST therefore be deployable completely from the Helm chart.
- "Completely" means: the containers run correctly in the Kubernetes cluster with the correct deployment parameters and secrets, and connect to other resources such as databases and other PodiumD applications.
- Final configuration inside the application is out of scope. [Terms](#terms) explains the difference between installation, configuration and functional setup.

#### Communication between components [COMP009-internal-communication]

- Communication between components in the same Kubernetes cluster SHOULD take place inside the cluster.
- Communication and data transfer between components SHOULD go through the APIs.

#### Database migrations [COMP010-sql-migration-scripts]

- Applications MUST have all database migrations built in, or provide configuration scripts for a migration tool such as Flyway or Liquibase, or for the tool that comes with the framework (for example Django).
- Database migrations MUST provide a rollback function.

#### State and storage [COMP011]

- Applications SHOULD be stateless and commit all data to a database.
- **COMP011-no-local-storage**: Persistent files that contain any data MUST NOT be written to local storage in the container. They are written to a Kubernetes PersistentVolumeClaim (PVC) instead.
- **COMP011-permanent-data**: Permanent data that needs a backup MUST be written to a file share that is created outside the Kubernetes cluster.
- Files SHOULD be written to an object store such as Azure Blob Storage. Files MAY also be stored on other storage types. The order of preference is:
  1. Object storage
  2. An SMB file share (shared volume)
  3. Persistent volume storage

#### Configuration and secrets [COMP012-secrets]

- All application configuration SHOULD be done through environment variables.
- Environment variables SHOULD be set through ConfigMaps in the Helm chart.
- **COMP012-secrets**: Secret or sensitive information MUST be written to the Kubernetes secrets store.
- Helm charts SHOULD be able to read secrets directly from the secrets store. Templates then refer to a secret (`env.valueFrom.secretKeyRef`) instead of embedding the secret value in `values.yaml`. This is known as the `secretKeyRef` or externalized secrets pattern.
- Secrets are read from Azure Key Vault into the secrets store at deployment time.

More rules for secrets are in [COMP023-security].

#### TLS termination [COMP013-ssl]

- Application endpoints MUST be presented as unencrypted HTTP.
- The public-facing Azure Application Gateway terminates TLS. It guarantees a secure path to the Kubernetes cluster through Azure Virtual WAN and Virtual Hub. Its listeners get their Let's Encrypt certificates at deployment time.

#### URLs and DNS

- Applications MUST support access from any URL, and from several URLs at the same time.
- There should be no restriction on changing the URL of an application at any time. If a municipality changes the URL that citizens use, this should not require any database updates.

More information: [Toegang API's via verschillende domeinnamen waaronder interne toegang](https://dimpact.atlassian.net/wiki/spaces/PCP/pages/175865864/Toegang+API+s+via+verschillende+domeinnamen+waaronder+interne+toegang).

#### Identity

- Entra ID SHOULD be the only method of identity management.
- If another identity tool is used, a waiver MUST be obtained, with a plan for future migration to Entra ID.

### Health checks and monitoring

Monitoring combines logs and metrics to find out whether the application works as expected. The aim is to notice a problem before the end user does, and to have time to fix the problem or its underlying cause.

#### Health checks [COMP014-healthchecks]

- Applications MUST provide probe endpoints, so that starting, running and failing containers are handled correctly.

#### Health states [COMP014.1-health-states]

Applications MUST clearly define and document these operational states:

| State | Meaning | What it looks like |
| --- | --- | --- |
| Healthy (normal) | The system operates within expected parameters. | All dependencies are available. Response times are within the SLA. Error rates are below the threshold. Resources are within normal limits. |
| Degraded | Reduced functionality, but still operational. | Some non-critical dependencies are unavailable. Response times are elevated but acceptable. The system runs in fallback mode. This should trigger warnings, not critical alerts. |
| Unhealthy (failed) | The system does not function correctly. | Critical dependencies are unavailable. Requests cannot be processed. There are data consistency issues. This should trigger immediate alerts. |
| Starting | The system is initializing. | Loading configuration, establishing database connections, warming up caches. Not ready for traffic. |
| Stopping | A graceful shutdown is in progress. | Completing in-flight requests, closing connections, cleaning up resources. |

#### Probes [COMP014.2-probe-config]

Applications MUST implement all three types of probe:

| Probe | What it decides | Rules |
| --- | --- | --- |
| Liveness | Whether the container needs a restart. | MUST be simple and fast (under 1 second). SHOULD NOT check dependencies: it is a low-cost internal heartbeat that makes no external calls. Example: an HTTP endpoint that returns 200 OK if the application can accept requests. |
| Readiness | Whether the container can receive traffic. | SHOULD check critical dependencies before traffic is routed: a simple database ping, required APIs, cache or broker connectivity, loaded configuration and secrets. MAY have a longer timeout than the liveness probe. Should take the pod out of service gracefully. |
| Startup | Whether a slow-starting application has finished initializing, including migrations and warm caches. Liveness and readiness checks are held off until then, which prevents premature restarts. | Should have a generous initial delay and timeout. |

Beyond the basics:

- A broader readiness check should also look at resource pressure: CPU throttling, memory near its limit, disk space and inode thresholds, and queue or backlog size against the SLA. Thread or worker pool saturation, recent spikes in the error rate and, optionally, feature flags or the freshness of external authentication tokens should also be monitored. The check itself must stay fast (under 100 ms).
- Complement the probes with a `/metrics` endpoint (see [COMP017-golden-signals]) that exposes latency histograms, error counters, request concurrency, queue depth, garbage collection pauses and custom saturation gauges. This enables autoscaling (HPA or KEDA) and alerts on trends, not just on a probe that fails.

Probe settings:

| Setting | Guideline |
| --- | --- |
| Initial delay | Allow for a realistic startup time. |
| Failure threshold | Avoid false positives from temporary issues. |
| Success threshold | Prevent flapping. |
| Period | Balance responsiveness against load. |
| Timeout | Allow for network variability. |

Example of the part of a Helm chart that defines the probes of a component:

```yaml
startupProbe:
  initialDelaySeconds: 40
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 12
  successThreshold: 1

livenessProbe:
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 12
  successThreshold: 1

readinessProbe:
  initialDelaySeconds: 0
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 3
  successThreshold: 1

probesEnabled:
  startupProbe: true
  livenessProbe: true
  readinessProbe: true
```

See [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) in the Kubernetes documentation.

#### Redundancy and fault tolerance [COMP014.3-redundancy]

Multi-zone deployment:

- Production deployments SHOULD spread replicas across multiple Azure Availability Zones.
- Critical services SHOULD run at least 3 replicas.
- Pod anti-affinity rules SHOULD be used to prevent replicas from landing on the same node.

Retries:

- Applications MUST use exponential backoff when they retry failed requests.
- The maximum number of retries should be configurable.
- The circuit breaker pattern SHOULD be implemented for external dependencies.

Load balancing:

- Traffic MUST be distributed across multiple instances.
- Session affinity SHOULD be avoided where possible, for better distribution.
- Health checks MUST remove unhealthy instances from rotation.

#### Pod Disruption Budgets [COMP014.4-pdb]

- Critical services MUST define Pod Disruption Budgets.
- The minimum number of available pods should ensure service continuity during updates.
- The budget should balance update speed against availability requirements.

#### Logging [COMP015, COMP016]

- **COMP015.1-logging-to-stdout**: Applications MUST stream all logging to STDOUT. These logs are captured and sent to the appropriate logging endpoint.
- **COMP016.2-utc-logs**: Log lines MUST have timestamps in UTC, without a time zone.
- The log level MUST be configurable through a `LOGLEVEL` environment variable, with the standard levels DEBUG, INFO, WARN, ERROR and FATAL.
- More verbose logging SHOULD be switched on by setting an environment variable and restarting the container.
- Logs SHOULD be written in JSON, so that the logging system can parse them. See [JSON logging](https://betterstack.com/community/guides/logging/json-logging/).
- Applications MAY also write logging in a form that can be read inside the application, but this MUST be in addition to the logs on STDOUT.

#### Metrics [COMP017-golden-signals]

Applications MUST provide metrics for the four golden signals:

| Signal | ID | What to measure | Example metrics |
| --- | --- | --- | --- |
| Latency | COMP017.1-latency-metrics | Request duration and response time. Latency of successful and failed requests separately. Percentile distributions (p50, p95, p99). | `http_request_duration_seconds`, `api_response_time_milliseconds` |
| Traffic | COMP017.2-traffic-metrics | Requests per second. Concurrent connections. Active users. | `http_requests_total`, `active_connections`, `concurrent_users` |
| Errors | COMP017.3-error-metrics | Error rate (errors divided by total requests). Error types and codes. Failed transactions. | `http_requests_failed_total`, `transaction_errors_total`, `error_rate_percentage` |
| Saturation | COMP017.4-saturation-metrics | Resource utilization (CPU, memory, disk, network). Queue depth. Thread pool usage. Database connection pool usage. | `cpu_usage_percentage`, `memory_usage_bytes`, `queue_depth`, `db_connection_pool_active` |

Applications SHOULD also provide metrics that make sense for the application itself, for example the number of transactions, transaction latency, and incomplete and failed transactions.

How to implement metrics:

- Metrics SHOULD be exposed in Prometheus format at a `/metrics` endpoint.
- Metrics MUST have labels for filtering: environment, version and instance.
- Collecting metrics SHOULD have minimal impact on performance (under 1% overhead).
- Counters SHOULD only ever increase.
- Gauges SHOULD represent current values.
- Histograms SHOULD be used for latency.

A tool such as Prometheus can then scrape the metrics for graphs, analysis and alerting. Metrics are an ongoing discussion: they will be updated and improved throughout the life cycle of the application.

Background: [Golden signals in Kubernetes](https://sysdig.com/blog/golden-signals-kubernetes/).

#### Alerting [COMP018-alerting]

Design principles:

- Alerts MUST only notify when human intervention is required.
- Alerts SHOULD be based on symptoms (impact on users), not on causes.
- Every alert MUST be actionable.
- An alert description MUST include:
  - what is broken
  - the impact on users or the business
  - the next steps for investigation
  - links to the relevant runbooks

Severity levels:

| Level | When | Action |
| --- | --- | --- |
| Critical (P1) | Production is down or major functionality is broken. | Immediately. |
| Warning (P2) | Degraded performance or a potential issue. | Within business hours. |
| Info (P3) | Informational. | No immediate action. |

**COMP018.1-slo-alerting**: alerts based on Service Level Objectives.

- Alerts SHOULD be tied to Service Level Objectives (SLOs).
- Use the burn rate of the error budget for alerting thresholds.
- Use fast-burn alerts (windows of 1 to 2 hours) for immediate issues.
- Use slow-burn alerts (windows of 24 to 72 hours) for trends.

Threshold guidelines:

| Signal | Alert when |
| --- | --- |
| Error rate | More than 1% of requests fail, or as the SLO says. |
| Latency | p95 is above the SLO threshold. |
| Saturation | Resource utilization is above 80%. |
| Availability | Uptime is below the SLO, for example 99.9%. |

Preventing alert fatigue:

- Avoid alerting on predicted problems.
- Use multi-window, multi-burn-rate alerting.
- Review and tune alert thresholds regularly.
- Remove alerts that do not result in action.

#### Service Level Objectives [COMP019-slo]

Each production service MUST define:

| What | Content |
| --- | --- |
| Service Level Indicators (SLIs) | Availability: the percentage of successful requests. Latency: request response time (p95, p99). Throughput: requests processed per second. Error rate: the percentage of failed requests. |
| Service Level Objectives (SLOs) | A target value for each SLI and a measurement window, for example 30 days. Examples: availability 99.9% (43.2 minutes of downtime per month); 95% of requests under 200 ms; error rate under 0.1% of requests. |
| Error budgets | Calculated from the SLO: a 99.9% objective leaves a budget of 0.1%. The budget balances speed of delivery against reliability. When it is exhausted, feature releases are frozen and the focus moves to reliability. |

- SLOs MUST be documented in the service documentation.
- SLOs SHOULD be reviewed every quarter.
- SLO violations MUST trigger a post-incident review.

### Operations

#### Resources [COMP020-resource-recommendations]

- As part of the Helm chart, applications MUST provide an estimate of the CPU, disk and memory needed to operate the application at a stated baseline load.
- Applications MUST specify requests, the guaranteed allocation: minimum CPU (in millicores) and minimum memory (in MB or GB).
- Applications MUST specify limits, which prevent resource hogging: maximum CPU (in millicores) and maximum memory (in MB or GB).

Example: for application **X** to process **Y** transactions per minute, we recommend:

- Requests: 1000 millicores CPU, 512 MB memory
- Limits: 2000 millicores CPU, 1 GB memory
- 3 container replicas (the minimum for high availability)
- File storage type and size
- Network bandwidth, if applicable

The autoscaling configuration states:

- the thresholds for the Horizontal Pod Autoscaler (HPA)
- the target CPU utilization (typically 70 to 80%)
- the minimum and maximum number of replicas
- the scale-up and scale-down behavior

#### Disaster recovery [COMP021-disaster-recovery]

Each application MUST define:

| Objective | Meaning |
| --- | --- |
| Recovery Time Objective (RTO) | The maximum acceptable downtime: the time to restore service after an incident. |
| Recovery Point Objective (RPO) | The maximum acceptable data loss. It sets the frequency of backups or replication. |

Both should align with the business requirements.

Backup and restore:

- The database backup strategy MUST be documented.
- The backup frequency MUST meet the RPO.
- Restore procedures MUST be tested every quarter.
- A backup retention policy MUST be defined.

Disaster recovery (DR) testing:

- DR procedures MUST be tested at least once a year.
- Chaos engineering tests SHOULD be performed regularly.
- Failover and failback procedures MUST be documented.
- DR test results MUST be documented.

#### Release notes [COMP022-release-notes]

The release notes of an application MUST contain:

- a full description of the parameters needed for the Helm deployment
- a full description of possible output parameters, for example API keys that are generated at deployment time
- dependencies on earlier versions of the application
- storage requirements
- database requirements
- external access and ingress requirements
- **breaking changes**, clearly marked and explained
- **security fixes**, with CVE numbers if applicable
- **migration steps**, if the upgrade needs them
- **rollback procedures**, in case of issues
- the **performance impact** of changes
- **known issues** in this release

Release notes SHOULD follow the [Keep a Changelog](https://keepachangelog.com/) format.

#### Documentation

- Documentation MUST be updated with each release.
- Documentation SHOULD be version-controlled.
- Documentation SHOULD be accessible to all stakeholders.
- Outdated documentation MUST be clearly marked.

### Security, compliance and performance

#### Security [COMP023-security]

Container security:

- Containers MUST run as a non-root user.
- Containers MUST use a read-only root filesystem where possible.
- Containers MUST drop all unnecessary Linux capabilities.
- Container images MUST be scanned for vulnerabilities before deployment (see [COMP005-software-supply-chain]).
- High and critical CVEs MUST be addressed within 7 days.

Network security:

- Network policies MUST be defined to restrict pod-to-pod communication.
- Only the necessary ports SHOULD be exposed.
- All external communication MUST use TLS 1.2 or higher.
- API authentication and authorization MUST be implemented.

Secret management:

- Secrets MUST NEVER be committed to source control.
- Secrets MUST be stored in Azure Key Vault.
- Secrets SHOULD be rotated regularly, at least once a year.
- Applications SHOULD support secret rotation without a restart where possible.

#### Software Bill of Materials [COMP024-sbom]

A Software Bill of Materials (SBOM) helps with vulnerability tracking and compliance.

- Applications SHOULD provide an SBOM in SPDX or CycloneDX format.
- The SBOM SHOULD include all direct and transitive dependencies.
- The SBOM SHOULD be updated with each release.

#### APIs [COMP025-api-practices]

REST principles:

- Use the standard HTTP methods (GET, POST, PUT, PATCH, DELETE).
- Use plural nouns for collections (`/users`, `/orders`).
- Use nested resources for relationships (`/users/{id}/orders`).
- Return appropriate HTTP status codes.

Versioning:

- The API version MUST be part of the URL path (`/api/v1/resource`).
- A new major version MUST be used only for changes that are not backward compatible.
- Old API versions SHOULD be supported for at least 12 months after deprecation.

Error handling:

- Errors MUST be returned in a consistent JSON format.
- Include an error code, a message and details.
- Use appropriate HTTP status codes.

Rate limiting:

- APIs SHOULD implement rate limiting.
- Rate limit headers SHOULD be returned.
- Rate limits SHOULD be documented.

Documentation:

- APIs MUST provide OpenAPI (Swagger) documentation.
- The documentation MUST be kept up to date.
- Example requests and responses SHOULD be provided.

#### Compliance and audit [COMP028-compliance]

Audit logging:

- All authentication attempts MUST be logged.
- All authorization failures MUST be logged.
- All data access SHOULD be logged, where applicable.
- Logs MUST be retained as regulations require.
- Logs MUST be tamper-proof.

Applications that handle personal data MUST:

- comply with the GDPR
- implement data retention policies
- support the right to erasure (the right to be forgotten)
- support data portability
- maintain audit trails

#### Performance [COMP029-performance]

Response time targets:

| Type of request | Target (p95) |
| --- | --- |
| Interactive requests | Under 200 ms |
| API calls | Under 500 ms |
| Database queries | Under 100 ms |
| Batch operations | Document the expected duration |

Throughput:

- Applications MUST document the expected throughput.
- Load testing results SHOULD be provided.
- Performance degradation under load MUST be documented.

Caching:

- Applications SHOULD implement caching where appropriate.
- The cache invalidation strategy MUST be documented.
- The cache hit rate SHOULD be monitored.

#### Cost [COMP030-cost-optimization]

Resource efficiency:

- Applications SHOULD use resources efficiently. Unnecessary resource requests waste money.
- Autoscaling SHOULD be used to match demand.
- Idle resources SHOULD be scaled to the minimum during off-hours.

Cost visibility:

- Resource requirements SHOULD be documented.
- The cost per transaction or per user SHOULD be estimated.
- Optimization opportunities SHOULD be identified.

## Proposed additions

> **Status: proposal.** Nothing in this section is a requirement yet. These are suggestions for discussion between Dimpact, SSC Hosting and the suppliers. A proposal that is accepted gets a `COMP` ID and moves to [Requirements](#requirements).

The requirements above mostly describe how one application should behave. These proposals are about delivery and integration: what is needed when applications from several suppliers are combined into one platform, and run for many municipalities on shared infrastructure.

| Theme | Aim | Proposals |
| --- | --- | --- |
| [A. A delivery that can be verified](#a-a-delivery-that-can-be-verified) | What is delivered is exactly what runs, and its origin can be proven. | PROP-01 to PROP-07 |
| [B. A good neighbor in a shared cluster](#b-a-good-neighbor-in-a-shared-cluster) | One component cannot disturb another component, or the cluster. | PROP-08 to PROP-16 |
| [C. Life cycle of the whole platform](#c-life-cycle-of-the-whole-platform) | The platform as a whole survives stops, upgrades and restores. | PROP-17 to PROP-23 |
| [D. Problems that cross supplier boundaries](#d-problems-that-cross-supplier-boundaries) | A fault can be followed through components of different suppliers. | PROP-24 to PROP-28 |
| [E. Compliance that can be checked](#e-compliance-that-can-be-checked) | Compliance is measured, not discussed. | PROP-29 to PROP-33 |

Suggested to discuss first: PROP-02, PROP-08, PROP-17, PROP-19 and PROP-29. These five have the widest effect, because each one concerns every component in every environment.

The "Level" column gives the proposed level.

### A. A delivery that can be verified

| ID | Level | Proposal | Why |
| --- | --- | --- | --- |
| PROP-01 | MUST | A released image tag is never overwritten, and `latest` is not used. The release notes give the digest of every image, and the Helm chart accepts a digest in place of a tag. | SSC Hosting copies each image to its own registry and rolls a release out over several weeks. If a tag changes in between, acceptance and production run different code under the same version number. |
| PROP-02 | MUST | Every image that a chart can start is set through values: containers, init containers, sidecars, jobs, hooks, tests and sub-charts. One global value overrides the registry for all of them. Each chart release lists all its images. | SSC Hosting runs images from its own registry ([COMP003.3-container-registry]). One hard-coded image in a hook breaks the deployment, or quietly keeps the platform dependent on a public registry with its rate limits and outages. |
| PROP-03 | MUST NOT | A container does not download code, plugins or other assets from the internet when it starts or while it runs. | What was scanned must be what runs. Runtime downloads also need outbound internet access, which a shared platform wants to keep closed. |
| PROP-04 | MUST | Images are built on a base image and a language runtime that still get security updates. Every supported release is rebuilt at least once a month, also when the application code has not changed. | Most findings of an image scanner are in the base layers. The 7-day limit in [COMP023-security] cannot be met without a routine rebuild. |
| PROP-05 | SHOULD | Images and charts are signed, for example with [Sigstore](https://docs.sigstore.dev/). The SBOM ([COMP024-sbom]) and the build provenance ([SLSA](https://slsa.dev/)) are attached to the image as attestations. | With several suppliers, the origin of an artifact must be proven automatically before the cluster admits it. An SBOM that travels with the image cannot go missing or out of date. |
| PROP-06 | MUST | A chart passes `helm lint` and renders with its default values. It contains a `values.schema.json`, declares the supported Kubernetes versions in `kubeVersion`, and uses no API that is deprecated in those versions. Each release includes tested example values for a minimal and for a high-availability installation. | Errors show up in the supplier's pipeline, not during a deployment for a municipality. Kubernetes is updated every month, so API removals arrive quickly. |
| PROP-07 | SHOULD | Images are published for both `linux/amd64` and `linux/arm64`. | SSC Hosting can then choose cheaper ARM nodes ([COMP030-cost-optimization]), and developers can run the platform on ARM laptops. |

### B. A good neighbor in a shared cluster

| ID | Level | Proposal | Why |
| --- | --- | --- | --- |
| PROP-08 | MUST NOT | A component chart does not create cluster-wide resources: CRDs, ClusterRoles and their bindings, admission webhooks, storage, ingress or priority classes, namespaces. An operator or CRD that cannot be avoided is delivered separately, with the versions it supports, so that SSC Hosting can install and upgrade it on its own. | Cluster-wide resources are shared by all tenants. Two suppliers that ship different versions of the same CRD or operator overwrite each other, and Helm does not upgrade CRDs. This fills the gap left by the withdrawn COMP007.2-operators. |
| PROP-09 | MUST | All workloads run under the Kubernetes [Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/) `restricted`. An exception needs a waiver per container. | Turns the container rules of [COMP023-security] into one check that the cluster itself enforces on the namespace. |
| PROP-10 | MUST | Each workload has its own ServiceAccount. The API token is mounted only if the workload calls the Kubernetes API. Any RBAC rights it needs are limited to the namespace and listed in the documentation. | All of PodiumD runs in one namespace. A pod of one supplier must not be able to read the secrets of another supplier's component. |
| PROP-11 | MUST | Each component declares its connections in a machine-readable file: inbound and outbound, with peer, port and protocol, and the host names of external systems. The chart includes NetworkPolicies that match, and that work when all other traffic is denied by default. | [COMP023-security] requires network policies, but only the supplier knows the traffic flows. SSC Hosting needs the same list for firewall and egress rules. |
| PROP-12 | MUST | Every workload, including jobs and hooks, has values for `nodeSelector`, `tolerations`, `affinity`, `topologySpreadConstraints` and `priorityClassName`. | The platform has separate node groups and several availability zones. Without these settings SSC Hosting cannot place workloads, and [COMP014.3-redundancy] cannot be met. |
| PROP-13 | MUST | Every container has CPU and memory requests and a memory limit, including init containers, sidecars, jobs and hooks. The chart installs in a namespace that has a ResourceQuota and a LimitRange. | [COMP020-resource-recommendations] covers the main workload. A single container without requests is invisible to the scheduler and can starve its neighbors. |
| PROP-14 | MUST | All resources have the [recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) `app.kubernetes.io/*` (name, instance, version, component, part-of, managed-by) and `helm.sh/chart`. Resource names are derived from the release name. The namespace and the cluster domain are never hard-coded. | With components of several suppliers in one namespace, labels are the only reliable way to see which pod belongs to which component and version, and to select it for monitoring, network policies and cost allocation. |
| PROP-15 | MUST | A component can use the shared services of the platform instead of its own copy: PostgreSQL, Redis, object storage, search, message broker, identity provider and mail. Every bundled dependency can be switched off in values. The component declares what it needs from each service: version range, extensions, size and number of connections. The database account used at run time needs no superuser rights. | Extends [COMP007.5-database] to all backing services. Otherwise the platform ends up with several Redis and search installations, each with its own life cycle. A managed PostgreSQL server only allows listed extensions, and all applications share its connection limit. |
| PROP-16 | MUST | An application accepts an additional CA bundle through configuration. It SHOULD also honor `HTTP_PROXY`, `HTTPS_PROXY` and `NO_PROXY`. | Municipal back-office systems often use certificates from a private CA, such as the PKIoverheid private root. Without this setting, each environment needs its own image. |

### C. Life cycle of the whole platform

| ID | Level | Proposal | Why |
| --- | --- | --- | --- |
| PROP-17 | MUST | An environment recovers without manual action after everything in it was stopped at the same time, whatever the order in which things start again. A component waits for a missing dependency and retries; it does not fail for good. Stateful components come back without repair work. | Development, test and acceptance environments are switched off every evening and weekend. [COMP006.1-stop-start-safe] covers the loss of one container or node, not of everything at once. |
| PROP-18 | MUST | A container handles SIGTERM: it stops accepting new work, finishes the requests in progress and exits within its `terminationGracePeriodSeconds`. | Monthly node upgrades and autoscaling stop pods all the time. Zero-downtime upgrades ([COMP006.2-upgrade-time-limit]) are impossible if a stopping pod drops its requests. |
| PROP-19 | MUST | Database migrations run as one explicit step with a clear exit status, such as a job or hook, and never from every replica at startup. They can be run again safely after an interruption. The release notes state how long the migration takes on production-sized data, and whether the release can still be rolled back after it. Schema changes SHOULD follow the expand-and-contract pattern, so that the previous application version works with the new schema. | Makes [COMP006.2-upgrade-time-limit], [COMP006.3-rollbacks] and [COMP010-sql-migration-scripts] achievable. Zero downtime and rollback only work if the old code can run against the new schema. |
| PROP-20 | MUST | Each release states from which versions a direct upgrade is supported. An upgrade from the previous minor release is always supported. A values key, environment variable or API is announced as deprecated at least one minor release before it is removed or renamed. Each release includes a machine-readable list of the values that changed. | Municipalities run different versions at the same time. The values for many environments and several suppliers are maintained in one place, where a silent rename becomes a failed deployment. |
| PROP-21 | MUST | Helm hooks can be run again and have a delete policy. Jobs set `backoffLimit`, `activeDeadlineSeconds` and `ttlSecondsAfterFinished`. CronJobs set `concurrencyPolicy` and can be suspended in values. An installation or upgrade finishes within a stated time when Helm waits for it and rolls back on failure. | A failed hook leaves a release stuck, and a leftover job blocks the next deployment. A stated time gives the pipeline a limit to test against. |
| PROP-22 | MUST | The configuration that connects components (endpoints, credentials, client registrations) is applied automatically from values and secrets, and can be applied again without side effects. No step needs an administration screen. Functional setup SHOULD be exportable and importable. | [COMP008.1-100percent-automated] stops at running containers. An environment that then needs hours of manual configuration is not automated, and cannot be rebuilt quickly after a disaster ([COMP021-disaster-recovery]). |
| PROP-23 | MUST | Each component lists where its state lives: database, object storage, volumes, search index, cache. It states which of these is the source of truth, which can be rebuilt, and how. The restore procedure gives the order relative to other components. | [COMP021-disaster-recovery] asks for RTO and RPO per application. Components refer to each other's data, so a restore has to be consistent across suppliers. |

### D. Problems that cross supplier boundaries

| ID | Level | Proposal | Why |
| --- | --- | --- | --- |
| PROP-24 | MUST | An application passes the [W3C Trace Context](https://www.w3.org/TR/trace-context/) headers on from incoming to outgoing calls, and writes the trace ID in every log line. It SHOULD export traces with [OpenTelemetry](https://opentelemetry.io/docs/). | One action of a user passes through components of several suppliers. Without a shared ID, every incident starts with a discussion about whose component failed. |
| PROP-25 | MUST | JSON logs have an agreed minimum set of fields: timestamp, level, message, component, version and trace ID. Logs contain no personal data, such as a BSN, names or document content, and no secrets or tokens. | All components log to one shared platform. Operators who read those logs have no need to see data about citizens. |
| PROP-26 | SHOULD | The chart can install a ServiceMonitor or PodMonitor, alert rules and dashboards for the component. Values switch them on. | The supplier knows which signals matter. [COMP017-golden-signals] and [COMP018-alerting] say what to measure; without this, each environment is set up by hand. |
| PROP-27 | MUST | The chart includes [`helm test`](https://helm.sh/docs/topics/chart_tests/) checks that show that the component is up, reaches its dependencies and answers on its API. | Gives the deployment pipeline an objective "deployed and working" result for each component. |
| PROP-28 | SHOULD | Where a component uses the API of another supplier's component, the consumer publishes contract tests, for example with [Pact](https://docs.pact.io/), and the provider runs them in its release pipeline. | A break between two suppliers is found before the platform release is put together, by the party that can fix it. |

### E. Compliance that can be checked

| ID | Level | Proposal | Why |
| --- | --- | --- | --- |
| PROP-29 | Process | Every requirement in this document gets a verification method: automated check, review, or declaration by the supplier. The automated checks are published as a pipeline that suppliers can run before they deliver. | Some fifty requirements, for every component and every release, cannot be reviewed by hand. Suppliers see the result before SSC Hosting does. |
| PROP-30 | MUST | Each release includes a machine-readable compliance declaration. For every requirement ID it says whether the component complies, the requirement does not apply, or a waiver exists, with a link to the evidence. | Replaces separate compliance reviews with a record that travels with the release. |
| PROP-31 | Process | Waivers are recorded in this repository, with requirement ID, component, reason, owner and end date. | This document mentions waivers in several places, but not who grants them, where they are kept or when they end. |
| PROP-32 | MUST | Each supplier publishes which versions get security fixes (at least the current and the previous minor release), a security contact with a disclosure procedure, and end-of-life dates at least six months ahead. | A platform release pins versions of all suppliers. Each pinned version has to stay supported for as long as it runs in production. |
| PROP-33 | SHOULD | APIs follow the Dutch government [REST API Design Rules](https://gitdocumentatie.logius.nl/publicatie/api/adr/), in place of the general list in [COMP025-api-practices]. Each component repository contains a [`publiccode.yml`](https://yml.publiccode.tools/). | The design rules are a mandatory (*pas toe of leg uit*) standard for Dutch government bodies, municipalities included. `publiccode.yml` is the metadata standard that lets catalogs of public-sector software index a component automatically. |

## References

- [Architecting Applications for Kubernetes (DigitalOcean)](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)
- [15 Principles for Designing and Deploying Scalable Applications on Kubernetes (Elastisys, archived copy)](https://web.archive.org/web/20251005184420/https://elastisys.com/designing-and-deploying-scalable-applications-on-kubernetes/)
- [Golden signals in Kubernetes (Sysdig)](https://sysdig.com/blog/golden-signals-kubernetes/)
- [Microsoft Well-Architected Framework for Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-kubernetes-service)
- [Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Azure Best Practices for AKS](https://learn.microsoft.com/en-us/azure/aks/best-practices)

## Version history

| Version | Date | Author | Comments by | Comments |
| --- | --- | --- | --- | --- |
| 0.1 | 2024-02-20 | Jim Leitch | Sander vd B, Andrew M | |
| 0.8 | 2024-03-04 | Jim Leitch | Stephan Z, Jesse H, Mahmut C, Petra C | |
| 1.0 | 2024-04-22 | Jim Leitch | SSC, Maykin, Dimpact | |
| 1.1 | 2024-05-06 | Jim Leitch | | |
| 1.2 | 2024-05-29 | Jim Leitch | Dimpact, ICATT | Updated "k8s operators" |
| 2024Q3 | 2024-08-27 | Jim Leitch | SSC | Converted to Markdown |
| 2025Q4 | 2025-12-17 | Jim Leitch | SSC, ICATT, Dimpact, Capgemini | |
| 2026Q1 (draft) | 2026-11-16 | Jim Leitch | | Simplified and restructured; no requirements added or removed. Proposals PROP-01 to PROP-33 added for discussion |

[COMP001-naming]: #naming-comp001-naming
[COMP002-versioning]: #versioning-comp002-versioning
[COMP002.1-version-api]: #version-endpoint-comp0021-version-api
[COMP003.1-app-as-containers]: #containers-comp003
[COMP003.3-container-registry]: #containers-comp003
[COMP003.3-semantic-versioning]: #containers-comp003
[COMP004-timezone]: #time-zone-comp004-timezone
[COMP005-software-supply-chain]: #software-supply-chain-comp005-software-supply-chain
[COMP006.1-stop-start-safe]: #stops-starts-and-scaling-comp0061-stop-start-safe
[COMP006.2-upgrade-time-limit]: #upgrades-comp0062-upgrade-time-limit
[COMP006.3-rollbacks]: #rollbacks-comp0063-rollbacks
[COMP007.1-helm]: #helm-charts-comp007
[COMP007.3-dependancy-apps]: #helm-charts-comp007
[COMP007.4-storage]: #helm-charts-comp007
[COMP007.5-ingress]: #helm-charts-comp007
[COMP007.5-database]: #helm-charts-comp007
[COMP008.1-100percent-automated]: #automated-deployment-comp0081-100percent-automated
[COMP009-internal-communication]: #communication-between-components-comp009-internal-communication
[COMP010-sql-migration-scripts]: #database-migrations-comp010-sql-migration-scripts
[COMP011-no-local-storage]: #state-and-storage-comp011
[COMP011-permanent-data]: #state-and-storage-comp011
[COMP012-secrets]: #configuration-and-secrets-comp012-secrets
[COMP013-ssl]: #tls-termination-comp013-ssl
[COMP014-healthchecks]: #health-checks-comp014-healthchecks
[COMP014.1-health-states]: #health-states-comp0141-health-states
[COMP014.2-probe-config]: #probes-comp0142-probe-config
[COMP014.3-redundancy]: #redundancy-and-fault-tolerance-comp0143-redundancy
[COMP014.4-pdb]: #pod-disruption-budgets-comp0144-pdb
[COMP015.1-logging-to-stdout]: #logging-comp015-comp016
[COMP016.2-utc-logs]: #logging-comp015-comp016
[COMP017-golden-signals]: #metrics-comp017-golden-signals
[COMP017.1-latency-metrics]: #metrics-comp017-golden-signals
[COMP017.2-traffic-metrics]: #metrics-comp017-golden-signals
[COMP017.3-error-metrics]: #metrics-comp017-golden-signals
[COMP017.4-saturation-metrics]: #metrics-comp017-golden-signals
[COMP018-alerting]: #alerting-comp018-alerting
[COMP018.1-slo-alerting]: #alerting-comp018-alerting
[COMP019-slo]: #service-level-objectives-comp019-slo
[COMP020-resource-recommendations]: #resources-comp020-resource-recommendations
[COMP021-disaster-recovery]: #disaster-recovery-comp021-disaster-recovery
[COMP022-release-notes]: #release-notes-comp022-release-notes
[COMP023-security]: #security-comp023-security
[COMP024-sbom]: #software-bill-of-materials-comp024-sbom
[COMP025-api-practices]: #apis-comp025-api-practices
[COMP028-compliance]: #compliance-and-audit-comp028-compliance
[COMP029-performance]: #performance-comp029-performance
[COMP030-cost-optimization]: #cost-comp030-cost-optimization
