# obs-wif

### Steps to install an OpenShift Cluster with Workload Identity

0. Prerequisite:
   - A linux machine is required to perform steps below, because the `ccoctl` command only has linux version
   - Pull secret is required to get image and OCP installation. Can be downloaded from [OpenShift Cluster Manager](https://console.redhat.com/openshift/install/pull-secret)
   - GCP service account info is required when creating install-config.yaml in step 4

1. Set the variable `$RELEASE_IMAGE`

   `$RELEASE_IMAGE` should be a recent and supported  OpenShift release image that you want to deploy in your cluster.
   Please refer to the [support matrix](../README.md#support-matrix) for compatibilities.

   A sample release image would be `RELEASE_IMAGE=quay.io/openshift-release-dev/ocp-release:${RHOCP_version}-${Arch}`

   Where `RHOCP_version` is the OpenShift version (e.g `4.10.0-fc.4` or `4.9.3`) and the `Arch` is the architecture type (e.g `x86_64`)

2. Extract the GCP Credentials Request objects from the above release image. You must use version 4.7 or newer of the `oc` CLI.
   ```
   mkdir credreqs ; oc adm release extract --cloud=gcp --credentials-requests $RELEASE_IMAGE --to=./credreqs
   ```
3. Extract the `openshift-install` and `ccoctl` binaries from the release image.
   ```
   oc adm release extract --command=openshift-install $RELEASE_IMAGE --registry-config=${PULL_SECRET_PATH:-.}/pull-secret
   CCO_IMAGE=$(oc adm release info --image-for='cloud-credential-operator' ${RELEASE_IMAGE}) && oc image extract ${CCO_IMAGE} --file='/usr/bin/ccoctl' --registry-config=${PULL_SECRET_PATH:-.}/pull-secret
   ```
4. Create an install-config.yaml
   ```
   ./openshift-install create install-config
   ```
5. Make sure that we install the cluster in Manual mode
   ```
   echo "credentialsMode: Manual" >> install-config.yaml
   ``` 
6. Create install manifests
   ```
   ./openshift-install create manifests   
   ```
7. Create GCP resources using the [ccoctl](./ccoctl.md) tool (you will need GCP credentials with sufficient permissions). The following command will generate public/private ServiceAccount signing keys, create the cloud storage bucket, upload the OIDC config into the bucket, set up a workload identity pool/provider, and create an IAM service account for each GCP Credentials Request. It will also dump the files needed by the installer in the `output_dir` directory
   ```
   ccoctl gcp create-all --name=<gcp_infra_name> --region=<gcp_region> --project=<gcp-project-id> --credentials-requests-dir=/path/to/credreqs/directory/created/in/step/2 --output-dir=<output_dir>
   ```
8. Copy the manifests created in the step 7 and put them in the same location as install-config.yaml in the `manifests` directory
   ```
   cp _output/manifests/* /path/to/dir/with/install-config.yaml/manifests/
   ```
9. Copy the private key for the ServiceAccount signer and put it in the same location as install-config.yaml
   ```
   cp -a _output/tls /path/to/dir/with/install-config.yaml
   ```
10. Run the OpenShift installer
    ```
    ./openshift-install create cluster --log-level=debug
    ```

### Steps to configure ACM Observability 
0. Prerequisite
   - Install ACM >= 2.8.0
   - Create bucket in GCP side

1. Create GCP Service Account for ACM Observability(<gcp_infra_name> should be the same value as the one in step 7 above)
```
./ccoctl gcp create-service-accounts --credentials-requests-dir=./cr --name=<gcp_infra_name> --project=<gcp-project-id> --workload-identity-pool=<gcp_infra_name> --workload-identity-provider=<gcp_infra_name>
```

2. Create k8s secret for ACM Observability using the generated yaml in step 1
```
oc apply -f ./manifests/open-cluster-management-observability-cloud-credentials-credentials.yaml
```

3. Get the the json content of the service account which created in step 1. Replace the <bucket_name> and content of service_account in the yaml and create the secret.
```
oc apply -f thanos-object-storage.yaml
```

4. Enalbe ACM Observabiilty.
```
oc apply -f mco_wif.yaml
```