# olm-testing-bundle-format-index
A simple OLM testing example for the bundle format with opm

Bundle Format Operator Directory Strcuture:

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

Let's build and push the operator bundle as an image:

- docker build -f bundle.Dockerfile -t quay.io/taneem/foobar-operator:v0.0.1 .
- login to quay.io: docker login quay.io -u <quay_username>
- docker push quay.io/taneem/foobar-operator:v0.0.1
- Login to quay.io web portal and set the repository to public (under repository settings). 

Let's validate the remote bundle-format image with operator-sdk (v1.0.0+):

- operator-sdk bundle validate 
- Note: if the validation is successful, then we should see only INFO level messages (no WARN or ERROR) in the output console and it should end with "All validation tests have completed successfully".

Let's build an index image for the operator:

- opm index add --bundles quay.io/taneem/foobar-operator:v0.0.1 --tag quay.io/taneem/foobar-operator-index:0.0.1 --build-tool docker