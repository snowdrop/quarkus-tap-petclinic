## Table of Contents

   * [Table of Contents](#table-of-contents)
   * [Quarkus Petclinic accelerator](#quarkus-petclinic-accelerator)
   * [1. Scenario to be re-tested using a pod able to perform the buildpack steps](#1-scenario-to-be-re-tested-using-a-pod-able-to-perform-the-buildpack-steps)
   * [2. Scenario tested using kpack deployed on a k8s cluster with a local docker registry](#2-scenario-tested-using-kpack-deployed-on-a-k8s-cluster-with-a-local-docker-registry)
      * [Kpack controller](#kpack-controller)
      * [Deploy the runtime resources](#deploy-the-runtime-resources)
      * [Build an image](#build-an-image)
      * [Deploy the quarkus application](#deploy-the-quarkus-application)
   * [Additional notes](#additional-notes)
   
## Quarkus Petclinic accelerator

A Quarkus Petclinic sample accelerator

You can build the image using docker, jib or using kpack if it has been deployed on the k8s platform
  
## 1. Scenario to be re-tested using a pod able to perform the buildpack steps

Build the quarkus buildpack images using the upstream project and push the images to the local registry: `local.registry:5000`
```bash
git clone https://github.com/quarkusio/quarkus-buildpacks.git && cd quarkus-buildpacks

# Generate the buildpack quarkus images (build, run and builder)
./create-buildpacks.sh

# Tag and push the images to the private docker registry
export REGISTRY_URL="registry.local:5000"
docker tag redhat/buildpacks-builder-quarkus-jvm:latest ${REGISTRY_URL}/redhat-builder/quarkus:latest
docker tag redhat/buildpacks-stack-quarkus-run:jvm ${REGISTRY_URL}/redhat-buildpacks/quarkus:run
docker tag redhat/buildpacks-stack-quarkus-build:jvm ${REGISTRY_URL}/redhat-buildpacks/quarkus:build

docker push ${REGISTRY_URL}/redhat-builder/quarkus:latest
docker push ${REGISTRY_URL}/redhat-buildpacks/quarkus:build
docker push ${REGISTRY_URL}/redhat-buildpacks/quarkus:run
```

As the `builder` image packages the lifecycle creator but also the `run` and `build` image, it is important to get its SHA in order
to use the matching image deployed on k8s. You can get it using this command `docker images --no-trunc --quiet redhat/buildpacks-builder-quarkus-jvm:latest`

Deploy next on the registry needed images
```bash
docker pull alpine:3.14 && \
  docker tag alpine:3.14 ${REGISTRY_URL}/alpine:3.14 && \
  docker push ${REGISTRY_URL}/alpine:3.14
docker pull busybox:1.28 && \
  docker tag busybox:1.28 ${REGISTRY_URL}/busybox:1.28 && \
  docker push ${REGISTRY_URL}/busybox:1.28 
```

Next, create a configMap containing the selfsigned certificate of the container registry under the namespace `demo`
```bash
kubectl create ns demo
kc create -n demo cm local-registry-cert --from-file $HOME/local-registry.crt
```

Create a secret to access your local registry
```bash
export REGISTRY_URL="registry.local:5000"
kubectl create secret docker-registry registry-creds -n demo \
  --docker-server="${REGISTRY_URL}" \
  --docker-username="admin" \
  --docker-password="snowdrop"
```
Next deploy the deployment resource able to perform a build using a runtime example (e.g. )
```bash
kubectl apply -f k8s/build-pod/manifest.yml
kubectl delete -f k8s/build-pod/manifest.yml
```
Watch the progression of the build 
```bash
kubectl -n demo logs -lapp=quarkus-petclinic-image-build -c build -f
```

## 2. Scenario tested using kpack deployed on a k8s cluster with a local docker registry

Install kind and a private secured/TLS registry locally (using registry version 2.6 !)
```bash
git clone kind-tls-pwd-registry https://github.com/snowdrop/k8s-infra.git && cd k8s-infra/kind
./k8s/kind-tls-secured-reg.sh
```
**NOTE**: The certificate generated is copied within the file `$HOME/local-registry.crt` and the user, password to be used to be authenticated with the registry are respectively` admin` and `snwodrop`

Build the quarkus buildpack images using the upstream project and push the images to the local registry: `local.registry:5000`
```bash
git clone https://github.com/quarkusio/quarkus-buildpacks.git && cd quarkus-buildpacks

# Generate the buildpack quarkus images (build, run and builder)
./create-buildpacks.sh

# Tag and push the images to the private docker registry
export REGISTRY_URL="registry.local:5000"
docker tag redhat/buildpacks-builder-quarkus-jvm:latest ${REGISTRY_URL}/redhat-buildpacks/quarkus-java:latest
docker tag redhat/buildpacks-stack-quarkus-run:jvm ${REGISTRY_URL}/redhat-buildpacks/quarkus:run
docker tag redhat/buildpacks-stack-quarkus-build:jvm ${REGISTRY_URL}/redhat-buildpacks/quarkus:build

docker push ${REGISTRY_URL}/redhat-buildpacks/quarkus-java:latest
docker push ${REGISTRY_URL}/redhat-buildpacks/quarkus:build
docker push ${REGISTRY_URL}/redhat-buildpacks/quarkus:run
```

### Kpack controller

To be able to use the upstream [kpack](https://github.com/pivotal/kpack) project with a TLS secured registry, it is needed to install a webhook on kubernetes
able to inject the `selfsigned certificate` of the registry.

This is why it is needed to execute the following commands described hereafter to: 
- Build the images needed (to run a webhook, inject the certificate),
- Generate the k8s manifest yaml file deploying the webhook,
- To configure the webhook to fetch pod having a specific label (e.g. `image.kpack.io/image`),
- To be able to inject in a pod an `initContainer` which will, from a secret, deploy the certificate using `/usr/sbin/update-ca-certificates`, 

**NOTE**: Please use the `paketobuildpacks/builder:base` ad the default builder which is `tiny` do not include the command `/usr/sbin/update-ca-certificates` - see [ticket](https://github.com/vmware-tanzu/cert-injection-webhook/issues/9)!
```bash
git clone -b support-private-docker-registry https://github.com/ch007m/cert-injection-webhook.git && cd cert-injection-webhook
REGISTRY_URL="registry.local:5000"
pack build ${REGISTRY_URL}/setup-ca-cert -e BP_GO_TARGETS="./cmd/setup-ca-certs" -B paketobuildpacks/builder:base
pack build ${REGISTRY_URL}/pod-webhook -e BP_GO_TARGETS="./cmd/pod-webhook"
docker push ${REGISTRY_URL}/setup-ca-cert
docker push ${REGISTRY_URL}/pod-webhook
  
LABELS="image.kpack.io/image"
ytt   -f ./deployments/k8s \
      -v pod_webhook_image="${REGISTRY_URL}/pod-webhook" \
      -v setup_ca_certs_image="${REGISTRY_URL}/setup-ca-cert" \
      -v docker_server="${REGISTRY_URL}/" \
      -v docker_username="admin" \
      -v docker_password="snowdrop" \
      --data-value-file ca_cert_data=$HOME/local-registry.crt \
      --data-value-yaml labels="[${LABELS}]" \
      > manifest.yaml

kapp deploy -a inject-cert-webhook -f manifest.yaml -y
kapp delete -a inject-cert-webhook -y
```
**NOTE**: The label `image.kpack.io/image` allows to inject the cert within all the pods which are created to build an image using kpack and buildpack builders.

Next, we can deploy kpack using ytt and overlay files. They have been created part of this project to patch the upstream release.yaml file in order the issues reported [issue-845](https://github.com/pivotal/kpack/issues/845) and [issue-844](https://github.com/pivotal/kpack/issues/844).
They will allow to inject the registry selfsigned certificate and to let the kpack controller to be logged with the registry using the mounted `.dockercfgjson` file

```bash
ytt -f ./k8s/kpack-upstream/values.yaml \
    -f ./k8s/kpack-upstream/config/ \
    -f $HOME/local-registry.crt \
    -v docker_repository="${REGISTRY_URL}/" \
    -v docker_username="admin" \
    -v docker_password="snowdrop" \
    | kapp deploy -a kpack -f- -y

kapp delete -a kpack
```

### Deploy the runtime resources
Create a secret to access your local registry
```bash
kubectl create ns demo
kubectl create secret docker-registry registry-creds -n demo \
  --docker-server="https://${REGISTRY_URL}" \
  --docker-username="admin" \
  --docker-password="snowdrop"
  
kubectl delete -n demo secret/registry-creds  
```

Deploy the kpack runtime CRs (Store, Builder and Stack)
```bash
kapp deploy -a runtime-kpack \
  -f k8s/runtime-kpack/sa.yml \
  -f k8s/runtime-kpack/clusterstore.yml \
  -f k8s/runtime-kpack/clusterbuilder.yml \
  -f k8s/runtime-kpack/clusterstack.yml -y

kapp delete -a runtime-kpack -y
```

### Build an image
To build a quarkus buildpack image using the code of the local project
```bash
# To be executed at the root of the project ;-)
kp image create quarkus-petclinic-image \
  --tag registry.local:5000/quarkus-petclinic \
  --local-path ./ \
  -c runtime \
  -n demo \
  --registry-ca-cert-path $HOME/local-registry.crt
```
To list the image and status
```bash
kp image list -n demo
kp image status quarkus-petclinic-image -n demo
```
To delete the image/build
```bash
kp image delete quarkus-petclinic-image -n demo
```

### Deploy the quarkus application
We can now deploy the application
```bash
kapp deploy -a quarkus-petclinic \
  -n demo \
  -f ./k8s/service.yml \
  -f ./k8s/deployment.yml

kapp delete -a quarkus-petclinic -n -y
```
and play with it from the browser `http://localhost:31000` :-)

## Additional notes

If you prefer to use Tanzu Build Service and not kpack, then follow the steps described hereafter

Fetch the TBS images and push them to your local registry
```bash
imgpkg copy -b "registry.pivotal.io/build-service/bundle:1.2.2" --to-repo registry.local:5000/kpack --registry-ca-cert-path $HOME/local-registry.crt
```
Extract the files to configure TBS to use your local registry
```bash
imgpkg pull -b "registry.local:5000/kpack:1.2.2" -o ./k8s/kpack --registry-ca-cert-path $HOME/local-registry.crt
```
Deploy TBS using your custom config
```bash
ytt -f ./k8s/kpack/values.yaml \
    -f ./k8s/kpack/config/ \
    -f $HOME/local-registry.crt \
    -v docker_repository="registry.local:5000/" \
    -v docker_username="admin" \
    -v docker_password="snowdrop" \
    | kbld -f ./k8s/kpack/.imgpkg/images.yml -f- \
    | kapp deploy -a kpack -f- -y

kapp delete -a kpack
```