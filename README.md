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
- opm index add --bundles quay.io/<quay_username>/foobar-operator:v0.0.1 --tag quay.io/<quay_username>/foobar-operator-index:latest --build-tool docker
- docker push quay.io/<quay_username>/foobar-operator-index:latest
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

Upgrade the operator version

We will do a simple operator upgrade test just by updating the operator version tag from “0.0.1” to “0.0.2” in the Catalog Service Version (CSV) file. Let’s run the following commands from the root directory of our foo-operator:

```
> sed -i 's/0.0.1/0.0.2/g' ./bundle/manifests/foobar-operator.clusterserviceversion.yaml
```

Next we will just repeat the step numbers 2 and 3 above to build and validate the new operator bundle image and substitute the image version tags to be 0.0.2 instead of 0.0.1 in the docker push commands.

We are now ready to add the new operator version to the registry. We can do that via the `opm add` command. Notice how we are adding the upgraded operator version cumulatively by inserting the “from-index” parameter. Also note that we tagged the new index image with the “latest” tag. This will help us to update the catalog source to point to the latest version of our index image automatically.

```
opm index add --bundles quay.io/<quay_username>/foobar-operator:v0.0.2 --from-index quay.io/<quay_username>/foobar-operator-index:latest --tag quay.io/<quay_username>/foobar-operator-index:latest --build-tool docker
```

We will push this new version of the index image to quay:

```
docker push quay.io/$quay_username/foobar-operator-index:latest
```

The catalog source automatically polls for the latest version every 5 minutes. So after 5 minutes have passed, we can go to the operator hub console and validate the operator version 0.0.2 is available and then install it to the operator-marketplace namespace.

```
> oc logs -f foobar-operator-controller-manager-<pod_id> -n openshift-marketplace
v0.0.2
```

**Additional Comments**

We can use ubi-8 for the base OS for our operator by changing the following in the operator metadata CSV file (`foobar-operator.clusterserviceversion.yaml`).

> image: docker.io/busybox to image: registry.access.redhat.com/ubi8/ubi
