# SonarQube Upgrade Guide from 9.9.5.1 to 2025.1.0 (Alauda Build of SonarQube Operator Version 2025.1.z)

## Overview

This document describes how to upgrade SonarQube from version 9.9.5.1(9.9.6) to 2025.1.0. Since this upgrade involves changes in operator 4.0 version, existing SonarQube instances need to be migrated to the new version operator.

::::tip Migration Duration
- Larger databases take longer to migrate.
- Storage performance also affects migration speed — using TopoLVM is recommended for better performance.

Example:
- SonarQube instance: 574 projects; PostgreSQL PVC usage 87 Gi; migration time ~1 hour
::::

## Prerequisites

- New version operator (sonarqube-ce-operator) has been installed in the cluster
- Ensure sufficient system resources are available for the upgrade
- Use product functionality to upgrade SonarQube to version 9.9.5.1
- Perform a complete data backup according to the [backup documentation](../howto/04_sonarqube_backup.md)

## Upgrade Steps

### 1. Stop the Old Version Instance Deployment

```bash
# Prevent operator from auto-syncing
kubectl patch deployment <old-sonarqube-name>-sonarqube -n <old-instance-namespace> \
  --type=json \
  -p '[{"op": "add", "path": "/metadata/annotations/skip-sync", "value": "true"}]'

# Scale the deployment down to 0
kubectl scale deployment <old-sonarqube-name>-sonarqube -n <old-instance-namespace> --replicas=0
```

### 2. Configure the New Version Instance

#### 2.1 Database Configuration

***Note: SonarQube 9.9.5 supports PG versions 11-15, while SonarQube 2025.1.0 supports PG versions 13-17. For data migration to a new PG version, refer to the official PG documentation***

The PG information used by the old SonarQube version is recorded in the spec.database.secretName of the SonarQube instance.

`kubectl get sonarqube.operator.devops.alauda.io <old-sonarqube-name> -n <old-instance-namespace> -o jsonpath='{.spec.database.secretName}'`

Use the above command to get the secretName, then use

`kubectl get secret <secretName> -n <old-instance-namespace>` -o yaml to get the secret information

The secret content is as follows:

```yaml
apiVersion: v1
data:
  database: <database base64>
  host: <host base64>
  password: <password base64>
  port: <port base64>
  sslmode: <sslmode base64>
  username: <username base64>
kind: Secret
metadata:
  name: old-sonarqube-pg-secret
type: Opaque
```

1. Perform a complete data backup according to the [backup documentation](../howto/04_sonarqube_backup.md)

2. Create a new PG database for the new SonarQube instance.
> It is recommended to create a new database for deploying the new SonarQube version to avoid contaminating the old database and ensure the old SonarQube can still function normally

3. Migrate the data according to the [backup documentation](../howto/04_sonarqube_backup.md)

4. Create a new database secret. The new version secret only needs to store the password:

    ```bash
    # Note: The password here is the password for the newly deployed PG database
    kubectl create secret -n <namespace> generic sonarqube-db-secret \
      --from-literal=jdbc-password=<password>
    ```

5. Create a new SonarQube instance configuration with database connection information, connecting to the previous PG instance

    ```yaml
    spec:
      helmValues:
        postgresql:
          enabled: false  # Disable default PostgreSQL
        jdbcOverwrite:
          enable: true
          # jdbcSecretName is the name of the newly created secret
          jdbcSecretName: sonarqube-db-secret
          # username, host, port, database are the username, host, port, database of the new PG data
          jdbcUrl: jdbc:postgresql://<host>:<port>/<database>?socketTimeout=1500
          jdbcUsername: <username>
    ```

#### 2.2 Storage Configuration

Choose the corresponding configuration based on the storage type used by the old SonarQube instance:

`kubectl get sonarqube.operator.devops.alauda.io <old-sonarqube-name> -n <old-instance-namespace> -o jsonpath='{.spec.persistence.type}'`

**Using StorageClass:**

Deploy the new instance using the existing PVC

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      ## The old version PVC name is <old-sonarqube-name>-sonarqube
      existingClaim: <old-sonarqube-name>-sonarqube
```

**Using Existing PVC:**

Use the command below to get the PVC name used by the old SonarQube version

`kubectl get sonarqube.operator.devops.alauda.io <old-sonarqube-name> -n <old-instance-namespace> -o jsonpath='{.spec.persistence.pvc.webServiceExistingClaim}'`

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      existingClaim: <spec.persistence.pvc.webServiceExistingClaim from old instance YAML>
```

**Using HostPath:**

**Note**: The node for the HostPath of the new and old versions must remain consistent. The `path` should add a `portal` directory based on the old version path. For example, if the old version path is /sonarqube/, the new version path should be /sonarqube/portal/

```yaml
spec:
  helmValues:
    nodeSelector:
      kubernetes.io/hostname: <node-name>
    persistence:
      enabled: false
      host:
        nodeName: <spec.persistence.location.nodeName from old instance YAML>
        path: <spec.persistence.location.path from old instance YAML>/portal # The old version added a portal directory in the controller, the new version needs to be consistent
```

#### 2.3 Network Configuration

Since the old version instance has not been cleaned up, the new version network configuration cannot be the same as the old version (**i.e., ingress cannot use the same domain name, nodePort cannot use the same port**), otherwise it will cause conflicts. Temporarily use other addresses, and after the upgrade is complete, clean up the old version instance, then modify the access address.

Use the command below to get the network type used by the old SonarQube version

`kubectl get sonarqube.operator.devops.alauda.io <old-sonarqube-name> -n <old-instance-namespace> -o jsonpath='{.spec.service.type}'`

**Ingress Configuration:**

```yaml
spec:
  helmValues:
    ingress:
      enabled: true
      hosts:
        - name: <spec.service.ingress.domainName from old instance YAML> ## Note: When the old instance exists, using the same domain name for the new instance will cause conflicts, so set a different domain name
      tls: ## If the old version is https, then tls needs to be configured, otherwise do not configure the following part
        - secretName: <spec.service.ingress.secretName from old instance YAML> ## The old version certificate may be under cpaas-system, it needs to be copied to the namespace where the instance is deployed
          hosts:
            - <spec.service.ingress.domainName from old instance YAML>
```

**NodePort Configuration:**

```yaml
spec:
  helmValues:
    service:
      name: sonarqube
      type: NodePort
      nodePort: <spec.service.nodePort.port from old instance YAML> ## Note: When the old instance exists, using the same nodePort for the new instance will cause conflicts, so set a different nodePort
```

**Configure spec.externalURL:**

```yaml
spec:
  ## If the network configuration is Ingress type, the value of <new-sonar-url> is <spec.service.ingress.domainName from old instance YAML>
  ## If the network configuration is NodePort type, get the IP address using the command kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'. The value of <new-sonar-url> is IP:<nodePort>
  externalURL: <new-sonar-url>
```

#### 2.4 Other Configurations

- **Retain Plugins**: The new version needs to retain the old version plugins. Add configuration spec.helmValues.plugins.deleteDefaultPlugins as false
- SSO Configuration Migration: Reconfigure according to [Deployment Document: SSO Configuration](../install/03_sonarqube_deploy.md#SSO-Configuration)
- Resources Configuration Migration: Configure the new instance's spec.helmValues.resources based on the old instance's spec.resourceAssign
- helmValues Configuration Migration: Most of the data in the old instance's helmValues can be directly migrated to the new instance's helmValues. For example, spec.helmValues.jvmOpts can be directly migrated to the new instance's spec.helmValues.jvmOpts.

Complete YAML example for the new version:

```yaml
apiVersion: operator.alaudadevops.io/v1alpha1
kind: Sonarqube
metadata:
  name: new-sonarqube
spec:
  helmValues:
    plugins:
      deleteDefaultPlugins: false # Retain old version plugins
    prometheusExporter: # Disable default prometheus monitoring, jar packages need to be added in advance when starting
      enabled: false
    resources: # Set resource limits, from spec.resourceAssign in old instance YAML
      limits:
       cpu: 800m
       memory: 4Gi
      requests:
       cpu: 400m
       memory: 2Gi
    postgresql: # Disable default PostgreSQL instance
      enabled: false
    jdbcOverwrite: # Configure using PG information from old version instance
      enable: true
      jdbcSecretName: sonarqube-db-secret # Use the newly created secret name
      jdbcUrl: jdbc:postgresql://<pg.host>:<pg.port>/<pg.database>?socketTimeout=1500
      jdbcUsername: <pg.username>
    service: # Configure based on old version instance
      name: sonarqube
      type: NodePort
      nodePort: <spec.service.nodePort.port from old instance YAML>
    persistence: # Configure based on old version instance
      enabled: true
      existingClaim: <spec.persistence.pvc.webServiceExistingClaim from old instance YAML>
```

### 3. Manually Perform Data Schema Migration

:::warning

1. Backup the old version data in advance.
2. The instance will be ready only after completing the setup.

:::

Access new-sonar-url/setup and follow the prompts

### 4. Verify Upgrade Results

Check the following to ensure the upgrade was successful:

- [ ] All project data is complete
- [ ] User accounts and permissions are correct
- [ ] Plugin status is normal (some plugins may need to be reinstalled)
- [ ] Historical analysis data is accessible

### 5. Cleanup Work

1. After confirming the new instance is running stably, the old instance can be deleted
2. Adjust the network configuration to be consistent with the original configuration

## Reference Documents

- [SonarQube Backup Documentation](../howto/04_sonarqube_backup.md)
- [SonarQube Deployment Documentation](../install/03_sonarqube_deploy.md)
