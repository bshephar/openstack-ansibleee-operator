# OpenStack Ansible EE operator

An operator to deploy and run an OpenStack Ansible Execution Environment container on Openshift

## Build and deploy

It uses operator-sdk to build and run.

To build and push to a container registry

```bash
make docker-build docker-push IMG="<your image name>"
```

To deploy in to the cluster

```bash
make deploy IMG="<your image name>"
```

To undeploy it from the cluster

```bash
make undeploy
```

## Use

Once the operator has been deployed succesfully to the openshift/kubernetes cluster, you can see it in action by creating a new "ansibleee" CR.

There are some examples on the examples directory.

The first one is openstack-ansibleee-playbook-local.yaml. This wil execute locally an example playbook, provided inline, which will print an "hello world" message using ansible debug module.

```bash
oc apply -f examples/openstack-ansibleee-playbook-local.yaml
```

There are other examples that also execute locally the playbook "test.yaml", but that serve as extraMounts demonstration: openstack-ansibleee-extravolumes.yaml and openstack-ansibleee-extravolumes_2_secret.yaml that need the secrets ceph-secret-example.yaml and ceph-secret-example2.yaml created:

```bash
oc apply -f ceph-secret-example.yaml
oc apply -f ceph-secret-example2.yaml
oc apply -f examples/openstack-ansibleee-extravolumes.yaml
```

There are also a number of examples that feature remote execution. By default, all of them expect a compute node to be available in 10.0.0.4, adjust the inventory accordingly for your environment.

The first remote example is openstack-ansibleee-playbook.yaml. This runs one of the standalone playbooks that is included in the default image.

To access an external node, you need to provide the ssh private key so ansible can connect to the node. This is being expected to be provided by a "ssh-key-secret" Secret with this format:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key-secret
  namespace: openstack
data:
  ssh-privatekey:  3390 bytes                                                                                       │
  ssh-publickey:   750 bytes
```

Once the key has been created, the CR should run the deploy-edpm-os-configure.yml playbook on the external node:

```bash
oc apply -f examples/openstack-ansibleee-playbook.yaml
```

The second remote example is openstack-ansibleee-role.yaml, which will run a certain number of tasks from specific standalone roles:

```bash
oc apply -f examples/openstack-ansibleee-role.yaml
```

And the last remote example is ansibleee-play.yaml, which will run a CR-defined playbook using an inventory stored in a ConfigMap.

```bash
oc apply -f examples/inventory-configmap.yaml
oc apply -f examples/openstack-ansibleee-play.yaml
```

## Example Development Cycle

The following has been verified on
[openshift-local](https://developers.redhat.com/products/openshift-local/overview).

Create the CRD managed by the operator. This must be deleted and re-created any time the api changes.

```bash
oc create -f config/crd/bases/ansibleee.openstack.org_openstackansibleees.yaml
```

Build and run a local copy of the OpenStack Ansible Execution Environment operator.

```bash
make generate
make manifests
make build
./bin/manager
```

Once the operator is running, create the examle CR to run the test playbook.

```bash
oc create -f examples/openstack-ansibleee-playbook-local.yaml
```

The operator will create an OpenStackAnsibleEE resource which will create a job. That job will spawn a pod using an ansible runner that will execute the given playbook. When the playbook is done, the pod will move to a `Completed` state.

You can get the OpenStackAnsibleEE resource created with the `oc get openstackansibleee` (or `osaee`) command.

```bash
$ oc get osaee
NAME                       NETWORKATTACHMENTS   STATUS   MESSAGE
ansibleee-playbook-local                        True     Job completed
```

After knowing the name of the OpenStackAnsibleEE resource, you can search for its pod by filtering the `openstackansibleee_cr` label.

```bash
$ oc get pods -l openstackansibleee_cr=ansibleee-playbook-local
NAME                             READY   STATUS      RESTARTS   AGE
ansibleee-playbook-local-lbl6c   0/1     Completed   0          44s
```

To see the result of the playbook run, use `oc logs`.

```bash
oc logs -l openstackansibleee_cr=ansibleee-playbook-local
No config file found; using defaults

PLAY [Print hello world] *******************************************************

TASK [Gathering Facts] *********************************************************
Wednesday 12 July 2023  15:43:22 +0000 (0:00:00.022)       0:00:00.022 ********
ok: [localhost]

TASK [Using debug statement] ***************************************************
Wednesday 12 July 2023  15:43:23 +0000 (0:00:01.320)       0:00:01.342 ********
ok: [localhost] => {
    "msg": "Hello, world this is openstack-ansibleee-play.yaml"
}

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
Wednesday 12 July 2023  15:43:24 +0000 (0:00:00.077)       0:00:01.420 ********
===============================================================================
Gathering Facts --------------------------------------------------------- 1.32s
Using debug statement --------------------------------------------------- 0.08s
```

## Using openstack-ansibleee-operator with EDPM Ansible

When the openstack-ansibleee-operator spawns a job ansible execution environment crafted image
can use playbooks and roles contained in its image.

An openstack-ansibleee-runner image is hosted at
[quay.io/openstack-k8s-operators/openstack-ansibleee-runner](https://quay.io/openstack-k8s-operators/openstack-ansibleee-runner)
which contains [edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible).
The following commands may be used to inspect the content.

```bash
podman pull quay.io/openstack-k8s-operators/openstack-ansibleee-runner:latest
IMAGE_ID=$(podman images --filter reference=openstack-ansibleee-runner:latest --format "{{.Id}}")
podman run $IMAGE_ID ls -l
```

The container is built by a github action from a [Dockerfile](https://github.com/openstack-k8s-operators/edpm-ansible/blob/main/openstack_ansibleee/Dockerfile) in the [edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible) repository.
