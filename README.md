# Providing release images to disconnected environments

While there are a number of components to offline support, this document specifically focuses on making a release image and all related images available from a registry internal to a customer’s network.  RHCoS images can be upgraded from the contents of the release image, and so a customer will only infrequently need to redownload those images from access.redhat.com.  OLM images are not covered as part of this design.

A customer should be able to retrieve the contents of a given update, mirror them to a local registry, and upgrade a cluster from that registry.

There are three goals for disconnected users

* Verifiability - ensure that the version of content running on a cluster can be traced back to content delivered by Red Hat
* Convenience - ensure that it is not unduly burdensome for the range of users who run disconnected (power users, travellers, trainers, production DMZs, and true air-gapped systems) to replicate content
* Simplicity - avoid creating novel processes for disconnected environment

In order to preserve verifiability, we do not want the customer to recreate the release image, which would remove any guarantee that the customer is running the exact bits released and signed by Red Hat.  Instead, we want customers to mirror the release image and the images referenced by the release image to an internal registry, then provide additional information during upgrade about that internal registry.  The cluster version operator would dynamically rewrite the image repository string of affected images (leaving digest unchanged) when an update was applied.

Example:

Admin runs the command to mirror the release contents to their internal registry

```
oc adm release mirror \
    --from=quay.io/openshift/ocp-release:4.0.1 \
    --to=mycompany.com/myregistry/openshift-4.0.1
```

Which copies all images that are part of 4.0.1 to the new location.  The command prints the information the admin would provide to a cluster.


Admin runs the command to upgrade to that mirrored version:

```
oc adm upgrade --to-mirror-image=mycompany.com/myregistry/openshift-4.0.1
```

Which infers the desired version from the payload at that location


The cluster version operator is passed the following API fields as a result

```
image: mycompany.com/myregistry/openshift-4.0.1:release
mirrorRepository: mycompany.com/myregistry/openshift-4.0.1
```

and knows to find images that were previously on registry.redhat.io or quay from the provided mirrorRepository location.


Any Docker compatible schema v2.2 registry can be used as a mirror, including Harbor, Docker Enterprise Registry, an OpenShift integrated registry, Quay.io (post v2.2 update), or the open source docker registry.

**Note: if someone wants to bypass verifiability (not something we want customers to do), a user can today (in 4.0 beta1) run:

```
	oc adm release new \
  --from=quay.io/openshift-release-dev/ocp-release:4.0.0-0.1 \
  --mirror=mycompany.com/myregistry/openshift-4.0.0-0.1 \
  --to-image=mycompany.com/myregistry/openshift-4.0.0-0.1:release
```

And then provide

```
	oc adm upgrade \
         --to-image mycompany.com/myregistry/openshift-4.0.0-0.1:release
```

To upgrade.  However, this results in a payload that can’t be traced back to red hat and so we would not support it.

We expect to iteratively improve disconnected clusters over time by refining and improving the process.  Future releases will explore:

* An easy to run command that copies these images to a disk (USB drive or otherwise) and then can upload them again afterwards.  This can be accomplished today with skopeo or docker save but is annoying.
* Possibly distribution of Cincinnati into customer environments to serve the necessary metadata
* Integrations into quay to automate loading this content

---
Threat model and security precautions for release images
---

Relevant workflow:
* The release image contains a binary (the CVO) that runs as root on the cluster. 
* It downloads and runs another image that will become the new running CVO, which then runs the manifests and images referenced
* The CVO can fetch images requested by the user to form the next payload
* The CVO fetches information about new versions from the cincinnati service, and the user can configure the cluster to point to another instance of that API

We do not need to defend against:
* Actions taken by the cluster admin that weaken security of the cluster intentionally
* Applications that run as root on the cluster
* An admin who decides to run a development version release image
* An admin who is tricked into disabling clearly identified flags or fields related to protecting the cluster version
* An attacker gaining access to the red hat content signing keys
* An attacker gaining access to the keys that sign the release payload
* An admin deliberately downgrading their cluster to an older version

We should have other defenses against:
* An attacker injecting malicious code into the upstream GitHub projects
* An attacker compromising the internal Red Hat build systems
* An attacker denying systems the ability to retrieve new update information to get critical fixes

We must defend against:
* An attacker compromising the Cincinnati service and suggesting malicious release images
* An attacker compromising the Quay server and serving malicious release images
* An attacker compromising the Red Hat content CDN and serving additional content
* An admin being tricked into switching to an update that is malicious
* An admin being tricked into downgrading their cluster to an older version

The CVO is designed to mitigate the known attacks by:

* Creating the payload from a set of known image digests produced by a trusted and verifiable build system (to inherit the verification of those systems)
* Using digests to refer to images in the payload and hence by operators (to prevent tags from being altered if quay or RH CDN is compromised)
* Only using binaries defined in the payload to construct the payload (the cli and cluster-version-operator are both part of the default payload)

In addition, we must:

* Ensure the set of inputs to building a release image is a known set of images by digest, trusted by the release build tooling (inc. signatures, attribution)
* Ensure the release image is signed by a key that is known to the CVO before we update to it
* Ensure we can rotate that key even for very old versions
* Ensure the CVO can access the signature proving that the release image was signed by the known key before it applies an update
* Default to requiring a known key to apply updates, but allow administrators to temporarily disable that protection in a way that balances testing and security
* Ensure the key that signs those images is protected
* Ensure that the permissions that control the ClusterVersion object are sufficient to prevent accidental mutation

---
How do I get added to the release payload?
---

* Add the following to your Dockerfile

```
FROM …

ADD manifests-for-operator/ /manifests
LABEL io.openshift.release.operator=true
```

* Ensure your image is published into the cluster release tag by ci-operator
* Wait for a new release payload to be created (usually once you push to master in your operator).

---
What do I put in /manifests?
---

You need the following:

1..N manifest yaml or JSON files (preferably YAML for readability) that deploy your operator, including:

* Namespace for your operator
* Roles your operator needs
* A service account and a service account role binding
* Deployment for your operator
* An OperatorStatus CR
* Any other config objects your operator might need
* An image-references file (See below)

In your deployment you can reference the latest development version of your operator image (quay.io/openshift/origin-machine-api-operator:latest).  If you have other hard-coded image strings, try to put them as environment variables on your deployment or as a config map.

Your manifests will be applied in alphabetical order by the CVO, so name your files in the order you want them run.

Names of manifest files:

If you are a normal operator (don’t need to run before the kube apiserver), you should name your manifest files in a way that feels easy:

```
/manifests/
  deployment.yaml
  roles.yaml
```

If you’d like to ensure your manifests are applied in order to the cluster add a numeric prefix to sort in the directory:

```
/manifests/
  01_roles.yaml
  02_deployment.yaml
```

When your manifests are added to the release payload, they’ll be given a prefix that corresponds to the name of your repo/image:

```
/release-manifests/
  99_ingress-operator_01_roles.yaml
  99_ingress-operator_02_deployment.yaml
```
---
How do I get added as a special run level?
---

Some operators need to run at a specific time in the release process (OLM, kube, openshift core operators, network, service CA).  These components can ensure they run in a specific order across operators by prefixing their manifests with:

```
	0000_<runlevel>_<dash-separated_component>-<manifest_filename>
```

For example, the Kube core operators run in runlevel 10 and have filenames like

```
	0000_13_cluster-kube-scheduler-operator_03_crd.yaml
```

Assigned runlevels

* 00-04 - CVO
* 07 - Network operator
* 08 - DNS operator
* 09 - Service signer CA
* 10-19 - Kube operators (master team)
* 20-29 - OpenShift core operators (master team)
* 30-39 - OLM
* 70 - auto assigned operators

If you don’t assign a runlevel, you are given 70 automatically.  Your operator can add manifests at multiple run levels, but in general we recommend not doing that.

---
How do I ensure the right images get used by my manifests?
---

Your manifests can contain a tag to the latest development image published by Origin.  You’ll annotate your manifests by creating a file that identifies those images.

Assume you have two images in your manifests - `quay.io/openshift/origin-ingress-operator:latest` and `quay.io/openshift/origin-haproxy-router:latest`.  Those correspond to the following tags `ingress-operator` and `haproxy-router` when the CI runs.

Create a file `image-references` in the `/manifests` dir with the following contents:

```
kind: ImageStream
apiVersion: image.openshift.io/v1
spec:
  tags:
  - name: ingress-operator
    from:
      kind: DockerImage
      name: quay.io/openshift/origin-ingress-operator
  - name: haproxy-router
    from:
      kind: DockerImage
      name: quay.io/openshift/origin-haproxy-router
```

The release tooling will read image-references and do the following operations:

* Verify that an tag “ingress-operator” and “haproxy-router” exists from the release / CI tooling (in the image stream openshift/origin-v4.0 on api.ci).  If they don’t exist, you’ll get a build error.
* Do a find and replace in your manifests (effectively a sed)  that replaces 

```
quay.io/openshift/origin-haproxy-router(:.*|@:.*) with registry.svc.ci.openshift.org/openshift/origin-v4.0@sha256:<latest SHA for :haproxy-router>
```

* Store the fact that operator ingress-operator uses both of those images in a metadata file alongside the manifests
* Bundle up your manifests and the metadata file as a docker image and push them to a registry

Later on, when someone wants to mirror a particular release, there will be tooling that can take the list of all images used by operators and mirror them to a new repo.

This pattern tries to balance between having the manifests in your source repo be able to deploy your latest upstream code *and* allowing us to get a full listing of all images used by various operators.



