# Openshift laboratory provisioner tool
This tool was designed to provision and configure Openshift Container Platform
to being used as labs or for testing purposes automatically and fast using
cloud-provided infrastructure.

## Setting up environment
Before to use this repo, some configurations should be prepared. Check the
current cloud compatibility to ensure yourself your cloud provider it's
supported by this tool.

### Check Cloud provider compatibility
This table represents the feature compatibility with each cloud provider
| Cloud Provider / Feature | Install | Config | Multi-Credential management | Bastionning |
|--------------------------|---------|--------|-----------------------------|-------------|
| AWS                      | Yes     | Yes    | Yes                         | No          |
| GCP                      | No      | No     | No                          | No          |
| Azure                    | No      | No     | No                          | No          |
| Bare Metal               | No      | No     | No                          | No          |


## Configuring
### AWS
For AWS it's needed to create an `Ansible Vault` secret which contains the
credentials of a nominal user or a service account which has enough permissions
to deploy `EC2`, `VPC`, `S3 Buckets` and `Route53` entries. The file must contain the following parameters:
```yaml
aws_public_domain: <PUBLIC_DOMAIN_ROUTE53_SERVICE>
aws_account_name: <ACCOUNT_NAME>
aws_access_key_id: <SA_ACCESS_KEY>
aws_secret_access_key: <SA_SECRET_ACCESS_KEY>
```

To easily decrypt the secrets, you could create a plain text file with the
password of the `Ansible Vault` file. In this case, remember to not push the
password file!

**NOTE:** Store those credential files in the `secrets` folder on this
repository (Files in that folder will not be pushed).

## Configure Cluster Deployment
To provision and configure the cluster, it's needed to fill a YAML file which
contains every required parameter. The following example shows how to configure
it:
```yaml
## Red Hat Intel Openshift Lab config tool
#################################################################################
## vim: noai:ts=2:sw=2

---
cluster:
  name: <CLUSTER_NAME>         # DNS compatible
  version: <CLUSTER_VERSION>   # X.Y.Z

  # Cloud provider config
  cloud:
    aws:
      profile: # Values defined in ansible vault secret (MANDATORY)
        aws_account_name: "{{ aws_account_name }}"
        aws_access_key_id: "{{ aws_access_key_id }}"
        aws_secret_access_key: "{{ aws_secret_access_key }}"


  # config_template.yaml inception
  config:
    name: laboratory-test
    #kubeconfig: "./auth/kubeconfig"    # Optional

    ## Openshift Image Registry Configuration
    registry:
      expose: true
      hostname: "openshift-registry" # Only type route prefix for *.apps domain. Cluster domain will be added automatically

    auth:
      provider:
        htpasswd:
          name: "htpasswd"
      users:
        - name: admin
          pass: admin
          group: admins
        - name: dev
          pass: dev
          group: developers
      groups:
        - name: admins
          clusterRole: cluster-admin
        - name: developers
          clusterRole: basic-user


  # install-config.yaml inception
  spec:
    apiVersion: v1
    baseDomain: "{{ aws_public_domain }}"

    metadata:
      name: <CLUSTER_NAME> # DNS compatible

    controlPlane:
      name: master
      replicas: 3
      platform:
        aws:
          type: <MASTER_VM_FLAVOUR>
          zones:
            - <AVAILABILITY_ZONE>

    compute:
      - name: worker
        replicas: <NUMBER_OF_WORKERS>
        platform:
          aws:
            type: <WORKER_VM_FLAVOUR>
            zones:
              - <AVAILABILITY_ZONE>

    networking:
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      machineNetwork:
      - cidr: 10.0.0.0/16
      networkType: OpenShiftSDN
      serviceNetwork:
      - 172.30.0.0/16

    platform:
      aws:
        region: <REGION>

    sshKey: '...'
    pullSecret: '...'
```



## Create Environments
One you had configured your environment and your cloud provider credentials, run
the following command to start the provisioning process.
```sh
ansible-playbook site.yaml \
  --vault-password-file=<VAULT_PASSWORD_FILE> \
  -e@secrets/<CLOUD_CREDENTIALS_FILE> \
  --extra-vars="deployment=<LAB_CONFIG_FILE>"
```
**WARNING:** Don't forget to backup `build` dir to keep environments tracked.

## Destroy Environments
```sh
ansible-playbook clean.yaml \
  --vault-password-file=<VAULT_PASSWORD_FILE> \
  -e@secrets/<CLOUD_CREDENTIALS_FILE> \
  --extra-vars="deployment=<LAB_CONFIG_FILE>.yaml"
```

