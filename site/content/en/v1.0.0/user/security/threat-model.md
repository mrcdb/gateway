---
title: "Threat Model"
---

# Envoy Gateway Threat Model and End User Recommendations

## About

This work was performed by [ControlPlane](https://control-plane.io/) and commissioned by the [Linux Foundation](https://www.linuxfoundation.org/). ControlPlane is a global cloud native and open source cybersecurity consultancy, trusted as the partner of choice in securing: multinational banks; major public clouds; international financial institutions; critical national infrastructure programs; multinational oil and gas companies, healthcare and insurance providers; and global media firms.

## Threat Modelling Team

James Callaghan, Torin van den Bulk, Eduardo Olarte

## Reviewers

Arko Dasgupta, Matt Turner, Zack Butcher, Marco De Benedectis

## Introduction

As we embrace the proliferation of microservice-based architectures in the cloud-native landscape, simplicity in setup and configuration becomes paramount as DevOps teams face the challenge of  choosing between numerous similar technologies. One such choice which every team deploying to Kubernetes faces is what to use as an ingress controller. With a plethora of options available, and the existence of vendor-specific annotations leading to small inconsistencies between implementations,  the [Gateway API](https://gateway-api.sigs.k8s.io/) project was introduced by the SIG-NETWORK community, with the goal of eventually replacing the Ingress resource.

Envoy Gateway is configured by Gateway API resources, and serves as an intuitive and feature-rich wrapper over the widely acclaimed Envoy Proxy. With a convenient setup based on Kubernetes (K8s) manifests, Envoy Gateway streamlines the management of Envoy Proxy instances in an edge-proxy setting, reducing the operational overhead of managing low-level Envoy configurations. Envoy Gateway benefits cloud-native DevOps teams through its role-oriented configuration, providing granular control based on Role-Based Access Control (RBAC) principles. These features form the basis of our exploration into Envoy Gateway and the rich feature set it brings to the table.

In this threat model, we aim to provide an analysis of Envoy Gateway's design components and their capabilities (at version 1.0) through a threat-driven approach. It should be noted that this does not constitute a security audit of the Envoy Gateway project, but instead focuses on different possible deployment topologies for Envoy Gateway with the goal of deriving recommendations and best practice guidance for end users.

The Envoy Gateway project recommends a [multi-tenancy model](https://gateway.envoyproxy.io/latest/user/operations/deployment-mode/#multi-tenancy) whereby each tenant deploys their own Envoy Gateway controller in a namespace which they own. We will also explore the implications and risks associated with multiple tenants using a shared controller.

### Scope

The primary focus of this threat model is to identify and assess security risks associated with deploying and operating Envoy Gateway within a multi-tenant Kubernetes (K8s) cluster. This model aims to provide a comprehensive understanding of the system, its transmission points, and potential vulnerabilities to enumerated threats.

### In Scope

**Envoy Gateway**: As the primary focus of this threat model, all aspects of Envoy Gateway, including its configuration, deployment, and operation will be analysed. This includes how the gateway manages TLS certificates, authentication, service-to-service traffic routing, and more.

**Kubernetes Cluster**: Configuration and operation of the underlying Kubernetes cluster, including how it manages network policies, access control, and resource isolation for different namespaces/tenants in relation to Envoy will be considered.

**Tenant Workloads**: Tenant workloads (and the pods they run on) will be considered, focusing on how they interact with the Envoy Gateway and potential vulnerabilities that could be exploited.

#### Out of Scope

This threat model will not consider security risks associated with the underlying infrastructure (e.g., EC2 compute instances and S3 buckets) or non-Envoy related components within the Kubernetes Cluster. It will focus solely on the Envoy Gateway and its interaction with the Kubernetes cluster and tenant workloads.

Implementation of Envoy Gateway as an egress traffic controller is out of scope for this threat model and will not be considered in the report's findings.

### Related Resources

[Introducing Envoy Gateway](https://blog.envoyproxy.io/introducing-envoy-gateway-ad385cc59532)

[Envoy Proxy Threat Model](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/security/threat_model#threat-model)

[Configuring Envoy as an Edge Proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge#best-practices-edge)

[Envoy Gateway Deployment Mode](https://gateway.envoyproxy.io/latest/user/operations/deployment-mode/)

[Kubernetes Gateway API Security Model](https://gateway-api.sigs.k8s.io/concepts/security-model/)

## Architecture Overview

### Summary

To provide an in-depth look into both the system design and end-user deployment of Envoy Gateway, we will be focusing on the [Deployment Architecture Diagram](#deployment-architecture-diagram) below.

The Deployment Architecture Diagram provides a high-level model of an end-user deployment of Envoy Gateway. For simplicity, we will look at different deployment topologies on a single multi-tenant Kubernetes cluster. Envoy Gateway operates as an edge proxy within this environment, handling the traffic flow between external interfaces and services within the cluster. The example will use two Envoy Gateway controllers - one dedicated controller for a single tenant, and one shared controller for two other tenants. Each Envoy Gateway controller will accept a single GatewayClass resource.

### Deployment Architecture Diagram

As Envoy Gateway implements the [Kubernetes GatewayAPI](https://gateway-api.sigs.k8s.io/concepts/api-overview/), this threat model will focus on the key objects in the Gateway API resource model:

1. **GatewayClass:** defines a set of gateways with a commonconfiguration and behaviour. It is a cluster scoped resource.

2. **Gateway:** requests a point where traffic can be translated to Services within the cluster.

3. **Routes:** describe how traffic coming via the Gateway maps to theServices.

At the time of writing, Envoy Gateway only supports a Kubernetes provider. As such, we will consider a reference architecture where multiple teams are working on the same Kubernetes cluster within different namespaces (Tenant A, B, & C). We will assume that some teams have similar security and performance needs, and a decision has been made to use a shared Gateway. However, we will also consider the case that some teams require dedicated Gateways, perhaps for compliance reasons or requirements driven by an internal threat model.

We will consider the following organisational roles, as per the [Gateway API security model](https://gateway-api.sigs.k8s.io/concepts/security-model/):

1. **Infrastructure provider**: The infrastructure provider (infra) is responsible for the overall environment that the cluster(s) are operating in. Examples include: the cloud provider (AWS, Azure, GCP, ...) or the PaaS provider in a company.

2. **Cluster operator**: The cluster operator (ops) is responsible for administration of entire clusters. They manage policies, network access, application permissions.

3. **Application developer**: The application developer (dev) is responsible for defining their application configuration (e.g. timeouts, request matching/filter) and Service composition (e.g. path routing to backends).

4. **Application admin**: The application admin has administrative access to some namespaces within a cluster, but not the cluster as a whole.

Our threat model will be based on the high-level setup shown below, where Envoy is used in an edge-proxy scenario:

![Architecture](/img/architecture_threat_model.png)

The following use cases will be considered, in line with the [Envoy Gateway User Guides](https://gateway.envoyproxy.io/latest/user/):

1. Routing and controlling traffic, including:
    a. HTTP \
    b. TCP \
    c. UDP \
    d. gRPC \
    e.TLS passthrough
2. TLS termination
3. Request Authentication
4. Rate Limiting

## Key Assumptions

This section outlines the foundational premises that shape our analysis and recommendations for the deployment and management of Envoy Gateway within an organisation. The key assumptions are as follows:

**1. Kubernetes Provider**: For the purposes of this analysis, we assume that a K8s provider will be used to host the cluster.

**2. Multi-tenant cluster**: In order to produce a broad set of recommendations, it is assumed that within the single cluster, there is:

- A dedicated cluster operation (ops) team responsible for maintaining the core cluster infrastructure.

- Multiple application teams who wish to define their own Gateway resources, which will route traffic to their respective applications. 

**3. Soft multi-tenancy model**: It is assumed that co-tenants will have some level of trust between themselves, and will not act in an overtly hostile manner to each other.

**4. Ingress Control**: It's assumed that Envoy Gateway is the only ingress controller in the K8s cluster as multiple controllers can lead to complex routing challenges and introduce out-of-scope security vulnerabilities.

**5. Container Security**: This threat model focuses on evaluating the security of the Envoy Gateway and Envoy Proxy images. All other container images running in tenant clusters, not associated with the edge proxy deployment, are assumed to be secure and obtained from trusted registries such as Docker Hub or Google Container Registry (GCR).

**6. Cloud Provider Security**: It is assumed that the K8s cluster is running on secure cloud infrastructure provided by a trusted Cloud Service Provider (CSP) such as AWS, GCP, or Azure Cloud.

## Data

### Data Dictionary

Ultimately, the data of interest in a threat model is the business data processed by the system in question. However, in the case of this threat model, we are looking at a generic deployment architecture involving Envoy Gateway in order to draw out a set of generalised threats which can be considered by teams looking to adopt an implementation of Gateway API. As such, we do not know the business impacts of a compromise of confidentiality, integrity or availability  that would typically be captured in a data impact assessment. Instead, will we base our threat assessment on high-level groupings of data structures used in the configuration and operation of the general use cases considered (e.g. HTTP routing, TLS termination, request authentication etc.). We will then assign a confidentiality, integrity and availability impact based on a worst-case scenario of how each compromise could potentially affect business data processed by the generic deployment.

| Data Name / Type | Notes | Confidentiality | Integrity | Availability |
| ------------ | ------------ | ------------ |--------------- | ------------ |
| Static Configuration Data | Static configuration data is used to configure Envoy Gateway at startup. This data structure allows for a Provider to be set, which Envoy Gateway calls to establish its runtime configuration, resolve services and persist data. Unauthorised modification of static configuration data could enable the Envoy Gateway admin interface to be configured, logging parameters to be modified, global rate limiting configuration to be misconfigured, or malicious extensions registered for the Envoy Gateway Control Plane.  A compromise of confidentiality could potentially give an attacker some useful reconnaissance information. A compromise of the availability of this information at startup time would result in Envoy Gateway starting with default parameters. | Medium | High | Low |
| Dynamic Configuration Data | Dynamic configuration data represents the desired state of the Data Plane, and is defined through Envoy Gateway and Gateway API Kubernetes resources. Unauthorised modification of this data could lead to vulnerabilities in an organisation’s Data Plane infrastructure via misconfiguration of an EnvoyProxy custom resource. Misconfiguration of Gateway API objects such as HTTPRoutes or TLSRoutes could result in traffic being directed to incorrect backends. A compromise of confidentiality could potentially give an attacker some useful reconnaissance information. A compromise of the availability of this information could result in tenant application traffic not being routable until the configuration is recovered and reapplied. | Medium | High | Medium |
| TLS Private Keys | TLS Private Keys, typically in PEM format, are used to initiate secure connections and encrypt communications. In the context of this threat model, private keys will be associated with the server side of an inbound TLS connection being terminated at a secure gateway configured through Envoy Gateway. Unauthorised exposure could lead to security threats such as person-in-the-middle attacks, whereby the confidentiality or integrity of business data could be compromised. A compromise of integrity may lead to similar consequences if an attacker could insert their own key material. An availability compromise could lead to tenant services being unavailable until new key material is generated and an appropriate CSR submitted. | High | High | Medium |
| TLS Certificates | X.509 certificates represent the binding of a public key (associated with the private key described above) to an identity in a TLS handshake. If an attacker could compromise the integrity of a certificate, they may be able to bind the identity of a TLS termination point to a key pair under their control, enabling person-in-the middle attacks. An availability compromise could lead to tenant services being unavailable until new key material is generated and an appropriate CSR submitted. | Low | High | Medium |
| JWKs | JWK (JSON Web Key) containing a public key used to validate JWTs for the client authentication use case considered in this threat model. If an attacker could compromise the integrity of a JWK or  JSON web key set (JWKS), they could potentially authenticate to a service maliciously. Unavailability of an endpoint exposing JWKs could lead to client requests which require authentication being denied. | Low | High | Medium |
| JWTs | JWTs, formatted as compact, URL-safe JSON data structures, are utilised for the client authentication use case considered in this threat model. Maintaining their confidentiality and integrity is vital to prevent unauthorised access and ensure correct user identification. | High | High | Low |
| OIDC credentials | In OIDC authentication scenarios, the application credentials are represented by a client ID and a client secret. A compromise of its confidentiality or integrity could allow malicious actors to impersonate the application, potentially being able to access resources on behalf of the application and request ID tokens on behalf of users. Unavailability of this data would produce a rejection of the requests coming from legitimate users. | High | High | Medium |
| Basic authentiation password hashes | In basic authentication scenarios, passwords are stored as Kubernetes secrets in [htpasswd](https://httpd.apache.org/docs/current/programs/htpasswd.html) format, where each entry is formed by the username and the hashed password. A compromise of these credentials' confidentiality and integrity could lead to unauthorised access to the application. Unavailability of these credentials will cause login failures from the application users. | High | High | Medium |

### CIA Impact Assessment

| Priority | Description |
| --- | --- |
| **Confidentiality** | |
| High | Compromise of sensitive client data |
| Medium | Information leaked which could be useful for attacker reconnaissance |
| Low | Non-sensitive information leakage |
| **Integrity** | |
| High | Compromise of source code repositories and gateway deployments |
| Medium | Traffic routing fails due to misconfiguration / invalid configuration |
| Low | Non-critical operation is blocked due to misconfiguration / invalid configuration |
| **Availability** |  |
| High | Large scale DoS |
| Medium | Tenant application is blocked for a significant period |
| Low | Tenant application is blocked for a short period |

### Data Flow Diagrams

The Data Flow Diagrams (DFDs) below describe the flow of data between the various processes, entities and data stores in a system, as well as the trust boundaries between different user roles and network interfaces. The DFDs are drawn at two different levels, starting at L0 (high-level system view) and increasing in granularity (to L1).

### DFD L0

![DFD L0](/img/DFDL0.png)

### DFD L1

![DFD L1](/img/DFDL1.png)

## Key Threats and Recommendations

The scope of this threat model led to us categorising threats into priorities of High, Medium or Low; notably in a production implementation some of the threats' prioritisation may be upgraded or downgraded depending on the business context and data classification.

### Risk vs. Threat

For every finding, the risk and threat are stated. Risk defines the potential for negative outcome while threat defines the event that causes the negative outcome.

### Threat Categorization

Throughout this threat model, we categorised threats into different areas based on their origin and the segment of the system that they impact. Here's an overview of each category:

**Container Security (CS)**: These threats are general to containerised applications. Therefore, they are not associated with Envoy Gateway or the Gateway API and could occur in most containerised workloads. They can originate from misconfigurations or vulnerabilities in the orchestrator or the container.

**Gateway API (GW)**: These are threats related to the Gateway API that could affect any of its implementations. Malicious actors could benefit from misconfigurations or excessive permissions on the Gateway API resources (e.g. xRoutes or Gateways) to compromise the confidentiality, integrity, or availability of the application.

**Envoy Gateway (EG)**: These threats are associated with specific configurations or features from Envoy Gateway or Envoy Proxy. If not set properly, these features could be leveraged to gain unauthorised access to protected resources.

### Threat Actors

In order to provide a realistic set of threats that is applicable to most organisations, we de-scoped the most advanced and hard to mitigate threat actors as described below:

#### In Scope Threat Actors

When considering internal threat actors, we chose to follow the [security model](https://gateway-api.sigs.k8s.io/concepts/security-model/) of the Kubernetes Gateway API.

##### Internal Attacker

- Cluster Operator: The cluster operator (ops) is responsible for administration of entire clusters. They manage policies, network access, application permissions.

- Application Developer: The application developer (dev) is responsible for defining their application configuration (e.g. timeouts, request matching/filter) and Service composition (e.g. path routing to backends).

- Application Administrator: The application admin has administrative access to some namespaces within a cluster, but not the cluster as a whole.

##### External Attacker

- Vandal: Script kiddie, trespasser

- Motivated Individual: Political activist, thief, terrorist

- Organised Crime: Syndicates, state-affiliated groups

#### Out of Scope Threat Actors

##### External Actors

- Infrastructure Provider: The infrastructure provider (infra) is responsible for the overall environment that the cluster(s) are operating in. Examples include: the cloud provider, or the PaaS provider in a company.

- Cloud Service Insider: Employee, external contractor, temporary worker

- Foreign Intelligence Services (FIS): Nation states

## High Priority Findings

### EGTM-001 Usage of self-signed certificates

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-001|EGTM-GW-001|Gateway API|High|

 **Risk**: Self-signed certificates (which do not comply with PKI best practices) could lead to unauthorised access to the private key associated with the certificate used for inbound TLS termination at Envoy Proxy, compromising the confidentiality and integrity of proxied traffic.

 **Threat**: Compromise of the private key associated with the certificate used for inbound TLS terminating at Envoy Proxy.

 **Recommendation**: The Envoy Gateway quickstart guide demonstrates how to set up a Secure Gateway using an example where a self-signed root certificate is created using openssl. As stated in the Envoy Gateway documentation, this is not a suitable configuration for Production usage. It is recommended that PKI best practices are followed, whereby certificates are signed by an Intermediary CA which sits underneath an organisational \'offline\' Root CA.

 PKI best practices should also apply to the management of client certificates when using mTLS. The Envoy Gateway [mTLS](https://gateway.envoyproxy.io/latest/user/security/mutual-tls/) guide shows how to set up client certificates using self-signed certificates. In the same way as gateway certificates and, as mentioned in the documentation, this configuration should not be used in production environments.

### EGTM-002 Private keys are stored as Kubernetes secrets

|**ID**|**UID**|**Category**|**Priority**|
|--------------|--------------|------------------------|-----------------|
|EGTM-002|EGTM-CS-001|Container Security|High|

 **Risk**: There is a risk that a threat actor could compromise the Kubernetes secret containing the Envoy private key, allowing the attacker to decrypt Envoy Proxy traffic, compromising the confidentiality of proxied traffic.

 **Threat**: Kubernetes secret containing the Envoy private key is compromised and used to decrypt proxied traffic.

 **Recommendation**: Certificate management best practices mandate short-lived key material where practical, meaning that a mechanism for rotation of private keys and certificates is required, along with a way for certificates to be mounted into Envoy containers. If Kubernetes secrets are used, when a certificate expires, the associated secret must be updated, and Envoy containers must be redeployed. Instead of a manual configuration, it is recommended that [cert-manager](https://github.com/cert-manager/cert-manager) is used.

### EGTM-004 Usage of ClusterRoles with wide permissions

|**ID**|**UID**|**Category**|**Priority**|
|--------------|--------------|------------------------|-----------------|
|EGTM-004|EGTM-K8-002|Container Security|High|

 **Risk**: There is a risk that a threat actor could abuse misconfigured RBAC to access the Envoy Gateway ClusterRole (envoy-gateway-role) and use it to expose all secrets across the cluster, thus compromising the confidentiality and integrity of tenant data.

 **Threat**: Compromised Envoy Gateway or misconfigured ClusterRoleBinding (envoy-gateway-rolebinding) to Envoy Gateway ClusterRole (envoy-gateway-role), provides access to resources and secrets in different namespaces.

 **Recommendation**: Users should be aware that Envoy Gateway uses a ClusterRole (envoy-gateway-role) when deployed via the Helm chart, to allow management of Envoy Proxies across different namespaces. This ClusterRole is powerful and includes the ability to read secrets in namespaces which may not be within the purview of Envoy Gateway.

 Kubernetes best-practices involve restriction of ClusterRoleBindings, with the use of RoleBindings where possible to limit access per namespace by specifying the namespace in metadata. Namespace isolation reduces the impact of compromise from cluster-scoped roles. Ideally, fine-grained K8s roles should be created per the principle of least privilege to ensure they have the minimum access necessary for role functions.

 The pull request \#[1656](https://github.com/envoyproxy/gateway/pull/1656) introduced the use of Roles and RoleBindings in [namespaced mode](https://gateway.envoyproxy.io/latest/api/extension_types/#kuberneteswatchmode). This feature can be leveraged to reduce the amount of permissions required by the Envoy Gateway.

### EGTM-007 Misconfiguration of Envoy Gateway dynamic config

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-007|EGTM-EG-002|Envoy Gateway|High|

 **Risk**: There is a risk that a threat actor could exploit misconfigured Kubernetes RBAC to create or modify Gateway API resources with no business need, potentially leading to the compromise of the confidentiality, integrity, and availability of resources and traffic within the cluster.

 **Threat**: Unauthorised creation or misconfiguration of Gateway API resources by a threat actor with cluster-scoped access.

 **Recommendation**: Configure the apiGroup and resource fields in RBAC policies to restrict access to [Gateway](https://gateway-api.sigs.k8s.io/) and [GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/) resources. Enable namespace isolation by using the namespace field, preventing unauthorised access to gateways in other namespaces.

### EGTM-009 Co-tenant misconfigures resource across namespaces

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-009|EGTM-GW-002|Gateway API|High|

 **Risk**: There is a risk that a co-tenant misconfigures Gateway or Route resources, compromising the confidentiality, integrity, and availability of routed traffic through Envoy Gateway.

 **Threat**: Malicious or accidental co-tenant misconfiguration of Gateways and Routes associated with other application teams.

 **Recommendation**: Dedicated Envoy Gateways should be provided to each tenant within their respective namespace. A one-to-one relationship should be established between GatewayClass and Gateway resources, meaning that each tenant namespace should have their own GatewayClass watched by a unique Envoy Gateway Controller as defined here in the [Deployment Mode](https://gateway.envoyproxy.io/latest/user/operations/deployment-mode/) documentation.

 Application Admins should have write permissions on the Gateway resource, but only in their specific namespaces, and Application Developers should only hold write permissions on Route resources. To enact this access control schema, follow the [Write Permissions for Advanced 4 Tier Model](https://gateway-api.sigs.k8s.io/concepts/security-model/#write-permissions-for-advanced-4-tier-model) described in the Kubernetes Gateway API security model. Examples of secured gateway-route topologies can be found [here](https://gateway-api.sigs.k8s.io/concepts/api-overview/#attaching-routes-to-gateways) within the Kubernetes Gateway API docs.

 Optionally, consider a GitOps model, where only the GitOps operator has the permission to deploy or modify custom resources in production.

### EGTM-014 Malicious image admission

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-014|EGTM-CS-006|Container Security|High|

 **Risk**: There is a risk that a supply chain attack on Envoy Gateway results in an arbitrary compromise of the confidentiality, integrity or availability of tenant data.

 **Threat**: Supply chain threat actor introduces malicious code into Envoy Gateway or Proxy.

 **Recommendation**: The Envoy Gateway project should continue to work towards conformance with supply-chain security best practices throughout the project lifecycle (for example, as set out in the [CNCF Software Supply Chain Best Practices Whitepaper](https://github.com/cncf/tag-security/blob/main/supply-chain-security/supply-chain-security-paper/CNCF_SSCP_v1.pdf)). Adherence to [Supply-chain Levels for Software Artefacts](https://slsa.dev/) (SLSA) standards is crucial for maintaining the security of the system. Employ version control systems to monitor the source and build platforms and assign responsibility to a specific stakeholder.

 Integrate a supply chain security tool such as Sigstore, which provides native capabilities for signing and verifying container images and software artefacts. [Software Bill of Materials](https://www.cisa.gov/sbom) (SBOM), [Vulnerability Exploitability eXchange](https://ntia.gov/files/ntia/publications/vex_one-page_summary.pdf) (VEX), and signed artefacts should also be incorporated into the security protocol.

### EGTM-020 Out of date or misconfigured Envoy Proxy image

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-020|EGTM-CS-009|Container Security|High|

 **Risk**: There is a risk that a threat actor exploits an Envoy Proxy vulnerability to remote code execution (RCE) due to out of date or misconfigured Envoy Proxy pod deployment, compromising the confidentiality and integrity of Envoy Proxy along with the availability of the proxy service.

 **Threat**: Deployment of an Envoy Proxy or Gateway image containing exploitable CVEs.

 **Recommendation**: Always use the latest version of the Envoy Proxy image. Regularly check for updates and patch the system as soon as updates become available. Implement a CI/CD pipeline that includes security checks for images and prevents deployment of insecure configurations. A suitable tool should be chosen to provide container vulnerability scanning to mitigate the risk of known vulnerabilities.

 Utilise the [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) controller to enforce [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) and configure the [pod security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) to limit its capabilities per the principle of least privilege.

### EGTM-022 Credentials are stored as Kubernetes Secrets

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-022|EGTM-CS-010|Container Security|High|

 **Risk**: There is a risk that the OIDC client secret (for OIDC authentication) and user password hashes (for basic authentication) get leaked due to misconfigured RBAC permissions.

 **Threat**: Unauthorised access to the application due to credential leakage.

 **Recommendation**: Ensure that only authorised users and service accounts are able to access secrets. This is especially important in namespaces where SecurityPolicy objects are configured, since those namespaces are the ones to store secrets containing the client secret (in OIDC scenarios) and user password hashes (in basic authentication scenarios).

 To do so, minimise the use of ClusterRoles and Roles allowing listing and getting secrets. Perform periodic audits of RBAC permissions.

### EGTM-023 Weak Authentication

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-023|EGTM-EG-007|Envoy Gateway|High|

 **Risk**: There is a risk of unauthorised access due to the use of basic authentication, which does not enforce any password restriction in terms of complexity and length. In addition, password hashes are stored in SHA1 format, which is a deprecated hashing function.

 **Threat**: Unauthorised access to the application due to weak authentication mechanisms.

 **Recommendation**: It is recommended to make use of stronger authentication mechanisms (i.e. JWT authentication and OIDC authentication) instead of basic authentication. These authentication mechanisms have many advantages, such as the use of short-lived credentials and a central management of security policies through the identity provider.

## Medium Priority Findings

### EGTM-008 Misconfiguration of Envoy Gateway static config

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-008|EGTM-EG-003|Envoy Gateway|Medium|

 **Risk**: There is a risk of a threat actor misconfiguring static config and compromising the integrity of Envoy Gateway, ultimately leading to the compromised confidentiality, integrity, or availability of tenant data and cluster resources.

 **Threat**: Accidental or deliberate misconfiguration of static configuration leads to a misconfigured deployment of Envoy Gateway, for example logging parameters could be modified or global rate limiting configuration misconfigured.

 **Recommendation**: Implement a GitOps model, utilising Kubernetes\' Role-Based Access Control (RBAC) and adhering to the principle of least privilege to minimise human intervention on the cluster. For instance, tools like [Flux](https://fluxcd.io/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) can be used for declarative GitOps deployments, ensuring all changes are tracked and reviewed. Additionally, configure your source control management (SCM) system to include mandatory pull request (PR) reviews, commit signing, and protected branches to ensure only authorised changes can be committed to the start-up configuration.

### EGTM-010 Weak pod security contexts and policies

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-010|EGTM-CS-005|Container Security|Medium|

 **Risk**: There is a risk that a threat actor exploits a weak pod security context, compromising the CIA of a node and the resources / services which run on it.

 **Threat**: Threat Actor who has compromised a pod exploits weak security context to escape to a node, potentially leading to the compromise of Envoy Proxy or Gateway running on the same node.

 **Recommendation**: To mitigate this risk, apply [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) at a minimum of [Baseline](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) level to all namespaces, especially those containing Envoy Gateway and Proxy Pods. Pod security standards are implemented through K8s [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) to provide [admission control modes](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces) (enforce, audit, and warn) for namespaces. Pod security standards can be enforced by namespace labels as shown [here](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/), to enforce a baseline level of pod security to specific namespaces.

 Further enhance the security by implementing a sandboxing solution such as [gVisor](https://gvisor.dev/) for Envoy Gateway and Proxy Pods to isolate the application from the host kernel. This can be set within the runtimeClassName of the Pod specification.

### EGTM-012 ClusterRoles and Roles with permission to deploy ReferenceGrants

|**ID**|**UID**|**Category**|**Priority**|
|--------------|----------------|----------------------|-----------------|
|EGTM-012|EGTM-GW-004|Gateway API|Medium|

 **Risk**: There is a risk that a threat actor could abuse excessive RBAC privileges to create ReferenceGrant resources. These resources could then be used to create cross-namespace communication, leading to unauthorised access to the application. This could compromise the confidentiality and integrity of resources and configuration in the affected namespaces and potentially disrupt the availability of services that rely on these object references.

 **Threat**: A ReferenceGrant is created, which validates traffic to cross namespace trust boundaries without a valid business reason, such as a route in one tenant\'s namespace referencing a backend in another.

 **Recommendation**: Ensure that the ability to create ReferenceGrant resources is restricted to the minimum number of people. Pay special attention to ClusterRoles that allow that action.

### EGTM-018 Network Denial of Service (DoS)

|**ID**|**UID**|**Category**|**Priority**|
|--------------|----------------|----------------------|-----------------|
|EGTM-018|EGTM-GW-006|Gateway API|Medium|

 **Risk**: There is a risk that malicious requests could lead to a Denial of Service (DoS) attack, thereby reducing API gateway availability due to misconfigurations in rate-limiting or load balancing controls, or a lack of route timeout enforcement.

 **Threat**: Reduced API gateway availability due to an attacker\'s maliciously crafted request (e.g., QoD) potentially inducing a Denial of Service (DoS) attack.

 **Recommendation**: To ensure high availability and to mitigate potential security threats, adhere to the Envoy Gateway documentation for the configuration of a [rate-limiting](https://gateway.envoyproxy.io/v0.6.0/user/rate-limit/) filter and load balancing.

 Further, adhere to best practices for configuring Envoy Proxy as an edge proxy documented [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge#configuring-envoy-as-an-edge-proxy) within the EnvoyProxy docs. This involves configuring TCP and HTTP proxies with specific settings, including restricting access to the admin endpoint, setting the [overload manager](https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager#config-overload-manager) and [listener](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#envoy-v3-api-field-config-listener-v3-listener-per-connection-buffer-limit-bytes) / [cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-per-connection-buffer-limit-bytes) buffer limits, enabling [use_remote_address](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-use-remote-address), setting [connection and stream timeouts](https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts#faq-configuration-timeouts), limiting [maximum concurrent streams](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-max-concurrent-streams), setting [initial stream window size limit](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-initial-stream-window-size), and configuring action on [headers_with_underscores](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-httpprotocoloptions-headers-with-underscores-action).

 [Path normalisation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-normalize-path) should be enabled to minimise path confusion vulnerabilities. These measures help protect against volumetric threats such as Denial of Service (DoS) attacks. Utilise custom resources to implement policy attachment, thereby exposing request limit configuration for route types.

### EGTM-019 JWT-based authentication replay attacks

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-019|EGTM-DP-004|Container Security|Medium|

 **Risk**: There is a risk that replay attacks using stolen or reused JSON Web Tokens (JWTs) can compromise transmission integrity, thereby undermining the confidentiality and integrity of the data plane.

 **Threat**: Transmission integrity is compromised due to replay attacks using stolen or reused JSON Web Tokens (JWTs).

 **Recommendation**: Comply with JWT best practices for enhanced security, paying special attention to the use of short-lived tokens, which reduce the window of opportunity for a replay attack. The [exp](https://datatracker.ietf.org/doc/html/rfc7519#page-9) claim can be used to set token expiration times.

### EGTM-024 Excessive privileges via extension policies

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-024|EGTM-EG-008|Envoy Gateway|Medium|

 **Risk**: There is a risk of developers getting more privileges than required due to the use of SecurityPolicy, ClientTrafficPolicy, EnvoyPatchPolicy and BackendTrafficPolicy. These resources can be attached to a Gateway resource. Therefore, a developer with permission to deploy them would be able to modify a Gateway configuration by targeting the gateway in the policy manifest. This conflicts with the [Advanced 4 Tier Model](https://gateway-api.sigs.k8s.io/concepts/security-model/#write-permissions-for-advanced-4-tier-model), where developers do not have write permissions on Gateways.

 **Threat**: Excessive developer permissions lead to a misconfiguration and/or unauthorised access.

 **Recommendation**: Considering the Tenant C scenario (represented in the Architecture Diagram), if a developer can create SecurityPolicy, ClientTrafficPolicy, EnvoyPatchPolicy or BackendTrafficPolicy objects in namespace C, they would be able to modify a Gateway configuration by attaching the policy to the gateway. In such scenarios, it is recommended to either:

 a.  Create a separate namespace, where developers have no permissions, to host tenant C\'s gateway. Note that, due to design decisions, the SecurityPolicy/EnvoyPatchPolicy/ClientTrafficPolicy/BackendTrafficPolicy object can only target resources deployed in the same namespace. Therefore, having a separate namespace for the gateway would prevent developers from attaching the policy to the gateway.

 b.  Forbid the creation of these policies for developers in namespace C.

 On the other hand, in scenarios similar to tenants A and B, where a shared gateway namespace is in place, this issue is more limited. Note that in this scenario, developers don\'t have access to the shared gateway namespace.

 In addition, it is important to mention that EnvoyPatchPolicy resources can also be attached to GatewayClass resources. This means that, in order to comply with the Advanced 4 Tier model, individuals with the Application Administrator role should not have access to this resource either.

## Low Priority Findings

### EGTM-003 Misconfiguration leads to insecure TLS settings

|**ID**|**UID**|**Category**|**Priority**|
|--------------|--------------|------------------------|-----------------|
|EGTM-003|EGTM-EG-001|Envoy Gateway|Low|

 **Risk**: There is a risk that a threat actor could downgrade the security of proxied connections by configuring a weak set of cipher suites, compromising the confidentiality and integrity of proxied traffic.

 **Threat**: Exploit weak cipher suite configuration to downgrade security of proxied connections.

 **Recommendation**: Users operating in highly regulated environments may need to tightly control the TLS protocol and associated cipher suites, blocking non-conforming incoming connections to the gateway.

 EnvoyProxy bootstrap config can be customised as per the [customise EnvoyProxy](https://gateway.envoyproxy.io/latest/user/operations/customize-envoyproxy/) documentation. In addition, from v.1.0.0, it is possible to configure common TLS properties for a Gateway or XRoute through the [ClientTrafficPolicy](https://gateway.envoyproxy.io/latest/api/extension_types/#clienttrafficpolicy) object.

### EGTM-005 Envoy Gateway Helm chart deployment does not set AppArmor and Seccomp profiles

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-005|EGTM-CP-002|Container Security|Low|

 **Risk**: Threat actor who has obtained access to Envoy Gateway pod could exploit the lack of AppArmor and Seccomp profiles in the Envoy Gateway deployment to attempt a container breakout, given the presence of an exploitable vulnerability, potentially impacting the confidentiality and integrity node resources.

 **Threat**: Unauthorised syscalls and malicious code running in the Envoy Gateway pod.

 **Recommendation**: Implement [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) policies by setting \<container_name\>: \<profile_ref\> within container.apparmor.security.beta.kubernetes.io (note, this config is set *per container*). Well-defined AppArmor policies may provide greater protection from unknown threats.

 Enforce [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/) profiles by setting the seccompProfile under securityContext. Ideally, a [fine-grained](https://kubernetes.io/docs/tutorials/security/seccomp/#create-pod-with-a-seccomp-profile-that-only-allows-necessary-syscalls) profile should be used to restrict access to only necessary syscalls, however the \--seccomp-default flag can be set to resort to [RuntimeDefault](https://kubernetes.io/docs/tutorials/security/seccomp/#create-pod-that-uses-the-container-runtime-default-seccomp-profile) which provides a container runtime specific. Example seccomp profiles can be found [here](https://kubernetes.io/docs/tutorials/security/seccomp/#download-profiles).

 To further enhance pod security, consider implementing [SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) via seLinuxOptions for additional syscall attack surface reduction. Setting readOnlyRootFilesystem == true enforces an immutable root filesystem, preventing the addition of malicious binaries to the PATH and increasing the attack cost. Together, these configuration items improve the pods [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

### EGTM-006 Envoy Proxy pods deployed with a shell enabled

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-006|EGTM-CS-004|Container Security|Low|

 **Risk**: There is a risk that a threat actor exploits a vulnerability in Envoy Proxy to expose a reverse shell, enabling them to compromise the confidentiality, integrity and availability of tenant data via a secondary attack.

 **Threat**: If an external attacker managed to exploit a vulnerability in Envoy, the presence of a shell would be greatly helpful for the attacker in terms of potentially pivoting, escalating, or establishing some form of persistence.

 **Recommendation**: By default, Envoy uses a [distroless](https://github.com/GoogleContainerTools/distroless) image since v.0.6.0, which does not ship a shell. Therefore, ensure EnvoyProxy image is up-to-date and patched with the latest stable version.

 If using private EnvoyProxy images, use a lightweight EnvoyProxy image without a shell or debugging tool(s) which may be useful for an attacker.

 An [AuditPolicy](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#audit-policy) (audit.k8s.io/v1beta1) can be configured to record API calls made within your cluster, allowing for identification of malicious traffic and enabling incident response. Requests are recorded based on stages which delineate between the lifecycle stage of the request made (e.g., RequestReceived, ResponseStarted, & ResponseComplete).

### EGTM-011 Route Bindings on custom labels

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-011|EGTM-GW-003|Gateway API|Low|

 **Risk**: There is a risk that a gateway owner (or someone with the ability to set namespace labels) maliciously or accidentally binds routes across namespace boundaries, potentially compromising the confidentiality and integrity of traffic in a multitenant scenario.

 **Threat**: If a Route Binding within a Gateway Listener is configured based on a custom label, it could allow a malicious internal actor with the ability to label namespaces to change the set of namespaces supported by the Gateway.

 **Recommendation**: Consider the use of custom admission control to restrict what labels can be set on namespaces through tooling such as [Kubewarden](https://kyverno.io/policies/pod-security/), [Kyverno](https://github.com/kubewarden), and [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper). Route binding should follow the Kubernetes Gateway API security model, as shown [here](https://gateway-api.sigs.k8s.io/concepts/security-model/#1-route-binding), to connect gateways in different namespaces.

### EGTM-013 GatewayClass namespace validation is not configured

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-013|EGTM-GW-005|Gateway API|Low|

 **Risk**: There is a risk that an unauthorised actor deploys an unauthorised GatewayClass due to GatewayClass namespace validation not being configured, leading to non-compliance with business and security requirements.

 **Threat**: Unauthorised deployment of Gateway resource via GatewayClass template which crosses namespace trust boundaries.

 **Recommendation**: Leverage GatewayClass namespace validation to limit the namespaces where GatewayClasses can be run through a tool such as [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper). Reference pull request \#[24](https://github.com/open-policy-agent/gatekeeper-library/pull/24) within gatekeeper-library which outlines how to add GatewayClass namespace validation through a GatewayClassNamespaces API resource kind within the constraints.gatekeeper.sh/v1beta1 apiGroup.

### EGTM-015 ServiceAccount token authentication

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-015|EGTM-CS-007|Container Security|Low|

 **Risk**: There is a risk that threat actors could exploit ServiceAccount tokens for illegitimate authentication, thereby leading to privilege escalation and the undermining of gateway API resources\' integrity, confidentiality, and availability.

 **Threat**: The threat arises from threat actors impersonating the envoy-gateway ServiceAccount through the replay of ServiceAccount tokens, thereby achieving escalated privileges and gaining unauthorised access to Kubernetes resources.

 **Recommendation**: Limit the creation of ServiceAccounts to only when necessary, specifically refraining from using default service account tokens, especially for high-privilege service accounts. For legacy clusters running Kubernetes version 1.21 or earlier, note that ServiceAccount tokens are long-lived by default. To disable the automatic mounting of the service account token, set automountServiceAccountToken: false in the PodSpec.

### EGTM-016 Misconfiguration leads to lack of Envoy Proxy access activity visibility

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-016|EGTM-EG-004|Envoy Gateway|Low|

 **Risk**: There is a risk that threat actors establish persistence and move laterally through the cluster unnoticed due to limited visibility into access and application-level activity.

 **Threat**: Threat actors establish persistence and move laterally through the cluster unnoticed.

 **Recommendation**: Configure [access logging](https://gateway.envoyproxy.io/latest/contributions/design/accesslog/) in the EnvoyProxy. Use [ProxyAccessLogFormatType](https://gateway.envoyproxy.io/latest/design/accesslog/#proxyaccesslog-api-type) (Text or JSON) to specify the log format and ensure that the logs are sent to the desired sink types by setting the [ProxyAccessLogSinkType](https://gateway.envoyproxy.io/latest/api/extension_types/#proxyaccesslogsinktype). Make use of [FileEnvoyProxyAccessLog](https://gateway.envoyproxy.io/latest/api/extension_types/#fileenvoyproxyaccesslog) or [OpenTelemetryEnvoyProxyAccessLog](https://gateway.envoyproxy.io/latest/api/extension_types/#opentelemetryenvoyproxyaccesslog) to configure File and OpenTelemetry sinks, respectively. If the settings aren\'t defined, the default format is sent to stdout.

 Additionally, consider leveraging a central logging mechanism such as [Fluentd](https://github.com/fluent/fluentd) to enhance visibility into access activity and enable effective incident response (IR).

### EGTM-017 Misconfiguration leads to lack of Envoy Gateway activity visibility

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-017|EGTM-EG-005|Envoy Gateway|Low|

 **Risk**: There is a risk that an insider misconfigures an envoy gateway component and goes unnoticed due to a low-touch logging configuration (via default) which responsible stakeholders are not aptly aware of or have immediate access to.

 **Threat**: The threat emerges from an insider misconfiguring an Envoy Gateway component without detection.

 **Recommendation**: Configure the logging level of the Envoy Gateway using the \'level\' field in [EnvoyGatewayLogging](https://gateway.envoyproxy.io/latest/api/extension_types/#envoygatewaylogging). Ensure the appropriate logging levels are set for relevant components such as \'gateway-api\', \'xds-translator\', or \'global-ratelimit\'. If left unspecified, the logging level defaults to \"info\", which may not provide sufficient detail for security monitoring.

 Employ a centralised logging mechanism, like [Fluentd](https://github.com/fluent/fluentd), to enhance visibility into application-level activity and to enable efficient incident response.

### EGTM-021 Exposed Envoy Proxy admin interface

|**ID**|**UID**|**Category**|**Priority**|
|--------------|---------------|-----------------------|-----------------|
|EGTM-021|EGTM-EG-006|Envoy Gateway|Low|

 **Risk**: There is a risk that the admin interface is exposed without valid business reason, increasing the attack surface.

 **Threat**: Exposed admin interfaces give internal attackers the option to affect production traffic in unauthorised ways, and the option to exploit any vulnerabilities which may be present in the admin interface (e.g. by orchestrating malicious GET requests to the admin interface through CSRF, compromising Envoy Proxy global configuration or shutting off the service entirely e.g. /quitquitquit).

 **Recommendation**: The Envoy Proxy admin interface is only exposed to localhost, meaning that it is secure by default. However, due to the risk of misconfiguration, this recommendation is included.

 Due to the importance of the admin interface, it is recommended to ensure that Envoy Proxies have not been accidentally misconfigured to expose the admin interface to untrusted networks.

### EGTM-025 Envoy Proxy pods deployed running as root user in the container

|**ID**|**UID**|**Category**|**Priority**|
|--------------|--------------|------------------------|-----------------|
|EGTM-025|EGTM-CS-011|Container Security|Low|

**Risk**: The presence of a vulnerability, be it in the kernel or another system component, when coupled with containers running as root, could enable a threat actor to escape the container, thereby compromising the confidentiality, integrity, or availability of cluster resources

 **Threat**: The Envoy Proxy container's root-user configuration can be leveraged by an attacker to escalate privileges, execute a container breakout, and traverse across trust boundaries.

 **Recommendation**: By default, Envoy Gateway deployments do not use root users. Nonetheless, in case a custom image or deployment manifest is to be used, make sure Envoy Proxy pods run as a non-root user with a high UID within the container.

Set runAsUser and runAsGroup security context options to specific UIDs (e.g., runAsUser: 1000 & runAsGroup: 3000) to ensure the container operates with the stipulated non-root user and group ID. If using helm chart deployment, define the user and group ID in the values.yaml file or via the command line during helm install / upgrade.

## Appendix

### In Scope Threat Actor Details

|Threat Actor | Capability | Personal Motivation | Envoy Gateway Attack Samples|
|-|-|-|-|
|Application Developer | Leverage internal knowledge and personal access to the Envoy Gateway infrastructure to move laterally and transit trust boundaries | Disgruntled / personal grievances.<br/><br/> Financial incentives | Misconfigure XRoute resources to expose internal applications.<br/><br/>Misconfigure SecurityPolicy objects, reducing the security posture of an application.|
|Application Administrator | Abuse privileged status to disrupt operations and tenant cluster services through Envoy Gateway misconfig | Disgruntled / personal grievances.<br/><br/> Financial incentives | Create malicious routes to internal applications.<br/><br/> Introduce malicious Envoy Proxy images.<br/><br/> Expose the Envoy Proxy Admin interface.|
|Cluster Operator | Alter application-level deployments by misconfiguring resource dependencies & SCM to introduce vulnerabilities | Disgruntled / personal grievances. <br/><br/> Financial incentives.<br/><br/> Notoriety | Deploy malicious resources to expose internal applications.<br/><br/> Access authentication secrets.<br/><br/> Fall victim to phishing attacks and inadvertently share authentication credentials to cloud infrastructure or Kubernetes clusters.|
|Vandal: Script Kiddie, Trespasser | Uses publicly available tools and applications (Nmap,Metasploit, CVE PoCs) | Curiosity.<br/><br/> Personal fame through defacement / denial of service of prominent public facing web resources | Small scale DOS.<br/><br/> Launches prepackaged exploits, runs crypto mining tools.<br/><br/> Exploit public-facing application services such as the bastion host to gain an initial foothold in the environment|
|Motivated individual: Political activist, Thief, Terrorist | Write tools and exploits required for their means if sufficiently motivated.<br/><br/> Tend to use these in a targeted fashion against specific organisations. May combine publicly available exploits in a targeted fashion. Tamper with open source supply chains | Personal Gain (Political or Ideological) | Phishing, DDOS, exploit known vulnerabilities.<br/><br/> Compromise third-party components such as Helm charts and container images to inject malicious codes to propagate access throughout the environment.|
|Organised crime: syndicates, state-affiliated groups | Write tools and exploits required for their means.<br/><br/> Tend to use these in a non-targeted fashion, unless motivation is sufficiently high.<br/><br/> Devotes considerable resources, writes exploits, can bribe/coerce, can launch targeted attacks | Ransom.<br/><br/> Mass extraction of PII / credentials / PCI data. <br/><br/>Financial incentives | Social Engineering, phishing, ransomware, coordinated attacks.<br/><br/> Intercept and replay JWT tokens (via MiTM) between tenant user(s) and envoy gateway to modify app configs in-transit|

### Identified Threats by Priority

|ID|UID|Category|Risk|Threat|Priority|Recommendation|
|-|-|-|-|-|-|-|
|EGTM-001|EGTM-GW-001|Gateway API| Self-signed certificates (which do not comply with PKI best practices) could lead to unauthorised access to the private key associated with the certificate used for inbound TLS termination at Envoy Proxy, compromising the confidentiality and integrity of proxied traffic.<br/><br/>| Compromise of the private key associated with the certificate used for inbound TLS terminating at Envoy Proxy.<br/><br/>|High| The Envoy Gateway quickstart guide demonstrates how to set up a Secure Gateway using an example where a self-signed root certificate is created using openssl. As stated in the Envoy Gateway documentation, this is not a suitable configuration for Production usage. It is recommended that PKI best practices are followed, whereby certificates are signed by an Intermediary CA which sits underneath an organisational \'offline\' Root CA.<br/><br/> PKI best practices should also apply to the management of client certificates when using mTLS. The Envoy Gateway [mTLS](https://gateway.envoyproxy.io/latest/user/security/mutual-tls/) guide shows how to set up client certificates using self-signed certificates. In the same way as gateway certificates and, as mentioned in the documentation, this configuration should not be used in production environments.|
|EGTM-002|EGTM-CS-001|Container Security| There is a risk that a threat actor could compromise the Kubernetes secret containing the Envoy private key, allowing the attacker to decrypt Envoy Proxy traffic, compromising the confidentiality of proxied traffic.<br/><br/>| Kubernetes secret containing the Envoy private key is compromised and used to decrypt proxied traffic.<br/><br/>|High| Certificate management best practices mandate short-lived key material where practical, meaning that a mechanism for rotation of private keys and certificates is required, along with a way for certificates to be mounted into Envoy containers. If Kubernetes secrets are used, when a certificate expires, the associated secret must be updated, and Envoy containers must be redeployed. Instead of a manual configuration, it is recommended that [cert-manager](https://github.com/cert-manager/cert-manager) is used.|
|EGTM-004|EGTM-K8-002|Container Security| There is a risk that a threat actor could abuse misconfigured RBAC to access the Envoy Gateway ClusterRole (envoy-gateway-role) and use it to expose all secrets across the cluster, thus compromising the confidentiality and integrity of tenant data.<br/><br/>| Compromised Envoy Gateway or misconfigured ClusterRoleBinding (envoy-gateway-rolebinding) to Envoy Gateway ClusterRole (envoy-gateway-role), provides access to resources and secrets in different namespaces.<br/><br/>|High| Users should be aware that Envoy Gateway uses a ClusterRole (envoy-gateway-role) when deployed via the Helm chart, to allow management of Envoy Proxies across different namespaces. This ClusterRole is powerful and includes the ability to read secrets in namespaces which may not be within the purview of Envoy Gateway.<br/><br/> Kubernetes best-practices involve restriction of ClusterRoleBindings, with the use of RoleBindings where possible to limit access per namespace by specifying the namespace in metadata. Namespace isolation reduces the impact of compromise from cluster-scoped roles. Ideally, fine-grained K8s roles should be created per the principle of least privilege to ensure they have the minimum access necessary for role functions.<br/><br/> The pull request \#[1656](https://github.com/envoyproxy/gateway/pull/1656) introduced the use of Roles and RoleBindings in [namespaced mode](https://gateway.envoyproxy.io/latest/api/extension_types/#kuberneteswatchmode). This feature can be leveraged to reduce the amount of permissions required by the Envoy Gateway.|
|EGTM-007|EGTM-EG-002|Envoy Gateway| There is a risk that a threat actor could exploit misconfigured Kubernetes RBAC to create or modify Gateway API resources with no business need, potentially leading to the compromise of the confidentiality, integrity, and availability of resources and traffic within the cluster.<br/><br/>| Unauthorised creation or misconfiguration of Gateway API resources by a threat actor with cluster-scoped access.<br/><br/>|High| Configure the apiGroup and resource fields in RBAC policies to restrict access to [Gateway](https://gateway-api.sigs.k8s.io/) and [GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/) resources. Enable namespace isolation by using the namespace field, preventing unauthorised access to gateways in other namespaces.|
|EGTM-009|EGTM-GW-002|Gateway API| There is a risk that a co-tenant misconfigures Gateway or Route resources, compromising the confidentiality, integrity, and availability of routed traffic through Envoy Gateway.<br/><br/>| Malicious or accidental co-tenant misconfiguration of Gateways and Routes associated with other application teams.<br/><br/>|High| Dedicated Envoy Gateways should be provided to each tenant within their respective namespace. A one-to-one relationship should be established between GatewayClass and Gateway resources, meaning that each tenant namespace should have their own GatewayClass watched by a unique Envoy Gateway Controller as defined here in the [Deployment Mode](https://gateway.envoyproxy.io/latest/user/operations/deployment-mode/) documentation.<br/><br/> Application Admins should have write permissions on the Gateway resource, but only in their specific namespaces, and Application Developers should only hold write permissions on Route resources. To enact this access control schema, follow the [Write Permissions for Advanced 4 Tier Model](https://gateway-api.sigs.k8s.io/concepts/security-model/#write-permissions-for-advanced-4-tier-model) described in the Kubernetes Gateway API security model. Examples of secured gateway-route topologies can be found [here](https://gateway-api.sigs.k8s.io/concepts/api-overview/#attaching-routes-to-gateways) within the Kubernetes Gateway API docs.<br/><br/> Optionally, consider a GitOps model, where only the GitOps operator has the permission to deploy or modify custom resources in production.|
|EGTM-014|EGTM-CS-006|Container Security| There is a risk that a supply chain attack on Envoy Gateway results in an arbitrary compromise of the confidentiality, integrity or availability of tenant data.<br/><br/>| Supply chain threat actor introduces malicious code into Envoy Gateway or Proxy.<br/><br/>|High| The Envoy Gateway project should continue to work towards conformance with supply-chain security best practices throughout the project lifecycle (for example, as set out in the [CNCF Software Supply Chain Best Practices Whitepaper](https://github.com/cncf/tag-security/blob/main/supply-chain-security/supply-chain-security-paper/CNCF_SSCP_v1.pdf). Adherence to [Supply-chain Levels for Software Artefacts](https://slsa.dev/) (SLSA) standards is crucial for maintaining the security of the system. Employ version control systems to monitor the source and build platforms and assign responsibility to a specific stakeholder.<br/><br/> Integrate a supply chain security tool such as Sigstore, which provides native capabilities for signing and verifying container images and software artefacts. [Software Bill of Materials](https://www.cisa.gov/sbom) (SBOM), [Vulnerability Exploitability eXchange](https://ntia.gov/files/ntia/publications/vex_one-page_summary.pdf) (VEX), and signed artefacts should also be incorporated into the security protocol.|
|EGTM-020|EGTM-CS-009|Container Security| There is a risk that a threat actor exploits an Envoy Proxy vulnerability to remote code execution (RCE) due to out of date or misconfigured Envoy Proxy pod deployment, compromising the confidentiality and integrity of Envoy Proxy along with the availability of the proxy service.<br/><br/>| Deployment of an Envoy Proxy or Gateway image containing exploitable CVEs.<br/><br/>|High| Always use the latest version of the Envoy Proxy image. Regularly check for updates and patch the system as soon as updates become available. Implement a CI/CD pipeline that includes security checks for images and prevents deployment of insecure configurations. A tool such as Snyk can be used to provide container vulnerability scanning to mitigate the risk of known vulnerabilities.<br/><br/> Utilise the [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) controller to enforce [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) and configure the [pod security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) to limit its capabilities per the principle of least privilege.|
|EGTM-022|EGTM-CS-010|Container Security| There is a risk that the OIDC client secret (for OIDC authentication) and user password hashes (for basic authentication) get leaked due to misconfigured RBAC permissions.<br/><br/>| Unauthorised access to the application due to credential leakage.<br/><br/>|High| Ensure that only authorised users and service accounts are able to access secrets. This is especially important in namespaces where SecurityPolicy objects are configured, since those namespaces are the ones to store secrets containing the client secret (in OIDC scenarios) and user password hashes (in basic authentication scenarios).<br/><br/> To do so, minimise the use of ClusterRoles and Roles allowing listing and getting secrets. Perform periodic audits of RBAC permissions.|
|EGTM-023|EGTM-EG-007|Envoy Gateway| There is a risk of unauthorised access due to the use of basic authentication, which does not enforce any password restriction in terms of complexity and length. In addition, password hashes are stored in SHA1 format, which is a deprecated hashing function.<br/><br/>| Unauthorised access to the application due to weak authentication mechanisms.<br/><br/>|High| It is recommended to make use of stronger authentication mechanisms (i.e. JWT authentication and OIDC authentication) instead of basic authentication. These authentication mechanisms have many advantages, such as the use of short-lived credentials and a central management of security policies through the identity provider.|
|EGTM-008|EGTM-EG-003|Envoy Gateway| There is a risk of a threat actor misconfiguring static config and compromising the integrity of Envoy Gateway, ultimately leading to the compromised confidentiality, integrity, or availability of tenant data and cluster resources.<br/><br/>| Accidental or deliberate misconfiguration of static configuration leads to a misconfigured deployment of Envoy Gateway, for example logging parameters could be modified or global rate limiting configuration misconfigured.<br/><br/>|Medium| Implement a GitOps model, utilising Kubernetes\' Role-Based Access Control (RBAC) and adhering to the principle of least privilege to minimise human intervention on the cluster. For instance, tools like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) can be used for declarative GitOps deployments, ensuring all changes are tracked and reviewed. Additionally, configure your source control management (SCM) system to include mandatory pull request (PR) reviews, commit signing, and protected branches to ensure only authorised changes can be committed to the start-up configuration.|
|EGTM-010|EGTM-CS-005|Container Security| There is a risk that a threat actor exploits a weak pod security context, compromising the CIA of a node and the resources / services which run on it.<br/><br/>| Threat Actor who has compromised a pod exploits weak security context to escape to a node, potentially leading to the compromise of Envoy Proxy or Gateway running on the same node.<br/><br/>|Medium| To mitigate this risk, apply [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) at a minimum of [Baseline](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) level to all namespaces, especially those containing Envoy Gateway and Proxy Pods. Pod security standards are implemented through K8s [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) to provide [admission control modes](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces) (enforce, audit, and warn) for namespaces. Pod security standards can be enforced by namespace labels as shown [here](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/), to enforce a baseline level of pod security to specific namespaces.<br/><br/> Further enhance the security by implementing a sandboxing solution such as [gVisor](https://gvisor.dev/) for Envoy Gateway and Proxy Pods to isolate the application from the host kernel. This can be set within the runtimeClassName of the Pod specification.|
|EGTM-012|EGTM-GW-004|Gateway API| There is a risk that a threat actor could abuse excessive RBAC privileges to create ReferenceGrant resources. These resources could then be used to create cross-namespace communication, leading to unauthorised access to the application. This could compromise the confidentiality and integrity of resources and configuration in the affected namespaces and potentially disrupt the availability of services that rely on these object references.<br/><br/>| A ReferenceGrant is created, which validates traffic to cross namespace trust boundaries without a valid business reason, such as a route in one tenant\'s namespace referencing a backend in another.<br/><br/>|Medium| Ensure that the ability to create ReferenceGrant resources is restricted to the minimum number of people. Pay special attention to ClusterRoles that allow that action.|
|EGTM-018|EGTM-GW-006|Gateway API| There is a risk that malicious requests could lead to a Denial of Service (DoS) attack, thereby reducing API gateway availability due to misconfigurations in rate-limiting or load balancing controls, or a lack of route timeout enforcement.<br/><br/>| Reduced API gateway availability due to an attacker\'s maliciously crafted request (e.g., QoD) potentially inducing a Denial of Service (DoS) attack.<br/><br/>|Medium| To ensure high availability and to mitigate potential security threats, adhere to the Envoy Gateway documentation for the configuration of a [rate-limiting](https://gateway.envoyproxy.io/v0.6.0/user/rate-limit/) filter and load balancing.<br/><br/> Further, adhere to best practices for configuring Envoy Proxy as an edge proxy documented [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge#configuring-envoy-as-an-edge-proxy) within the EnvoyProxy docs. This involves configuring TCP and HTTP proxies with specific settings, including restricting access to the admin endpoint, setting the [overload manager](https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager#config-overload-manager) and [listener](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#envoy-v3-api-field-config-listener-v3-listener-per-connection-buffer-limit-bytes) / [cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-per-connection-buffer-limit-bytes) buffer limits, enabling [use_remote_address](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-use-remote-address), setting [connection and stream timeouts](https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts#faq-configuration-timeouts), limiting [maximum concurrent streams](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-max-concurrent-streams), setting [initial stream window size limit](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-initial-stream-window-size), and configuring action on [headers_with_underscores](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-httpprotocoloptions-headers-with-underscores-action).<br/><br/> [Path normalisation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-normalize-path) should be enabled to minimise path confusion vulnerabilities. These measures help protect against volumetric threats such as Denial of Service (DoS)nattacks. Utilise custom resources to implement policy attachment, thereby exposing request limit configuration for route types.|
|EGTM-019|EGTM-DP-004|Container Security| There is a risk that replay attacks using stolen or reused JSON Web Tokens (JWTs) can compromise transmission integrity, thereby undermining the confidentiality and integrity of the data plane.<br/><br/>| Transmission integrity is compromised due to replay attacks using stolen or reused JSON Web Tokens (JWTs).<br/><br/>|Medium| Comply with JWT best practices for enhanced security, paying special attention to the use of short-lived tokens, which reduce the window of opportunity for a replay attack. The [exp](https://datatracker.ietf.org/doc/html/rfc7519#page-9) claim can be used to set token expiration times.|
|EGTM-024|EGTM-EG-008|Envoy Gateway| There is a risk of developers getting more privileges than required due to the use of SecurityPolicy, ClientTrafficPolicy, EnvoyPatchPolicy and BackendTrafficPolicy. These resources can be attached to a Gateway resource. Therefore, a developer with permission to deploy them would be able to modify a Gateway configuration by targeting the gateway in the policy manifest. This conflicts with the [Advanced 4 Tier Model](https://gateway-api.sigs.k8s.io/concepts/security-model/#write-permissions-for-advanced-4-tier-model), where developers do not have write permissions on Gateways.<br/><br/>| Excessive developer permissions lead to a misconfiguration and/or unauthorised access.<br/><br/>|Medium| Considering the Tenant C scenario (represented in the Architecture Diagram), if a developer can create SecurityPolicy, ClientTrafficPolicy, EnvoyPatchPolicy or BackendTrafficPolicy objects in namespace C, they would be able to modify a Gateway configuration by attaching the policy to the gateway. In such scenarios, it is recommended to either:<br/><br/> a.  Create a separate namespace, where developers have no permissions, > to host tenant C\'s gateway. Note that, due to design decisions, > the > SecurityPolicy/EnvoyPatchPolicy/ClientTrafficPolicy/BackendTrafficPolicy > object can only target resources deployed in the same namespace. > Therefore, having a separate namespace for the gateway would > prevent developers from attaching the policy to the gateway.<br/><br/> b.  Forbid the creation of these policies for developers in namespace C.<br/><br/> On the other hand, in scenarios similar to tenants A and B, where a shared gateway namespace is in place, this issue is more limited. Note that in this scenario, developers don\'t have access to the shared gateway namespace.<br/><br/> In addition, it is important to mention that EnvoyPatchPolicy resources can also be attached to GatewayClass resources. This means that, in order to comply with the Advanced 4 Tier model, individuals with the Application Administrator role should not have access to this resource either.|
|EGTM-003|EGTM-EG-001|Envoy Gateway| There is a risk that a threat actor could downgrade the security of proxied connections by configuring a weak set of cipher suites, compromising the confidentiality and integrity of proxied traffic.<br/><br/>| Exploit weak cipher suite configuration to downgrade security of proxied connections.<br/><br/>|Low| Users operating in highly regulated environments may need to tightly control the TLS protocol and associated cipher suites, blocking non-conforming incoming connections to the gateway.<br/><br/> EnvoyProxy bootstrap config can be customised as per the [customise EnvoyProxy](https://gateway.envoyproxy.io/latest/user/operations/customize-envoyproxy/) documentation. In addition, from v.1.0.0, it is possible to configure common TLS properties for a Gateway or XRoute through the [ClientTrafficPolicy](https://gateway.envoyproxy.io/latest/api/extension_types/#clienttrafficpolicy) object.|
|EGTM-005|EGTM-CP-002|Container Security| Threat actor who has obtained access to Envoy Gateway pod could exploit the lack of AppArmor and Seccomp profiles in the Envoy Gateway deployment to attempt a container breakout, given the presence of an exploitable vulnerability, potentially impacting the confidentiality and integrity of namespace resources.<br/><br/>| Unauthorised syscalls and malicious code running in the Envoy Gateway pod.<br/><br/>|Low| Implement [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) policies by setting \<container_name\>: \<profile_ref\> within container.apparmor.security.beta.kubernetes.io (note, this config is set *per container*). Well-defined AppArmor policies may provide greater protection from unknown threats.<br/><br/> Enforce [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/) profiles by setting the seccompProfile under securityContext. Ideally, a [fine-grained](https://kubernetes.io/docs/tutorials/security/seccomp/#create-pod-with-a-seccomp-profile-that-only-allows-necessary-syscalls) profile should be used to restrict access to only necessary syscalls, however the \--seccomp-default flag can be set to resort to [RuntimeDefault](https://kubernetes.io/docs/tutorials/security/seccomp/#create-pod-that-uses-the-container-runtime-default-seccomp-profile) which provides a container runtime specific. Example seccomp profiles can be found [here](https://kubernetes.io/docs/tutorials/security/seccomp/#download-profiles).<br/><br/> To further enhance pod security, consider implementing [SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) via seLinuxOptions for additional syscall attack surface reduction. Setting readOnlyRootFilesystem == true enforces an immutable root filesystem, preventing the addition of malicious binaries to the PATH and increasing the attack cost. Together, these configuration items improve the pods [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).|
|EGTM-006|EGTM-CS-004|Container Security| There is a risk that a threat actor exploits a vulnerability in Envoy Proxy to expose a reverse shell, enabling them to compromise the confidentiality, integrity and availability of tenant data via a secondary attack.<br/><br/>| If an external attacker managed to exploit a vulnerability in Envoy, the presence of a shell would be greatly helpful for the attacker in terms of potentially pivoting, escalating, or establishing some form of persistence.<br/><br/>|Low| By default, Envoy uses a [distroless](https://github.com/GoogleContainerTools/distroless) image since v.0.6.0, which does not ship a shell. Therefore, ensure EnvoyProxy image is up-to-date and patched with the latest stable version.<br/><br/> If using private EnvoyProxy images, use a lightweight EnvoyProxy image without a shell or debugging tool(s) which may be useful for an attacker.<br/><br/> An [AuditPolicy](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#audit-policy) (audit.k8s.io/v1beta1) can be configured to record API calls made within your cluster, allowing for identification of malicious traffic and enabling incident response. Requests are recorded based on stages which delineate between the lifecycle stage of the request made (e.g., RequestReceived, ResponseStarted, & ResponseComplete).|
|EGTM-011|EGTM-GW-003|Gateway API| There is a risk that a gateway owner (or someone with the ability to set namespace labels) maliciously or accidentally binds routes across namespace boundaries, potentially compromising the confidentiality and integrity of traffic in a multitenant scenario.<br/><br/>| If a Route Binding within a Gateway Listener is configured based on a custom label, it could allow a malicious internal actor with the ability to label namespaces to change the set of namespaces supported by the Gateway<br/><br/>|Low| Consider the use of custom admission control to restrict what labels can be set on namespaces through tooling such as [Kubewarden](https://kyverno.io/policies/pod-security/), [Kyverno](https://github.com/kubewarden), and [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper). Route binding should follow the Kubernetes Gateway API security model, as shown [here](https://gateway-api.sigs.k8s.io/concepts/security-model/#1-route-binding), to connect gateways in different namespaces.|
|EGTM-013|EGTM-GW-005|Gateway API| There is a risk that an unauthorised actor deploys an unauthorised GatewayClass due to GatewayClass namespace validation not being configured, leading to non-compliance with business and security requirements.<br/><br/>| Unauthorised deployment of Gateway resource via GatewayClass template which crosses namespace trust boundaries.<br/><br/>|Low| Leverage GatewayClass namespace validation to limit the namespaces where GatewayClasses can be run through a tool such as using [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper). Reference pull request \#[24](https://github.com/open-policy-agent/gatekeeper-library/pull/24) within gatekeeper-library which outlines how to add GatewayClass namespace validation through a GatewayClassNamespaces API resource kind within the constraints.gatekeeper.sh/v1beta1 apiGroup.|
|EGTM-015|EGTM-CS-007|Container Security| There is a risk that threat actors could exploit ServiceAccount tokens for illegitimate authentication, thereby leading to privilege escalation and the undermining of gateway API resources\' integrity, confidentiality, and availability.<br/><br/>| The threat arises from threat actors impersonating the envoy-gateway ServiceAccount through the replay of ServiceAccount tokens, thereby achieving escalated privileges and gaining unauthorised access to Kubernetes resources.<br/><br/>|Low| Limit the creation of ServiceAccounts to only when necessary, specifically refraining from using default service account tokens, especially for high-privilege service accounts. For legacy clusters running Kubernetes version 1.21 or earlier, note that ServiceAccount tokens are long-lived by default. To disable the automatic mounting of the service account token, set automountServiceAccountToken: false in the PodSpec.|
|EGTM-016|EGTM-EG-004|Envoy Gateway| There is a risk that threat actors establish persistence and move laterally through the cluster unnoticed due to limited visibility into access and application-level activity.<br/><br/>| Threat actors establish persistence and move laterally through the cluster unnoticed.<br/><br/>|Low| Configure [access logging](https://gateway.envoyproxy.io/latest/contributions/design/accesslog/) in the EnvoyProxy. Use [ProxyAccessLogFormatType](https://gateway.envoyproxy.io/latest/design/accesslog/#proxyaccesslog-api-type) (Text or JSON) to specify the log format and ensure that the logs are sent to the desired sink types by setting the [ProxyAccessLogSinkType](https://gateway.envoyproxy.io/latest/api/extension_types/#proxyaccesslogsinktype). Make use of [FileEnvoyProxyAccessLog](https://gateway.envoyproxy.io/latest/api/extension_types/#fileenvoyproxyaccesslog) or [OpenTelemetryEnvoyProxyAccessLog](https://gateway.envoyproxy.io/latest/api/extension_types/#opentelemetryenvoyproxyaccesslog) to configure File and OpenTelemetry sinks, respectively. If the settings aren\'t defined, the default format is sent to stdout.<br/><br/> Additionally, consider leveraging a central logging mechanism such as [Fluentd](https://github.com/fluent/fluentd) to enhance visibility into access activity and enable effective incident response (IR).|
|EGTM-017|EGTM-EG-005|Envoy Gateway| There is a risk that an insider misconfigures an envoy gateway component and goes unnoticed due to a low-touch logging configuration (via default) which responsible stakeholders are not aptly aware of or have immediate access to.<br/><br/>| The threat emerges from an insider misconfiguring an Envoy Gateway component without detection.<br/><br/>|Low| Configure the logging level of the Envoy Gateway using the \'level\' field in [EnvoyGatewayLogging](https://gateway.envoyproxy.io/latest/api/extension_types/#envoygatewaylogging). Ensure the appropriate logging levels are set for relevant components such as \'gateway-api\', \'xds-translator\', or \'global-ratelimit\'. If left unspecified, the logging level defaults to \"info\", which may not provide sufficient detail for security monitoring.<br/><br/> Employ a centralised logging mechanism, like [Fluentd](https://github.com/fluent/fluentd), to enhance visibility into application-level activity and to enable efficient incident response.|
|EGTM-021|EGTM-EG-006|Envoy Gateway| There is a risk that the admin interface is exposed without valid business reason, increasing the attack surface.<br/><br/>| Exposed admin interfaces give internal attackers the option to affect production traffic in unauthorised ways, and the option to exploit any vulnerabilities which may be present in the admin interface (e.g. by orchestrating malicious GET requests to the admin interface through CSRF, compromising Envoy Proxy global configuration or shutting off the service entirely (e.g., /quitquitquit).<br/><br/>|Low| The Envoy Proxy admin interface is only exposed to localhost, meaning that it is secure by default. However, due to the risk of misconfiguration, this recommendation is included.<br/><br/> Due to the importance of the admin interface, it is recommended to ensure that Envoy Proxies have not been accidentally misconfigured to expose the admin interface to untrusted networks.|
|EGTM-025 | EGTM-CS-011 | Container Security | The presence of a vulnerability, be it in the kernel or another system component, when coupled with containers running as root, could enable a threat actor to escape the container, thereby compromising the confidentiality, integrity, or availability of cluster resources. | The Envoy Proxy container's root-user configuration can be leveraged by an attacker to escalate privileges, execute a container breakout, and traverse across trust boundaries. | Low | By default, Envoy Gateway deployments do not use root users. Nonetheless, in case a custom image or deployment manifest is to be used, make sure Envoy Proxy pods run as a non-root user with a high UID within the container. Set runAsUser and runAsGroup security context options to specific UIDs (e.g., runAsUser: 1000 & runAsGroup: 3000) to ensure the container operates with the stipulated non-root user and group ID. If using helm chart deployment, define the user and group ID in the values.yaml file or via the command line during helm install / upgrade.|


## Attack Trees

Attack trees offer a methodical way of describing the security of systems, based on varying attack patterns. It's important to approach the review of attack trees from a top-down perspective. The top node, also known as the root node, symbolises the attacker's primary objective. This goal is then broken down into subsidiary aims, each reflecting a different strategy to attain the root objective. This deconstruction persists until reaching the lowest level objectives or 'leaf nodes', which depict attacks that can be directly launched. 

It is essential to note that attack trees presented here are speculative paths for potential exploitation. The Envoy Gateway project is in a continuous development cycle, and as the project evolves, new vulnerabilities may be exposed, or additional controls could be introduced. Therefore, the threats illustrated in the attack trees should be perceived as point-in-time reflections of the project’s current state at the time of writing this threat model. 

### Node ID Schema

Each node in the attack tree is assigned a unique identifier following the AT#-## schema. This allows easy reference to specific nodes in the attack trees throughout the threat model. The first part of the ID (AT#) signifies the attack tree number, while the second part (##) represents the node number within that tree.

### Logical Operators

Logical AND/OR operators are used to represent the relationship between parent and child nodes. An AND operator means that all child nodes must be achieved to satisfy the parent node. An OR operator between a parent node and its child nodes means that any of the child nodes can be achieved to satisfy the parent node.

### Attack Tree Node Legend

![AT Legend](/img/AT-legend.png)

### AT0

![AT0](/img/AT0.png)

### AT1

![AT1](/img/AT1.png)

### AT2

![AT2](/img/AT2.png)

### AT3

![AT3](/img/AT3.png)

### AT4

![AT4](/img/AT4.png)

### AT5

![AT5](/img/AT5.png)

### AT6

![AT6](/img/AT6.png)
