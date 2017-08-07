# APB Create and test playbook

This is a set of playbooks to replace the `apb` command line.

## Try it out

1. You'll need the `oc` client. You should authenticate against an OpenShift cluster.
1. Run a playbook. See below to figure out which one.

### init

This playbook bootstraps an application, writing files and directories. Use this playbook if you want to start your APB from scratch. It will not attempt to query OpenShift for any values.

```
ansible-playbook init.yml
```

**Optional Variables**

By providing some input data as extra vars (`-e` option) some customization can be achieved to save development time. See file `roles/init/defaults/main.yml` for the default values. These keys may be overwritten. For example:

```
ansible-playbook init.yml -e name=foo -e overwrite_existing_files=yes 
```

- `name`: application name

### snapshot

This playbook performs bootstrapping initialization by taking a snapshot of an application running in OpenShift. Use this playbook if you already have something running and want to create an APB based on that. It won't get you 100% there but it will save you a lot of time.

**Requirements / Dependencies**

- Kubernetes module

        ansible-galaxy install ansible.kubernetes-modules
- Authenticated `oc` client

        oc login -u <user>

**Run the playbook**

It will create files in a unique directory.

    ansible-playbook snapshot.yml

**Optional Variables**

See file `roles/snapshot/defaults/main.yml` for defaults. Example:

```
oc login -u <user>
ansible-playbook snapshot.yml -e name=mongodb -e template_name=mongodb-persistent  -e overwrite_existing_files=yes
```

- `openshift_objects`: comma-separated list of objects to snapshot passed in as a JSON object, e.g. `'["deploymentconfig", "service"]'`
- `namespace`: namespace to take snapshot from. Default: current namespace (`oc project -q`)
- `template_name`: name of template application was based on to get metadata from. Default: 'name' variable
- `template_namespace`: template namespace application was based on to get metadata from. Default: "openshift"

