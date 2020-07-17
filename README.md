# Building Synopsys operator using the Operator SDK toolkit

## Setup and Requirements

- operator-sdk

Get the operator sdk version 0.19.0 [here]()
And check the install instructions [here]()

- Helm Charts

Create a folder and copy the blackduck, and the blackduck-connector helm charts there. We chose these two for testing purposes.

## Creating the Operator Project

The project is completely generated by the operator sdk tool. The `operator-sdk new` command creates a new operator application and generates a default directory layout based on the input.

From the root of the folder where you have put the helm charts, run the following commands:

`operator-sdk new synopsys-operator --api-version=blackduck.synopsys.com/v1alpha1 --kind Blackduck --type helm --helm-chart ../blackduck`

You should see something like below:
```
INFO[0000] Creating new Helm operator 'synopsys-operator'.
INFO[0000] Created helm-charts/blackduck
INFO[0000] Generating RBAC rules
I0717 09:22:05.476349   84093 request.go:621] Throttling request took 1.026366842s, request: GET:https://api.alex-test.coreostrain.me:6443/apis/config.openshift.io/v1?timeout=32s
WARN[0002] The RBAC rules generated in deploy/role.yaml are based on the chart's default manifest. Some rules may be missing for resources that are only enabled with custom values, and some existing rules may be overly broad. Double check the rules generated in deploy/role.yaml to ensure they meet the operator's permission requirements.
INFO[0002] Created build/Dockerfile
INFO[0002] Created deploy/service_account.yaml
INFO[0002] Created deploy/role.yaml
INFO[0002] Created deploy/role_binding.yaml
INFO[0002] Created deploy/operator.yaml
INFO[0002] Created deploy/crds/blackduck.synopsys.com_v1alpha1_blackduck_cr.yaml
INFO[0002] Generated CustomResourceDefinition manifests.
INFO[0002] Project creation complete.
```
  Note: There is a warning about the RBAC rules. The automation tries to guess what are the permissions needed by the application based on what's present in the helm chart. Adjustments will be necessary in order to better secure your operator deployment and the platform it's running on. 

And finally you have what is the actual operator project that will be stored in your git repo. 
Let's enter that folder and check it:

```
$ cd synopsys-operator
$ tree -L 3

.
├── build
│   └── Dockerfile
├── deploy
│   ├── crds
│   │   ├── blackduck.synopsys.com_blackducks_crd.yaml
│   │   └── blackduck.synopsys.com_v1alpha1_blackduck_cr.yaml
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── helm-charts
│   └── blackduck
│       ├── Chart.yaml
│       ├── README.md
│       ├── bd.yaml
│       ├── external-postgres-init.pgsql
│       ├── large.yaml
│       ├── medium.yaml
│       ├── small.yaml
│       ├── templates
│       ├── values.yaml
│       └── x-large.yaml
└── watches.yaml
```
This is a pretty simple folder:

`build`

This is where the `operator-sdk build` will look for the Dockerfile to build the container image after compiling the code.

`deploy`

Here we have all the necessary artifacts to deploy the operator. The CRDs it's managing and the RBAC yamls as well as the deployment itself.

`helm-charts`

Note that the blackkduck helm chart was copied here. From that chart the operator will be able to deploy all the secondary resources.

`watches.yaml`

Finally we have the most important part that links the embedded operator controller to the deployment. Helm charts will be listed here in order to be watched by the operator controller. And every a new helm chart is added to the project using `operator-sdk add api` it will be appended to this file in order to be reconciled by that controller.



## Adding a New Helm Chart to the Operator Project 

#### Example: adding the blackduck-connector api using the blackduck-connector helm chart

`operator-sdk add api --api-version=blackduck-connector.synopsys.com/v1alpha1 --kind=BlackduckConnector --helm-chart ../blackduck-connector`

At this point you may have seen some warnings and specifically errors in the RBAC rules it tries to generate. So some adjustments may need to be done to the roles and rolebindings to fix the permissions. Not everything can be merged when using two or more helm charts at the same time.

```
FATA[0002] failed to merge rules in the RBAC manifest for resource (blackduck-connector.synopsys.com/v1alpha1, BlackduckConnector): cannot Merge Cluster scoped rules with existing deploy/role.yaml. please modify existing deploy/role.yaml and deploy/role_binding.yaml to reflect Cluster scope and try again
```

But it actually creates the new CRD and put this helm in the watches file. Let's check:
```
.
├── build
│   └── Dockerfile
├── deploy
│   ├── crds
│   │   ├── blackduck-connector.synopsys.com_blackduckconnectors_crd.yaml
│   │   ├── blackduck-connector.synopsys.com_v1alpha1_blackduckconnector_cr.yaml
│   │   ├── blackduck.synopsys.com_blackducks_crd.yaml
│   │   └── blackduck.synopsys.com_v1alpha1_blackduck_cr.yaml
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── helm-charts
│   ├── blackduck
│   │   ├── Chart.yaml
│   │   ├── README.md
│   │   ├── bd.yaml
│   │   ├── external-postgres-init.pgsql
│   │   ├── large.yaml
│   │   ├── medium.yaml
│   │   ├── small.yaml
│   │   ├── templates
│   │   ├── values.yaml
│   │   └── x-large.yaml
│   └── blackduck-connector
│       ├── Chart.yaml
│       ├── README.md
│       ├── blackduck-connector
│       ├── templates
│       └── values.yaml
└── watches.yaml

```
Now we have a second helm chart and a second CRD. And also a new one on the watches.yaml file:

```
$ cat watches.yaml
---
- group: blackduck.synopsys.com
  version: v1alpha1
  kind: Blackduck
  chart: helm-charts/blackduck
- group: blackduck-connector.synopsys.com
  version: v1alpha1
  kind: BlackduckConnector
  chart: helm-charts/blackduck-connector
```

This gives us a multiple helm chart operator project folder. And testing may begin.

First thing to use the local testing feature of operator sdk is to create the CRDs:

```
oc apply -f deploy/crds/blackduck.synopsys.com_blackducks_crd.yaml
oc apply -f deploy/crds/blackduck-connector.synopsys.com_blackduckconnectors_crd.yaml
```

And then you can grab a terminal window and run if you create a namespaces synopsys first:

`operator-sdk run local --watch-namespace synopsys`

Finally the example CR can be used as a test bed by running the command below in another terminal window:

`oc apply -f deploy/crds/blackduck.synopsys.com_v1alpha1_blackduck_cr.yaml`

    Note: here the CR needs to reflect what is in your preferred or default configuration. It's just a copy of what the operator derived from the helm chart.

After applying that you should be able to run:

```
$ oc get blackduck
NAME                AGE
example-blackduck   10m
```

## Generate the operator metadata bundle for the Operator Lifecycle Manager

`operator-sdk generate bundle  -version 0.1.0`

You should get an interactive wizard like below:
```
operator-sdk generate bundle  --version 0.1.0
INFO[0000] Generating bundle manifests version 0.1.0

Display name for the operator (required):
> synopsys

Description for the operator (required):
> deploys synopsys applications

Provider's name for the operator (required):
> synopsys

Any relevant URL for the provider name (optional):
> synopsys.example.com

Comma-separated list of keywords for your operator (required):
> synopsys,blackduck

Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required):
> gautam:example@email.com
INFO[0067] Bundle manifests generated successfully in deploy/olm-catalog/synopsys-operator
INFO[0067] Building annotations.yaml
INFO[0067] Writing annotations.yaml in /Users/alex/go/src/github.com/acmenezes/test/synopsys/synopsys-operator/deploy/olm-catalog/synopsys-operator/metadata
INFO[0067] Building Dockerfile
INFO[0067] Writing bundle.Dockerfile in /Users/alex/go/src/github.com/acmenezes/test/synopsys/synopsys-operator
```
And finally have a good kick start in the new olm-catalog folder with the generated bundle package. 

```
$ tree -L 4
├── build
│   └── Dockerfile
├── bundle.Dockerfile
├── deploy
│   ├── crds
│   │   ├── blackduck-connector.synopsys.com_blackduckconnectors_crd.yaml
│   │   ├── blackduck-connector.synopsys.com_v1alpha1_blackduckconnector_cr.yaml
│   │   ├── blackduck.synopsys.com_blackducks_crd.yaml
│   │   └── blackduck.synopsys.com_v1alpha1_blackduck_cr.yaml
│   ├── olm-catalog
│   │   └── synopsys-operator
│   │       ├── manifests
│   │       └── metadata

...
```
This command can be ran to upgrade the version and then it's just a matter to copy this folder into the community-operators repo and set a PR to publish it.