# Openshift laboratory provisioner tool

## Create Environments
```sh
ansible-playbook site.yaml --extra-vars="config=config_template.yaml"
```
**WARNING:** Don't forget to backup `build` dir to keep environments tracked.

## Destroy Environments
```sh
ansible-playbook clean.yaml --extra-vars="config=config_template.yaml"
```

