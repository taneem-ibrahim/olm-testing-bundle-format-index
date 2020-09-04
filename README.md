# Operator Lifecyle Manager (OLM) integration Tests

A simple OLM integraion testing operator for the [bundle format](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html/operators/olm-packaging-format#olm-bundle-format_olm-packaging-format) with [OPM](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html/operators/olm-packaging-format#olm-opm_olm-packaging-format)

**Bundle Format Operator Directory Structure**

```
foo-operator % tree
├── bundle
│   ├── manifests
│   │   ├── example.com_foobars.yaml
│   │   ├── foobar-operator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   └── foobar-operator.clusterserviceversion.yaml
│   ├── metadata
│   │   └── annotations.yaml
│   └── tests
│       └── scorecard
│           └── config.yaml
├── bundle.Dockerfile
├── catalogsource.yaml
└── subscription.yaml
```

**Let's build and push the operator bundle as an image**

```
- docker build -f bundle.Dockerfile -t quay.io/<quay_username>/foobar-operator:v0.0.1 .
- login to quay.io: docker login quay.io -u <quay_username>
- docker push quay.io/<quay_username>/foobar-operator:v0.0.1
- Login to quay.io web portal and set the repository to public (under repository settings). 
```

**Let's validate the remote bundle-format image with operator-sdk (v1.0.0+)**

```
- operator-sdk bundle validate quay.io/<quay_username>/foobar-operator:v0.0.1
- Note: if the validation is successful, then we should see only INFO level messages (no WARN or ERROR) in the output console and it should end with "All validation tests have completed successfully".
```

**Let's build and push a catalog index image for the operator**

```
- opm index add --bundles quay.io/<quay_username>/foobar-operator:v0.0.1 --tag quay.io/<quay_username>/foobar-operator-index:0.0.1 --build-tool docker
- docker push quay.io/<quay_username>/foobar-operator-index:0.0.1
- Login to quay.io on the web portal and set the repository to public.
```

**Create CatalogSource**

```
- oc create -f catalofsource.yaml
- Validate the pod is up and running: oc logs custom-<> -n openshift-marketplace

time="2020-09-02T23:10:08Z" level=info msg="Keeping server open for infinite seconds" database=/database/index.db port=50051
time="2020-09-02T23:10:08Z" level=info msg="serving registry" database=/database/index.db port=50051
```

**Create Subscription**


- Let's get the operator package name: 
```
oc get packagemanifests| grep foobar
> foobar-operator
```

- Let's get the default channel: 
```
oc get packagemanifests foobar-operator -o jsonpath='{.status.defaultChannel}'
> alpha
```

- Valide that our subscrption.yaml `spec:name` matches operator package name (ie. foobar-operator), and `spec:channel` matches alpha. Let's create the subscription resource.

```
- oc create -f subscription.yaml
```

**Install Operator from OperatorHub**

Now if we go to the OperatorHub in our local OpenShift cluster console, we can see our foobar operator listed. It can also be filtered by resource type "custom" on the left hand side. 

The bundle-format foobar-operator is now ready to be installed. We can validate the installation by checking the operator pod logs and looking for the echo message "v0.0.1":

```
oc logs -f foobar-operator-controller-manager-<> -n openshift-marketplace
> v0.0.1
```

**Upgrade the operator bundle version and corresponding index image pointer**

_coming soon_
