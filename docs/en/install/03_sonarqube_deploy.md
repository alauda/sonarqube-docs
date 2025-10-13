---
title: SonarQube Instance Deployment
description: >-
  SonarQube is a code quality management tool that can quickly identify potential or obvious errors in code.
weight: 200
---

This document describes the subscription of SonarQube Operator and the functionality for deploying SonarQube instances.

## Prerequisites

- This document applies to SonarQube 9.9.5 and above versions provided by the platform. It is decoupled from the platform using technologies such as Operator.

- Please ensure that the SonarQube Operator has been deployed (subscribed) to the target cluster, meaning that creating instances from the SonarQube Operator is ready.

## Deployment Planning

SonarQube supports various resource configurations to accommodate different customer scenarios. In different scenarios, the required resources, configurations, etc., will have significant differences. Therefore, this section introduces which aspects need to be considered in deployment planning before deploying a SonarQube instance, and what the impact of decision points is, to facilitate users in making subsequent specific instance deployments based on this information.

### Basic Information

1. The SonarQube Operator provided by the platform is based on the community's official SonarQube Chart, with enhanced enterprise capabilities such as IPv6 support and security vulnerability fixes. It is fully compatible with the community version in terms of functionality, and in terms of user experience, it enhances the convenience of SonarQube deployment through optional, customizable templates and other methods.

### Pre-deployment Resource Planning

Pre-deployment resource planning involves making decisions before deployment that will take effect during the deployment process.

For more recommendations on environmental resources, please refer to the official documentation: https://docs.sonarsource.com/sonarqube-server/2025.1/setup-and-upgrade/installation-requirements/server-host/#hardware

## Instance Deployment

### Deploying from the `Quickstart Template` Template

This template is used to quickly create a lightweight SonarQube instance, suitable for development and testing scenarios, not recommended for production environments.

- Compute resources: 800m CPU, 4 Gi memory
- Storage: Use node local storage, configure the storage node IP and path
- Network access: Use NodePort to access the service, share the node IP with storage, and specify the port

### Deploying from the `Production Template` Template

This template is used to quickly create a Production SonarQube instance, suitable for production scenarios, recommended for production environments.

- Compute resources: 8 CPU cores, 16 Gi memory
- Storage: Use PVC storage, configure the storage class
- Network access: Use Domain to access the service.

### Deploying from YAML

#### Resource Configuration

SonarQube only needs to configure the overall resources, for example:

```yaml
spec:
  helmValues:
    resources:
      limits:
        cpu: 800m
        memory: 4Gi
      requests:
        cpu: 400m
        memory: 2Gi
```

For more information, refer to [Resource description in SonarQube Chart](https://github.com/SonarSource/helm-chart-sonarqube/blob/sonarqube-2025.1.0-sonarqube-dce-2025.1.0/charts/sonarqube/values.yaml#L454)

#### Network Configuration

Network configurations are categorized into two types:

- Network configuration based on ingress
- Network configuration based on NodePort

Network configuration based on ingress supports both https and http protocols. An ingress controller needs to be deployed in the cluster in advance.

```yaml
spec:
  helmValues:
    ingress:
      enabled: true
      hosts:
        - name: test-ingress-https.example.com
      tls:
        - secretName: test-tls-cert
          hosts:
            - test-ingress-http.example.com
```

Network configuration based on NodePort:

```yaml
spec:
  helmValues:
    service:
      name: sonarqube
      type: NodePort
      nodePort: <nodeport.http>
```

#### Storage Configuration

Storage configurations are mainly divided into three categories:

- Storage configuration based on StorageClass
- Storage configuration based on PVC
- Storage configuration based on HostPath

Storage configuration based on StorageClass:

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      storageClass: <storage-class.rwm> ## StorageClass needs to be created in advance
      size: 10Gi
```

Storage configuration based on PVC:

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      existingClaim: sonarqube-pvc ## PVC needs to be created in advance
```

Storage configuration based on HostPath:

```yaml
spec:
  helmValues:
    nodeSelector:
      kubernetes.io/hostname: <node.random>  ## Select the deployment node
    
    persistence:
      enabled: false
      host:
        nodeName: <node.random> ## Name of the node
        path: /tmp/sonarqube-<template.{{randAlphaNum 10}}> ## Select the deployment node and specify the storage path
```

#### PostgreSQL Access Credentials Configuration

A PostgreSQL instance needs to be created in advance on the platform, and a database needs to be created in PostgreSQL for use.

The supported PostgreSQL versions for SonarQube 25.1.0 are 13 to 17.

PostgreSQL access is accomplished by configuring a secret resource with specific format content. See [Configuring PostgreSQL and Account Access Credentials](./02_sonarqube_credential) for details.

Using a secret for the credentials to access PG in SonarQube yaml:

```yaml
spec:
  helmValues:
    postgresql:
      enabled: false ## Disable the default PostgreSQL instance
    jdbcOverwrite:
      enable: true
      jdbcSecretName: postgres-password ## Credentials for connecting to PG
      jdbcUrl: jdbc:postgresql://<pg.host>:<pg.port>/<pg.database>?socketTimeout=1500 ## Address for connecting to PG
      jdbcUsername: postgres ## PG access user
```

#### Admin Account Configuration

When initializing a SonarQube instance, you need to configure the admin account and its password. This is done by configuring a secret resource. See [Configuring PostgreSQL, and Account Access Credentials](./02_sonarqube_credential) for details.

Specify it to SonarQube through YAML:

```yaml
spec:
  helmValues:
    account:
      adminPasswordSecretName: sonarqube-root-password
```

### Complete YAML Example

NodePort, PVC, PostgreSQL, Admin account

```yaml
apiVersion: operator.alaudadevops.io/v1alpha1
kind: Sonarqube
metadata:
  name: sonarqube-demo
spec:
  helmValues:
    prometheusExporter: # Disable the default prometheus monitoring, jar packages need to be added in advance when starting
      enabled: false
    resources: # Set resource limits
      limits:
       cpu: 800m
       memory: 4Gi
      requests:
       cpu: 400m
       memory: 2Gi
    postgresql: # Disable the default PostgreSQL instance
      enabled: false
    jdbcOverwrite: # Use a pre-created PostgreSQL instance
      enable: true
      jdbcSecretName: postgres-password
      jdbcUrl: jdbc:postgresql://<pg.host>:<pg.port>/<pg.database>?socketTimeout=1500
      jdbcUsername: <pg.username>
    service: # Use NodePort to expose the SonarQube instance
      name: sonarqube
      type: NodePort
      nodePort: <nodeport.http>
    account: # Set Root password
      adminPasswordSecretName: sonarqube-root-password
    persistence: # Use PVC for storage mounting
      enabled: true
      existingClaim: sonarqube-pvc
```

### SSO Configuration

Configuring SSO involves the following steps:

Register an SSO authentication client in the global cluster

- Create the following OAuth2Client resource in the global cluster to register the SSO authentication client.
- Configure the SonarQube instance to use SSO authentication

<!-- lint ignore code-block-split-list -->

```yaml
apiVersion: dex.coreos.com/v1
kind: OAuth2Client
name: OIDC
metadata:
  name: m5uxi3dbmiwwizlyzpzjzzeeeirsk # This value is calculated based on the hash of the id field, online calculator: https://go.dev/play/p/QsoqUohsKok
  namespace: cpaas-system
id: sonarqube-dex # Client id
public: false
redirectURIs:
  - <sonarqube-host>/* # SonarQube authentication callback address, where <sonarqube-host> is replaced with the access address of the SonarQube instance
secret: Z2l0bGFiLW9mZmljaWFsLTAK # Client secret
spec: {}
```

Add the SSO configuration to the SonarQube instance:

```yaml
spec:
  helmValues:
    sonarProperties:
      sonar.core.serverBaseURL: <sonarqube url>
      sonar.forceAuthentication: false
      sonar.auth.oidc.enabled: true
      sonar.auth.oidc.issuerUri: <platform-url>/dex
      sonar.auth.oidc.clientId.secured: sonarqube-dex # Client id
      sonar.auth.oidc.clientSecret.secured: Z2l0bGFiLW9mZmljaWFsLTAK  # Client secret
      sonar.auth.oidc.loginStrategy: Preferred username
      sonar.auth.oidc.providerConfiguration: '{"issuer":"<platform-url>/dex","authorization_endpoint":"<platform-url>/dex/auth","token_endpoint":"<platform-url>/dex/token","jwks_uri":"<platform-url>/dex/keys","response_types_supported":["code","id_token","token"],"subject_types_supported":["public"],"id_token_signing_alg_values_supported":["RS256"],"scopes_supported":["openid","email","groups","profile","offline_access"],"token_endpoint_auth_methods_supported":["client_secret_basic"],"claims_supported":["aud","email","email_verified","exp","iat","iss","locale","name","sub"]}'
```

Enabling SSO for Platforms Using Self-Signed Certificates

If the platform is accessed via https and uses a self-signed certificate, you need to mount the CA of the self-signed certificate to the SonarQube instance. The method is as follows:

In the cpaas-system namespace of the global cluster, find the secret named dex.tls, get the ca.crt and tls.crt content from the secret, save it as a new secret, and create it in the namespace of the SonarQube instance.

```yaml
apiVersion: v1
data:
  ca.crt: <base64 encode data>
  tls.crt: <base64 encode data>
kind: Secret
metadata:
  name: dex-tls
  namespace: cpaas-system
```

Edit the SonarQube instance to use this CA:

```yaml
spec:
  helmValues:
    caCerts:
      enabled: true
      secret: dex-tls
```

#### Using in Pure IPv6 Clusters

When deploying in a pure IPv6 cluster environment, you need to explicitly configure IPv6 protocol settings since Java supports dual-stack by default. Add the following configuration to ensure proper connectivity:

```yaml
spec:
  helmValues:
    env:
      - name: JAVA_TOOL_OPTIONS
        value: '-Djava.net.preferIPv6Addresses=true'
    sonarProperties:
      sonar.cluster.node.search.host: '[::1]'
      sonar.cluster.node.es.host: '[::1]'
      sonar.web.javaAdditionalOpts: '-javaagent:/opt/sonarqube/extensions/plugins/sonarqube-community-branch-plugin-1.23.1.jar=web -Djava.net.preferIPv6Addresses=true'
```

## Additional Information

### Kubernetes - Pod Security Standards

The following section describes the compatibility of SonarQube containers with [Kubernetes Pod Security levels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-levels):

By default, SonarQube deployment requires `privileged` permissions. However, when `initSysctl.enabled` is set to `false`, it can run in `baseline` mode.

> **Important Note**: When using node-local storage, deployment is only supported in `privileged` mode.

When `initSysctl.enabled` is set to `false`, the Kubernetes administrator must configure the sysctl-related values at the node level.

For host configuration, refer to [Configuring the host to comply with Elasticsearch](https://docs.sonarsource.com/sonarqube-server/2025.1/setup-and-upgrade/pre-installation/linux/#Elasticsearch). Configuration steps:

```bash
# Configure memory mapping area limit
sysctl -w vm.max_map_count=524288
sysctl -n vm.max_map_count

# Configure system file descriptor limit
sysctl -w fs.file-max=131072
sysctl -n fs.file-max

# Configure process file descriptor limit
ulimit -n 131072
ulimit -n

# Configure process limit
ulimit -u 8192
ulimit -u
```

When `initSysctl.enabled` is set to `true` (which is the default value), the above sysctl configurations are not required.
