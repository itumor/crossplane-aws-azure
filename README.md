# crossplane-aws-azure
[![Build Status](https://github.com/jonashackt/crossplane-aws-azure/workflows/provision/badge.svg)](https://github.com/jonashackt/crossplane-aws-azure/actions)
[![License](http://img.shields.io/:license-mit-blue.svg)](https://github.com/jonashackt/crossplane-aws-azure/blob/master/LICENSE)
[![renovateenabled](https://img.shields.io/badge/renovate-enabled-yellow)](https://renovatebot.com)

Example project showing how to get started with Crossplane, connect it to multiple providers like AWS, Azure - and provision some resources like a S3Bucket or a StorageAccount through K8s CRDs


Crossplane https://crossplane.io/ claims to be the "The cloud native control plane framework". It introduces a new way how to manage any cloud resource (beeing it Kubernetes-native or not). It's an alternative Infrastructure-as-Code tooling to Terraform, AWS CDK/Bicep or Pulumi and introduces a higher level of abstraction - based on Kubernetes CRDs. 

> This repo is accompanied by this blog post https://blog.codecentric.de/en/2022/07/crossplane/

Litterally the best intro post to Crossplane for me was https://blog.crossplane.io/crossplane-vs-cloud-infrastructure-addons/ - here the real key benefits especially compared to other tools are described. Without marketing blabla. If you love deep dives, I can also recommend Nate Reid's blog https://vrelevant.net/ who works as Staff Solutions Engineer at Upbound.


# Crossplane basic concepts

https://crossplane.io/docs/v1.8/concepts/overview.html

* [Managed Resourced (MR)](https://crossplane.io/docs/v1.8/concepts/managed-resources.html): Kubernetes custom resources (CRDs) that represent infrastructure primitives (mostly in cloud providers). All Crossplane Managed Resources could be found via https://doc.crds.dev/ 
* [Composite Resources (XR)](https://crossplane.io/docs/v1.8/concepts/composition.html): compose Managed Resources into higher level infrastructure units (especially interesting for platform teams). They are defined by:
    * a `CompositeResourceDefinition` (XRD) (which defines an OpenAPI schema the `Composition` needs to be conform to)
    * (optional) `CompositeResourceClaims` (XRC) (which is an abstraction of the XR for the application team to consume) - but is fantastic to hold the exact configuration parameters for the concrete resources you want to provision
    * a `Composition` that describes the actual infrastructure primitives aka `Managed Resources` used to build the Composite Resource. One XRD could have multiple Compositions - e.g. to one for every environment like development, stating and production
    * and configured by a `Configuration`
* [Packages](https://crossplane.io/docs/v1.8/concepts/packages.html): OCI container images to handle distribution, version updates, dependency management & permissions for Providers & Configurations. Packages were formerly named `Stacks`.
    * [Providers](https://crossplane.io/docs/v1.8/concepts/providers.html): are Packages that bundle a set of Managed Resources & __a Controller to provision infrastructure resources__ - all providers can be found on GitHub, e.g. [provider-aws](https://github.com/crossplane-contrib/provider-aws) or on [docs.crds.dev](https://doc.crds.dev/github.com/crossplane/provider-aws). A [list of all available Providers](https://github.com/orgs/crossplane-contrib/repositories?type=all) can also be found on GitHub.
    * [Configuration](https://crossplane.io/docs/v1.8/getting-started/create-configuration.html): define your own Composite Resources (XRs) & package them via `kubectl crossplane build configuration` (now they are a Package) - and push them to an OCI registry via `kubectl crossplane push configuration`. With this Configurations can also be easily installed into other Crossplane clusters.


![composition-how-it-works](screenshots/composition-how-it-works.svg)


# Getting started with Crossplane

In order to use Crossplane we'll need any kind of Kubernetes cluster to let it operate in. This management cluster with Crossplane installed will then provision the defined infrastructure. Using any managed Kubernetes cluster like EKS, AKS and so on is possible - or even a local [Minikube](https://minikube.sigs.k8s.io/docs/start/), [kind](https://kind.sigs.k8s.io) or [k3d](https://k3d.io/).


## Install prerequisites & fire up a K8s cluster with kind

https://crossplane.io/docs/v1.8/getting-started/install-configure.html

Be sure to have kind, the package manager Helm and kubectl installed:

```shell
brew install kind helm kubectl
```

Also we should install the crossplane CLI

```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
```

Now the `kubectl crossplane --help` command should be ready to use.


Now spin up a local kind cluster

```shell
kind create cluster --image kindest/node:v1.23.0 --wait 5m
```


## Install Crossplane with Helm

The Crossplane docs tell us to use Helm for installation:

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#install-crossplane

but create the namespace first:

```shell
  kubectl create namespace crossplane-system
```

and than:

```shell
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

As an Renovate-powered alternative we can [create our own simple [Chart.yaml](crossplane-config/install/Chart.yaml) to enable automatic updates](https://stackoverflow.com/a/71765472/4964553) of our installation if new crossplane versions get released:

```yaml
apiVersion: v2
type: application
name: crossplane
version: 0.0.0 # unused
appVersion: 0.0.0 # unused
dependencies:
  - name: crossplane
    repository: https://charts.crossplane.io/stable
    version: 1.8.0
```

To install Crossplane using our own `Chart.yaml` simply run:

```shell
helm dependency update crossplane-config/install
helm upgrade --install crossplane --namespace crossplane-system crossplane-config/install
```

Be sure to exclude `charts` and `Chart.lock` files via [.gitignore](.gitignore).

Check Crossplane version installed with `helm list -n crossplane-system` :

```shell
$ helm list -n crossplane-system
NAME      	NAMESPACE        	REVISION	UPDATED                              	STATUS  	CHART           	APP VERSION
crossplane	crossplane-system	1       	2022-06-21 09:28:21.178357 +0200 CEST	deployed	crossplane-1.8.1	1.8.1
```

Before we can actually apply a Provider we have to make sure that crossplane is actually healthy and running. Therefore we can use the `kubectl wait` command like this:

```shell
kubectl wait --for=condition=ready pod -l app=crossplane --namespace crossplane-system --timeout=120s
```

Otherwise we will run into errors like this when applying a `Provider`:

```shell
error: resource mapping not found for name: "provider-aws" namespace: "" from "provider-aws.yaml": no matches for kind "Provider" in version "pkg.crossplane.io/v1"
ensure CRDs are installed first
```

Finally check crossplane status with `kubectl get all -n crossplane-system`:

```shell
$ kubectl get all -n crossplane-system
NAME                                           READY   STATUS    RESTARTS   AGE
pod/crossplane-7c88c45998-d26wl                1/1     Running   0          69s
pod/crossplane-rbac-manager-8466dfb7fc-db9rb   1/1     Running   0          69s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/crossplane                1/1     1            1           69s
deployment.apps/crossplane-rbac-manager   1/1     1            1           69s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/crossplane-7c88c45998                1         1         1       69s
replicaset.apps/crossplane-rbac-manager-8466dfb7fc   1         1         1       69s
```



# Configure Crossplane to access AWS

https://crossplane.io/docs/v1.8/reference/configure.html

https://crossplane.io/docs/v1.8/cloud-providers/aws/aws-provider.html

https://github.com/crossplane-contrib/provider-aws


### Create aws-creds.conf file

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#get-aws-account-keyfile

I assume here that you have [aws CLI installed and configured](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). So that the command `aws configure` should work on your system. With this prepared we can create an `aws-creds.conf` file:

```shell
echo "[default]
aws_access_key_id = $(aws configure get aws_access_key_id)
aws_secret_access_key = $(aws configure get aws_secret_access_key)
" > aws-creds.conf
```

> Don't ever check this file into source control - it holds your AWS credentials! For this repository I added `*-creds.conf` to the [.gitignore](.gitignore) file. 

If you're using a CI system like GitHub Actions (as this repository is based on), you need to have both `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` configured as Repository Secrets:

![github-actions-secrets](screenshots/github-actions-secrets.png)

Also make sure to have your `Default region` configured locally - or as a `env:` variable in your CI system. All three needed variables [in GitHub Actions](.github/workflows/provision.yml) for example look like this:

```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: 'eu-central-1'
```

### Create AWS Provider secret

Now we need to use the `aws-creds.conf` file to create the Crossplane AWS Provider secret:

```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./aws-creds.conf
```

If everything went well there should be a new `aws-creds` Secret ready:

![provider-aws-secret](screenshots/provider-aws-secret.png)



### Install the Crossplane AWS Provider

https://crossplane.io/docs/v1.8/concepts/packages.html#installing-a-package

We can install crossplane Packages (which can be Providers or Configurations) via the Crossplane CLI with for example:

```shell
kubectl crossplane install provider crossplanecontrib/provider-aws:v0.22.0
```

Or we can create our own [provider-aws.yaml](crossplane-config/provider-aws.yaml) file like this:

> This `kind: Provider` with `apiVersion: pkg.crossplane.io/v1` is completely different from the `kind: Provider` which we want to consume! These use `apiVersion: meta.pkg.crossplane.io/v1`.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplanecontrib/provider-aws:v0.34.0
  packagePullPolicy: Always
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

Install the AWS provider using `kubectl`:

```shell
kubectl apply -f crossplane-config/provider-aws.yaml
```

The `package` version in combination with the `packagePullPolicy` configuration here is crucial, since we can configure an update strategy for the Provider here. I'am not sure, if the Crossplane team will provide an installation method where we can use tools like Renovate to keep our Crossplane providers up to date. A full table of all possible fields can be found in the docs: https://crossplane.io/docs/v1.8/concepts/packages.html#specpackagepullpolicy We can also let crossplane itself manage new versions for us. If you installed multiple package versions, you'll see them as `providerrevision.pkg.x` when running `kubectl get crossplane`:

```shell
$ kubectl get crossplane
...
NAME                                                           HEALTHY   REVISION   IMAGE                             STATE      DEP-FOUND   DEP-INSTALLED   AGE
providerrevision.pkg.crossplane.io/provider-aws-2189bc61e0bd   True      1          crossplanecontrib/provider-aws:v0.33.0   Inactive                               6d22h
providerrevision.pkg.crossplane.io/provider-aws-d87796863f95   True      2          crossplanecontrib/provider-aws:v0.34.0   Active                                 43h
...
```

Now our first Crossplane Provider has been installed. You may check it with `kubectl get provider`:

```shell
$ kubectl get provider.pkg.crossplane.io
NAME           INSTALLED   HEALTHY   PACKAGE                           AGE
provider-aws   True        Unknown   crossplanecontrib/provider-aws:v0.34.0   13s
```

Before we can actually apply a `ProviderConfig` to our AWS provider we have to make sure that it's actually healthy and running. Therefore we can use the `kubectl wait` command like this:

```shell
kubectl wait --for=condition=healthy --timeout=120s provider/provider-aws
```

Otherwise we may run into errors like this when applying the `ProviderConfig` right after the Provider.


### Create ProviderConfig to consume the Secret containing AWS credentials

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#configure-the-provider

https://crossplane.io/docs/v1.8/cloud-providers/aws/aws-provider.html#optional-setup-aws-provider-manually

Now we need to create `ProviderConfig` object that will tell the AWS Provider where to find it's AWS credentials. Therefore we create a [provider-config-aws.yaml](crossplane-config/provider-config-aws.yaml):

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

> Crossplane resources use the `ProviderConfig` named `default` if no specific ProviderConfig is specified, so this ProviderConfig will be the default for all AWS resources.

The `secretRef.name` and `secretRef.key` has to match the fields of the already created Secret.

Apply it with:

```shell
kubectl apply -f crossplane-config/provider-config-aws.yaml
```



# Prepare your IDE to help understand crossplane manifests

https://blog.upbound.io/moving-crossplane-package-authoring-from-plain-yaml-to-ide-aided-development/

Upbound VS Code plugin https://blog.upbound.io/crossplane-vscode-plugin-announcement/

![upbound-vscode-extension](screenshots/upbound-vscode-extension.png)




# Provision a S3 Bucket with Crossplane

__The crossplane core Controller and the Provider AWS Controller should now be ready to provision any infrastructure component in AWS!__

You heard right: we don't create a Kubernetes based infrastructure component - but we start with a simple S3 Bucket.

Therefore we can have a look into the Crossplane AWS provider API docs: https://doc.crds.dev/github.com/crossplane/provider-aws/s3.aws.crossplane.io/Bucket/v1beta1@v0.18.1


## Defining all Composite Resource components to provide an AWS EKS cluster

https://crossplane.io/docs/v1.8/concepts/composition.html#defining-composite-resources

> A CompositeResourceDefinition (or XRD) defines the type and schema of your XR. It lets Crossplane know that you want a particular kind of XR to exist, and what fields that XR should have.

Since defining your own CompositeResourceDefinitions and Compositions is the main work todo with Crossplane, it's always good to know the full Reference documentation which can be found here https://crossplane.io/docs/v1.8/reference/composition.html

One of the things to know is that Crossplane automatically injects some common 'machinery' into the manifests of the XRDs and Compositions: https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims



### Defining a CompositeResourceDefinition (XRD) for our S3 Bucket

All possible fields an XRD can have are documented here:

https://crossplane.io/docs/v1.8/reference/composition.html#compositeresourcedefinitions

The field `spec.versions.schema` must contain a OpenAPI schema, which is similar to the ones used by any Kubernetes CRDs. They determine what fields the XR (and claim) will have. The full CRD documentation and a guide on how to write OpenAPI schemas could be found in the Kubernetes docs: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Note that Crossplane will be automatically extended this section. Therefore the following fields are used by Crossplane and will be ignored if they're found in the schema:

    spec.resourceRef
    spec.resourceRefs
    spec.claimRef
    spec.writeConnectionSecretToRef
    status.conditions
    status.connectionDetails


So our Composite Resource Definition (XRD) for our S3 Bucket could look like [aws/s3/definition.yaml](aws/s3/definition.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # XRDs must be named 'x<plural>.<group>'
  name: xobjectstorages.crossplane.jonashackt.io
spec:
  # This XRD defines an XR in the 'crossplane.jonashackt.io' API group.
  # The XR or Claim must use this group together with the spec.versions[0].name as it's apiVersion, like this:
  # 'crossplane.jonashackt.io/v1alpha1'
  group: crossplane.jonashackt.io
  
  # XR names should always be prefixed with an 'X'
  names:
    kind: XObjectStorage
    plural: xobjectstorages
  # This type of XR offers a claim, which should have the same name without the 'X' prefix
  claimNames:
    kind: ObjectStorage
    plural: objectstorages
  
  # default Composition when none is specified (must match metadata.name of a provided Composition)
  # e.g. in composition.yaml
  defaultCompositionRef:
    name: objectstorage-composition

  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    # OpenAPI schema (like the one used by Kubernetes CRDs). Determines what fields
    # the XR (and claim) will have. Will be automatically extended by crossplane.
    # See https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
    # for full CRD documentation and guide on how to write OpenAPI schemas
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            # We define 2 needed parameters here one has to provide as XR or Claim spec.parameters
            properties:
              parameters:
                type: object
                properties:
                  bucketName:
                    type: string
                  region:
                    type: string
                required:
                  - bucketName
                  - region
```

Install the XRD into our cluster with:

```shell
kubectl apply -f xrd.yaml
```

We can double check the CRDs beeing created with `kubectl get crds` and filter them using `grep` to our group name `crossplane.jonashackt.io`:

```shell
$ kubectl get crds | grep crossplane.jonashackt.io
objectstorages.crossplane.jonashackt.io                         2022-06-27T09:54:18Z
xobjectstorages.crossplane.jonashackt.io                        2022-06-27T09:54:18Z
```


### Craft a Composition to manage our needed cloud resources

The main work in Crossplane has to be done crafting the Compositions. This is because they interact with the infrastructure primitives the cloud provider APIs provide.

Detailled docs to many of the possible manifest configurations can be found here https://crossplane.io/docs/v1.8/reference/composition.html#compositions

A Composite to manage an S3 Bucket in AWS with public access for static website hosting could for example look like this [aws/s3/composition.yaml](aws/s3/composition.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: objectstorage-composition
  labels:
    # An optional convention is to include a label of the XRD. This allows
    # easy discovery of compatible Compositions.
    crossplane.io/xrd: xobjectstorages.crossplane.jonashackt.io
    # The following label marks this Composition for AWS. This label can 
    # be used in 'compositionSelector' in an XR or Claim.
    provider: aws
spec:
  # Each Composition must declare that it is compatible with a particular type
  # of Composite Resource using its 'compositeTypeRef' field. The referenced
  # version must be marked 'referenceable' in the XRD that defines the XR.
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XObjectStorage
  
  # When an XR is created in response to a claim Crossplane needs to know where
  # it should create the XR's connection secret. This is configured using the
  # 'writeConnectionSecretsToNamespace' field.
  writeConnectionSecretsToNamespace: crossplane-system
  
  # Each Composition must specify at least one composed resource template.
  resources:
    # Providing a unique name for each entry is good practice.
    # Only identifies the resources entry within the Composition. Required in future crossplane API versions.
    - name: bucket
      base:
        # see https://doc.crds.dev/github.com/crossplane/provider-aws/s3.aws.crossplane.io/Bucket/v1beta1@v0.18.1
        apiVersion: s3.aws.crossplane.io/v1beta1
        kind: Bucket
        metadata: {}
        spec:
          forProvider:
            # public-read should enable public access for static website hosting
            # see https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
            acl: "public-read"
          deletionPolicy: Delete
      
      # Each resource can optionally specify a set of 'patches' that copy fields
      # from (or to) the XR.
      patches:
        # All those fieldPath refer to XR or Claim spec.parameters
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.locationConstraint"
  
  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  #patchSets:
```

Install our Composition with 

```shell
kubectl apply -f aws/s3/composition.yaml
```



### Craft a Composite Resource (XR) or Claim (XRC)

Crossplane could look quite intimidating when having a first look. There are few guides around to show how to approach a setup when using Crossplane the first time. You can choose between writing an XR __OR__ XRC! You don't need both, since the XR will be generated from the XRC, if you choose to craft a XRC.

https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims

Since we want to create a S3 Bucket, here's an suggestion for an [claim.yaml](aws/s3/claim.yaml):

```yaml
---
# Use the spec.group/spec.versions[0].name defined in the XRD
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: ObjectStorage
metadata:
  # Only claims are namespaced, unlike XRs.
  namespace: default
  name: managed-s3
spec:
  # The compositionRef specifies which Composition this XR will use to compose
  # resources when it is created, updated, or deleted.
  compositionRef:
    name: objectstorage-composition
  
  # Parameters for the Composition to provide the Managed Resources (MR) with
  # to create the actual infrastructure components
  parameters:
    bucketName: microservice-ui-nuxt-js-static-bucket2
    region: eu-central-1
```


Testdrive with `kubectl apply -f aws/s3/claim.yaml`.

When somthing goes wrong with the validation, this could look like this:

```shell
$ kubectl apply -f claim.yaml
error: error validating "claim.yaml": error validating data: [ValidationError(S3Bucket.metadata): unknown field "crossplane.io/external-name" in io.k8s.apimachinery.pkg.apis.meta.v1.ObjectMeta_v2, ValidationError(S3Bucket.spec): unknown field "parameters" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec.writeConnectionSecretToRef): missing required field "namespace" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec.writeConnectionSecretToRef, ValidationError(S3Bucket.spec): missing required field "bucketName" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec): missing required field "region" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec]; if you choose to ignore these errors, turn validation off with --validate=false
```

The Crossplane validation is a great way to debug your yaml configuration - it hints you to the actual problems that are still present.


### Troubleshooting your crossplane configuration

https://crossplane.io/docs/v1.8/reference/composition.html#tips-tricks-and-troubleshooting

https://crossplane.io/docs/v1.8/reference/troubleshoot.html


> Per Kubernetes convention, Crossplane keeps errors close to the place they happen. This means that if your claim is not becoming ready due to an issue with your Composition or with a composed resource you’ll need to “follow the references” to find out why. Your claim will only tell you that the XR is not yet ready.


The docs also tell us what they mean by "follow the references":

* Find your XR by running `kubectl describe <claim-kind> <claim-metadata.name>` and look for its “Resource Ref” (aka `spec.resourceRef`).
* Run `kubectl describe` on your XR. This is where you’ll find out about issues with the Composition you’re using, if any.
* If there are no issues but your XR doesn’t seem to be becoming ready, take a look for the “Resource Refs” (or `spec.resourceRefs`) to find your composed resources.
* Run `kubectl describe` on each referenced composed resource to determine whether it is ready and what issues, if any, it is encountering.





### Waiting for resources to become ready

There are some possible things to check while your resources (may) get deployed after running a `kubectl apply -f claim.yaml` (see https://crossplane.io/docs/v1.8/getting-started/provision-infrastructure.html#claim-your-infrastructure).

The best overview gives a `kubectl get crossplane` which will simply list all the crossplane resources:

```shell
$ kubectl get crossplane
Warning: Please use v1beta1 version of this resource that has identical schema.
NAME                                                                                          ESTABLISHED   OFFERED   AGE
compositeresourcedefinition.apiextensions.crossplane.io/xs3buckets.crossplane.jonashackt.io   True          True      23m

NAME                                               AGE
composition.apiextensions.crossplane.io/s3bucket   2d17h

NAME                                      INSTALLED   HEALTHY   PACKAGE                           AGE
provider.pkg.crossplane.io/provider-aws   True        True      crossplanecontrib/provider-aws:v0.34.0   4d21h

NAME                                                           HEALTHY   REVISION   IMAGE                             STATE    DEP-FOUND   DEP-INSTALLED   AGE
providerrevision.pkg.crossplane.io/provider-aws-2189bc61e0bd   True      1          crossplanecontrib/provider-aws:v0.34.0   Active                               4d21h

NAME                                        AGE     TYPE         DEFAULT-SCOPE
storeconfig.secrets.crossplane.io/default   5d23h   Kubernetes   crossplane-system
```

* `kubectl get claim`: get all resources of all claim kinds, like PostgreSQLInstance.
* `kubectl get composite`: get all resources that are of composite kind, like XPostgreSQLInstance.
* `kubectl get managed`: get all resources that represent a unit of external infrastructure.
* `kubectl get <name-of-provider>`: get all resources related to <provider>.
* `kubectl get crossplane`: get all resources related to Crossplane.



We can also check our claim with `kubectl get <claim-kind> <claim-metadata.name>` like this:

```shell
$ kubectl get ObjectStorage managed-s3
NAME         READY   CONNECTION-SECRET               AGE
managed-s3           managed-s3-connection-details   5s
```

To watch the provisioned resources become ready we can run `kubectl get crossplane -l crossplane.io/claim-name=<claim-metadata.name>`:

```shell
kubectl get crossplane -l crossplane.io/claim-name=managed-s3
```


Check if the S3 Bucket has been created successfully via aws CLI with `aws s3 ls | grep microservice-ui-nuxt-js-static-bucket2`.

```shell
$ aws s3 ls | grep microservice-ui-nuxt-js-static-bucket2
2022-06-27 11:56:26 microservice-ui-nuxt-js-static-bucket2
```

Our bucket should be there! We can also double check in the AWS console:

![aws-console-s3-bucket-created](screenshots/aws-console-s3-bucket-created.png)

Let's deploy our app (a simple [index.html](static/index.html)) to our S3 Bucket using the aws CLI like this:

```shell
aws s3 sync static s3://microservice-ui-nuxt-js-static-bucket2 --acl public-read
```

Now we can open up http://microservice-ui-nuxt-js-static-bucket2.s3-website.eu-central-1.amazonaws.com/ in our Browser and should see our app beeing deployed:

![s3-static-webseite-deployed](screenshots/s3-static-webseite-deployed.png)




Before removing the claim, we should remove our `index.html` - otherwise we'll run into errors like this:

```shell
  Warning  CannotDeleteExternalResource  37s (x16 over 57s)  managed/bucket.s3.aws.crossplane.io  (combined from similar events): operation error S3: DeleteBucket, https response error StatusCode: 409, RequestID: 0WHR906YZRF0YDSH, HostID: x7cz2iYF/8Ag2wKtKRZUy1j3hPk67tBUOTFeR//+grrD7plqQ5Zo6EecO70KOOgHKbY7hUyp9vU=, api error BucketNotEmpty: The bucket you tried to delete is not empty
```

So first delete the `index.html`:

```shell
aws s3 rm s3://microservice-ui-nuxt-js-static-bucket2/index.html
```

Finally remove our S3 Bucket again with 

```shell
kubectl delete claim managed-s3
```

Now also the S3 Bucket should be removed by crossplane.






# Configure Crossplane to access Azure

https://crossplane.io/docs/v1.8/cloud-providers/azure/azure-provider.html

https://github.com/crossplane-contrib/provider-azure


### Create crossplane-azure-provider-key.json

https://crossplane.io/docs/v1.8/cloud-providers/azure/azure-provider.html#preparing-your-microsoft-azure-account

I assume here that you have [azure CLI installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and you're logged in to your Azure subscription via `az login`. So that the command `az account show` should work on your system.

> The current Crossplane docs on Azure propose a `az ad sp create-for-rbac` command that isn't working: `ERROR: Usage error: To create role assignments, specify both --role and --scopes.` When using `--role` we must use `--scopes` also. The documentation about the latter isn't easy to grasp https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create-for-rbac at first sight. As there are examples like `--scopes /subscriptions/{subscriptionId}/resourceGroups/{resourceGroup1}` I thought I need to create a resourceGroup before even connecting to Azure to create a resourceGroup with Crossplane (that doesn't make any sense). BUT: It's possible to use `--scopes /subscriptions/{subscriptionId}` here without any resourceGroups, which will create a scope for your whole subscription.

So prepare the `SUBSCRIPTION_ID` variable like this:

```shell
export SUBSCRIPTION_ID=$(az account show --query id --output tsv)
```

With this prepared we can create a service principal in Azure AD using the following command:

```shell
az ad sp create-for-rbac --sdk-auth --role Owner --scopes /subscriptions/$SUBSCRIPTION_ID --name servicePrincipalCrossplaneGHActions > crossplane-azure-provider-key.json
```

> Although this produces a waring message like `WARNING: Option '--sdk-auth' has been deprecated and will be removed in a future release.` we definitely need to use the `--sdk-auth` parameter. Otherwise our Managed Resources will run into errors like `connect failed: cannot get authorizer from client credentials config: failed to get SPT from client credentials: parameter 'activeDirectoryEndpoint' cannot be empty` because there are configuration entries missing in the file like `"activeDirectoryEndpointUrl": "https://login.microsoftonline.com",`.

This produces a `crossplane-azure-provider-key.json` file you should never ever check into version control! For this repository I added `*-creds.conf` to the [.gitignore](.gitignore) file. 


If you're using a CI system like GitHub Actions (as this repository is based on), define 3 repository secrets from the output. Choose the `appId` as the `ARM_CLIENT_ID`, the `password` as the `ARM_CLIENT_SECRET` and the `tenant` as the `ARM_TENANT_ID`. Additionally we need to define the `ARM_SUBSCRIPTION_ID` secret. Therefore run a `az account show` (after you logged your CLI into your Azure subscription via `azure login`) and use the value of `"id":`.

![github-actions-secrets-azure](screenshots/github-actions-secrets-azure.png)

On GitHub Actions we need to 

```shell
echo "{
  \"clientId\": \"$ARM_CLIENT_ID\",
  \"clientSecret\": \"$ARM_CLIENT_SECRET\",
  \"subscriptionId\": \"$ARM_SUBSCRIPTION_ID\",
  \"tenantId\": \"$ARM_TENANT_ID\",
  \"activeDirectoryEndpointUrl\": \"https://login.microsoftonline.com\",
  \"resourceManagerEndpointUrl\": \"https://management.azure.com/\",
  \"activeDirectoryGraphResourceId\": \"https://graph.windows.net/\",
  \"sqlManagementEndpointUrl\": \"https://management.core.windows.net:8443/\",
  \"galleryEndpointUrl\": \"https://gallery.azure.com/\",
  \"managementEndpointUrl\": \"https://management.core.windows.net/\"
}" > crossplane-azure-provider-key.json
```


All three needed variables [in GitHub Actions](.github/workflows/provision-azure.yml) for example look like this:

```yaml
env:
  ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
  ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
```


### Create Azure Provider secret

Now we need to use the `crossplane-azure-provider-key.json` file to create the Crossplane Azure Provider secret:

```shell
kubectl create secret generic azure-account-creds -n crossplane-system --from-file=creds=./crossplane-azure-provider-key.json
```

If everything went well there should be a new `azure-account-creds` Secret ready.



### Install the Crossplane Azure Provider

https://crossplane.io/docs/v1.8/concepts/packages.html#installing-a-package

We can install crossplane Packages (which can be Providers or Configurations) via the Crossplane CLI with for example:

```shell
kubectl crossplane install provider crossplane/provider-azure:v0.19.0
```

Or we can create our own [provider-aws.yaml](crossplane-config/provider-aws.yaml) file like this:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure
spec:
  package: crossplane/provider-azure:v0.19.0
  packagePullPolicy: Always
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

Install the Azure provider using `kubectl`:

```
kubectl apply -f crossplane-config/provider-azure.yaml
```

Now our first Crossplane Provider has been installed. You may check it with `kubectl get provider`:

```shell
$ kubectl get provider
NAME             INSTALLED   HEALTHY   PACKAGE                            AGE
provider-azure   True        Unknown   crossplane/provider-azure:v0.19.0  13s
```

Before we can actually apply a `ProviderConfig` to our Azure provider we have to make sure that it's actually healthy and running. Therefore we can use the `kubectl wait` command like this:

```shell
kubectl wait --for=condition=healthy --timeout=120s provider/provider-azure
```

Otherwise we may run into errors like this when applying the `ProviderConfig` right after the Provider.


### Create ProviderConfig to consume the Secret containing AWS credentials

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#configure-the-provider

https://crossplane.io/docs/v1.8/cloud-providers/azure/azure-provider.html#setup-azure-providerconfig

Now we need to create `ProviderConfig` object that will tell the AWS Provider where to find it's AWS credentials. Therefore we create a [provider-config-azure.yaml](crossplane-config/provider-config-azure.yaml):

```yaml
apiVersion: azure.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-account-creds
      key: creds
```

> Crossplane resources use the `ProviderConfig` named `default` if no specific ProviderConfig is specified, so this ProviderConfig will be the default for all Azure resources.

The `secretRef.name` and `secretRef.key` has to match the fields of the already created Secret.

Apply it with:

```shell
kubectl apply -f crossplane-config/provider-config-azure.yaml
```

__The crossplane core Controller and the Provider Azure Controller should now be ready to provision any infrastructure component in Azure!__



# Provision a StorageAccount in Azure with Crossplane

https://doc.crds.dev/github.com/crossplane/provider-azure/storage.azure.crossplane.io/Account/v1alpha3@v0.19.0

https://doc.crds.dev/github.com/crossplane/provider-azure/azure.crossplane.io/ResourceGroup/v1alpha3@v0.19.0



### Defining a CompositeResourceDefinition (XRD) for our Storage Account

So our Composite Resource Definition (XRD) for our Storage Account could look like [azure/storageaccount/definition.yaml](azure/storageaccount/definition.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xstoragesazure.crossplane.jonashackt.io
spec:
  group: crossplane.jonashackt.io
  names:
    kind: XStorageAzure
    plural: xstoragesazure
  claimNames:
    kind: StorageAzure
    plural: storagesazure
  
  defaultCompositionRef:
    name: storageazure-composition

  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    # See https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  location:
                    type: string
                  resourceGroupName:
                    type: string
                  storageAccountName:
                    type: string
                required:
                  - location
                  - resourceGroupName
                  - storageAccountName
```

Install the XRD into our cluster with:

```shell
kubectl apply -f azure/storageaccount/definition.yaml
```

Let's wait for the XRD to become `Offered`:

```shell
kubectl wait --for=condition=Offered --timeout=120s xrd xstoragesazure.crossplane.jonashackt.io  
```


### Craft a Composition to manage our needed cloud resources

A Composite to manage an Storage Account in Azure with public access for static website hosting could for example look like this [azure/storageaccount/composition.yaml](azure/storageaccount/composition.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: storageazure-composition
  labels:
    crossplane.io/xrd: xstoragesazure.crossplane.jonashackt.io
    provider: azure
spec:
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XStorageAzure
  
  writeConnectionSecretsToNamespace: crossplane-system
  
  resources:
    - name: storageaccount
      base:
        # see https://doc.crds.dev/github.com/crossplane/provider-azure/storage.azure.crossplane.io/Account/v1alpha3@v0.19.0
        apiVersion: storage.azure.crossplane.io/v1alpha3
        kind: Account
        metadata: {}
        spec:
          storageAccountSpec: 
            kind: Storage
            sku:
              name: Standard_LRS
              tier: Standard
      patches:
        - fromFieldPath: spec.parameters.storageAccountName
          toFieldPath: metadata.name
        - fromFieldPath: spec.parameters.resourceGroupName
          toFieldPath: spec.resourceGroupName
        - fromFieldPath: spec.parameters.location
          toFieldPath: spec.storageAccountSpec.location
          
    - name: resourcegroup
      base:
        # see https://doc.crds.dev/github.com/crossplane/provider-azure/azure.crossplane.io/ResourceGroup/v1alpha3@v0.19.0
        apiVersion: azure.crossplane.io/v1alpha3
        kind: ResourceGroup
        metadata: {}
      patches:
        - fromFieldPath: spec.parameters.resourceGroupName
          toFieldPath: metadata.name
        - fromFieldPath: spec.parameters.location
          toFieldPath: spec.location
```

Install our Composition with 

```shell
kubectl apply -f azure/storageaccount/composition.yaml
```



### Craft a Composite Resource (XR) or Claim (XRC)

Since we want to create a Storage Account, here's an suggestion for an [claim.yaml](azure/storageaccount/claim.yaml):

```yaml
---
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: StorageAzure
metadata:
  namespace: default
  name: account
spec:
  compositionRef:
    name: storageazure-composition
  parameters:
    location: West Europe
    resourceGroupName: rg-crossplane
    storageAccountName: account4c8672e
```


Testdrive with 

```shell
kubectl apply -f azure/storageaccount/claim.yaml
```

Now have a look into the Azure Portal. Our Resource Group should show up:

![azure-console-resourcegroup](screenshots/azure-console-resourcegroup.png)

And also our Storage account should be visible inside the group:

![azure-console-storageaccount](screenshots/azure-console-storageaccount.png)









# TODO:


## Publish Crossplane Configuration Package into GitHub Container Registry

Let's package our Composite Resource definitions as a Configuration. Therefore we need to use the Crossplane CLI via <code>kubectl crossplane build configuration</code> (now they are a Package) - and push them to an OCI registry via <code>kubectl crossplane push configuration</code>. With this Configurations can also be easily installed into other Crossplane clusters.

Therefore we need a `crossplane.yaml` file as described in https://crossplane.io/docs/v1.8/getting-started/create-configuration.html#build-and-push-the-configuration 

See also https://crossplane.io/docs/v1.8/concepts/packages.html#configuration-packages

Our [aws/s3/crossplane.yaml](aws/s3/crossplane.yaml) is of `kind: Configuration` and defines the minimum crossplane version needed alongside the crossplane AWS provider:

```yaml
apiVersion: meta.pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: s3-bucket-composition
spec:
  crossplane:
    version: ">=v1.8"
  dependsOn:
    - provider: crossplanecontrib/provider-aws
      version: ">=v0.33.0"
```

Having this `crossplane.yaml` in place we can build our Configuration at last. On a command line go into the directory where the `crossplane.yaml` resides and run the `kubectl crossplane build` command:

```shell
$ cd crossplane-s3
$ kubectl crossplane build configuration
```



Really strange, getting

```shell
kubectl crossplane build configuration
kubectl crossplane: error: failed to build package: failed to parse package: {path:/Users/jonashecht/dev/kubernetes/crossplane-kind-eks/aws/s3/composition.yaml position:0}: no kind "S3Bucket" is registered for version "crossplane.jonashackt.io/v1alpha1" in scheme "/home/runner/work/crossplane/crossplane/internal/xpkg/scheme.go:47"
```








# Provision an EKS cluster with crossplane

__The crossplane core Controller and the Provider AWS Controller should now be ready to provision any infrastructure component in AWS!__

So in order to maximise the Inception let's provision an EKS based Kubernetes cluster in AWS with our crossplane Kubernetes cluster :) 

A EKS cluster is a complex AWS construct leveraging different basic AWS services. That's exactly what a crossplane Composite Resource (XR) is all about. So let's create our own Composite Resource.





## Higher level abstractions - or why it is so hard to write your own Compositions

Looking only at the Crossplane docs and blog I thought something is missing: A curated library of higher level abstractions (like Pulumi Crosswalk). First initiatives from cloud vendors like AWS Blueprints for Crossplane: https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/

But then I found the Upbound blog - and finally stumbled upon the __Upbound Platform Reference Architectures__ https://blog.upbound.io/azure-reference-platform/ which are intended as

> foundations to help accelerate understanding and application of Crossplane within your organization


### Use Upbound Platform Reference Architecture AWS to provision a EKS cluster

__What I didn't wanted to do is to duplicate some code__ into my `Composition` from all those sources I found (mainly this https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane, this https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/ and https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/) about provisioning a EKS cluster with crossplane. 




### Install platform-ref-aws

https://github.com/upbound/platform-ref-aws#install-the-platform-configuration






# Links

https://crossplane.io/

Intro post https://medium.com/@khelge/crossplane-7197ad200013

Crossplane is based on OAM (3 roles: Developer, application operator & infrastructure operator) https://github.com/oam-dev/spec

https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane

https://www.forbes.com/sites/janakirammsv/2021/09/15/how-crossplane-transforms-kubernetes-into-a-universal-control-plane/

> Crossplane essentially transforms Kubernetes into a universal control plane that can orchestrate the lifecycle of public cloud-based services such as virtual machines, database instances, Big Data clusters, machine learning jobs, and other managed services offered by the hyperscalers. 

> Crossplane takes the concept of Infrastructure as Code (IaC) to the next level through its tight integration with Kubernetes.

Intro Slides: https://docs.google.com/presentation/d/1PxZweRpB6HElxd9qGK1McboGZ1kluCDCS5qxgYnX5f0/edit#slide=id.g8801599ecb_2_169


Promoted from sandbox to incubation by the CNCF: https://blog.crossplane.io/crossplane-cncf-incubation/ 

Crossplane vs. Terraform: https://blog.crossplane.io/crossplane-vs-terraform/

Crossplane 101 https://vrelevant.net/crossplane-bringing-the-basics-together/



https://www.techtarget.com/searchitoperations/news/252508176/Crossplane-project-could-disrupt-infrastructure-as-code


Argo + crossplane https://morningspace.medium.com/using-crossplane-in-gitops-what-to-check-in-git-76c08a5ff0c4


Infrastructure-as-Apps https://codefresh.io/blog/infrastructure-as-apps-the-gitops-future-of-infra-as-code/


VMWare support https://blog.crossplane.io/adding-vmware-support-to-crossplane-using-terraform/


### Crossplane + AWS

https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/

> Crossplane’s infrastructure provider for AWS relies on code generated by the AWS Go Code Generator, which is also used by [AWS Controllers for Kubernetes (ACK)](https://github.com/aws-controllers-k8s/community). 

https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane




AWS Blueprints for Crossplane: 

https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/

> Writing your first Composition and debugging can be intimidating.

And it's quite a doublicate effort IMHO compared to other tools like Pulumi that provide higher level abstractions including well-architectured best practices (like Pulumi https://www.pulumi.com/docs/guides/crosswalk/aws/)

https://github.com/aws-samples/crossplane-aws-blueprints


Thoughworks Tech Radar: Assess https://www.thoughtworks.com/de-de/radar/tools/crossplane


IONOS and crossplane https://docs.ionos.com/crossplane-provider/example


### Stacks now called Packages

https://github.com/crossplane/crossplane/blob/master/design/design-doc-composition.md

>  Note that Crossplane packages were formerly known as 'Stacks'. This term has been rescoped to refer specifically to a package of infrastructure configuration but is frequently synonymous with packages in historical design documents and some remaining Crossplane APIs.


## Others

#### KUTTL - KUbernetes Test TooL

https://kuttl.dev/

example: https://github.com/aws-samples/crossplane-aws-blueprints/blob/main/tests/kuttl/test-suite.yaml