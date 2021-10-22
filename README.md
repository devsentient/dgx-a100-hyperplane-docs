# Installing Hyperplane on DGX A100
Documentation for Hyperplane installation on DGX A100s

## Prerequisites
- DGX OS 5.x
- ssh and password-less sudo access to your DGX A100
- nvidia-docker version >= 2.0 with docker as the [default runtime](https://github.com/NVIDIA/nvidia-container-runtime)
- Kubernetes version >= 1.10
- NVIDIA drivers >= 450.119
- CUDA Version >= 11.0

Other installation tools:
- `kubectl` on your local machine. See [install guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)
- `helm` on your local machine. See [install guide](https://helm.sh/docs/intro/install/)
- `k3sup` on your local machine. See [install guide](https://github.com/alexellis/k3sup)

## DGX A100 GPU setup
SSH into your baremetal DGX A100 to begin set up. Example:
```
ssh cyxtera@131.226.200.233
```
Ensure your GPUs are MIG-enabled.

## Save your ssh key before k3sup
From your local machine:
```
ssh-copy-id -i ~/.ssh/id_rsa.pub cyxtera@131.226.200.235
```

## Deploy k3s cluster with k3sup

From your local machine, run the following command to get a k3s cluster started on your DGX A100:
```
k3sup install --ip [SERVER_IP] --user [USERNAME] \
  --k3s-extra-args '--no-deploy traefik --docker' \
  --context hyperplane-dgx --local-path ~/.kube/config_dgx
```
Example:
```
k3sup install --ip 131.226.200.233 --user cyxtera \
  --k3s-extra-args '--no-deploy traefik --docker' \
  --context hyperplane-dgx --local-path ~/.kube/config_dgx
```
Note that you will need password-less sudo access for the above command to work. 

Ensure you are using the new kube context file for the rest of the installation.
```
export KUBECONFIG=~/.kube/config_dgx
kubectl config set-context hyperplane-dgx
kubectl get node -o wide
```

Verify that you are on the "hyperplane-dgx" context with 
```
kubectx
```

If you are not on the correct context, add the config from `~/.kube/config_dgx` to `~/.kube/config` and change the "current-context" field in your `~/.kube/config` file to "hyperplane-dgx". 


## Set up MIG devices for k3s using the NVIDIA GPU Operator

Ensure your GPUs are MIG-enabled and set up with some profiles.
```
sudo nvidia-smi -i 0,1,2,3,4,5,6,7 -mig 1
sudo nvidia-smi mig -cgi 9,14,14 -C
```

From your local machine, download and install the GPU Operator using `helm`.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
   && chmod 700 get_helm.sh \
   && ./get_helm.sh

helm repo add nvidia https://nvidia.github.io/gpu-operator \
   && helm repo update
```

Install the GPU operator with MIG.
See [documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/gpu-operator-mig.html#install-gpu-operator-mig).
```
helm install --wait --generate-name      nvidia/gpu-operator --set mig.strategy=mixed --set driver.enabled=false
```

Configure MIG profiles. For example
```
kubectl label nodes [nodename] nvidia.com/mig.config=all-3g.20gb
```


Ensure MIG pods work by deploying a test pod: 
```
kubectl apply -f mig-pod.yaml
```

Ensure the mig-pod status is "RUNNING":
```
kubectl get pods
```
You may have to wait 30 seconds and retry until the status is "RUNNING".


## Label your node(s)
From your local machine, run the following to label your node(s). This label is a node selector for various Hyperplane elements.
```
kubectl label nodes [nodename] hyperplane.dev/nodeType=dgx-pool
```
Example:
```
kubectl label nodes iad1-c632678dgx001 hyperplane.dev/nodeType=dgx-pool
```

## Deploy Hyperplane YAMLs

1. Deploy istio; you may use your own gateway and istio deployment and skip this step--just use your istio gateway in step 2. We have provided some examples of istio setup under the istio_yamls folder. To use our examples:

  - change the "subdomain_name" in `value.yaml` to your new name of choice
  - run `python generate_yamls_from_template.py` to generate yamls with the new template.
  - `/bin/bash apply_yamls_istio.sh`
  
  - Register a new subdomain if you wish to expose your dashboard externally. For our example, go to Google Network Services and set up [Cloud DNS](https://console.cloud.google.com/net-services/dns/zones/). Add the LoadBalancer external IP address and DNS Name `*.[subdomain_name].hyperplane.dev` and `[subdomain_name].hyperplane.dev`. With our Cyxtera DGX A100, the load balancer IP is an internal IP address, so we used the IP address provided for us by Cyxtera.

  Check that your oauth proxy pod is running with `kubectl get pods --all-namespaces`.

  - apply `certs/`. These yamls will use letsencrypt to provide certs (subject to limits and throttling). Please replace with your own as needed. 
  For our example, apply using
  ```
  find ./generated_yamls/certs* -type f -print0 | xargs -0 ls -tr | while read file
  do
    kubectl apply -f "$file" --validate=false
  done
  ```
  wait until status of certs is "Certificate is up to date and has not expired"
  
  - `/bin/bash apply_yamls_istio_2.sh`

2. Before you use `python generate_yamls.py` to generate yamls with values from `value.yaml`
   - If you're using your own istio gateway, choose a `subdomain_name` if you haven't already, and update it `values.yaml`. Your hyperplane deployment will be at [subdomain_name].hyperplane.dev (once the deployment is complete)
   - If you have your own istio gateway, change `istio_gateway_full` to match yours in the format of __istio_gateway_namespace/istio_gateway_name__. If you used our setup, you can keep this as is.
   Now, run `python generate_yamls.py` to generate yamls.
   
3. Apply the rest of the Hyperplane YAMLs with `/bin/bash apply_all_hyperplane_yamls.sh`


## Configure Keycloak
1) If using Google, add URIs to [Google APIs & Services Credentials](https://console.developers.google.com/apis/credentials/oauthclient/)

- URIs: `https://[subdomain_name].hyperplane.dev`
- Authorized redirect URIs: `https://[subdomain_name].hyperplane.dev/auth/realms/Hyperplane/broker/google/endpoint`

Note that [subdomain_name] is the one set in `value.yaml`.

2) Add Hyperplane Realm to Keycloak  
Go to `[subdomain_name].hyperplane.dev/auth/` in your browser. Go to the Admin Console and use the username `admin` and password `password`. Add the following information:
- Realm: `Hyperplane`
  - Themes:
    - Login themes: `devsentient`
- Clients: 
  - Settings:
    - clientID: `istio`
    - Valid Redirect URIs: 
      - `https://grafana.[subdomain_name].hyperplane.dev/*`
      - `https://*.[subdomain_name].hyperplane.dev/*`
      - `https://[subdomain_name].hyperplane.dev/*`
      - `https://jhub.[subdomain_name].hyperplane.dev/*`
      - `https://pgadmin.[subdomain_name].hyperplane.dev/*`
      - `https://[subdomain_name].hyperplane*`
    - Standard Flow Enabled
    - Implicit Flow Enabled
    - weborigins: 
      - `"*"`
      - `*`
  - Mappers:
    - Name: `istio`
    - Mapper Type: `"Audience"`
    - Included Client Audience: `istio`
    - Add to access token: `on`
  - [Optional] Identity provider 
    - Google: using client ID and secret from the Google creds in step 1. 
    - Enable "Trust Email" (or you will have to manually approve each login) from Keycloak (see below).


## Visit `[subdomain_name].hyperplane.dev`
If you have set up Hyperplane with istio and keycloak, you should be able to login with keycloak credentials. 

If you are seeing "Access to [subdomain_name].hyperplane.dev was denied", go to Keycloak > Users > View all users > select your user id > turn on "Email Verified" > Save. Refresh and try to login again.

## Add users to jupyterhub
Login with `admin` and `password`.  
Add new users from the "admin" tab

## Clone our demo repo into jhub
From within your jupyterhub single user server, run the following to clone our demo repo.
```
cp /etc/git-secret/id_rsa ~/.ssh/
chmod 400 ~/.ssh/id_rsa
git clone git@github.com:devsentient/dgx3-bfcc4e0.git
```

## Add gsutil to user jhub notebooks
Add gsutils to be able to access output files for each run. From within your jupyterhub single user server, run the following:
```
wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-349.0.0-linux-x86_64.tar.gz
tar -zxvf google-cloud-sdk-349.0.0-linux-x86_64.tar.gz
gcloud auth activate-service-account --key-file=/etc/gke-service-account-json/gcp-service-account-credentials.json
```


## Other notes for Shakudo:

### Tearing down/ cleaning up server
If you would like to tear down your cluster, on the baremetal DGX A100 (ssh), run the following:
```
/usr/local/bin/k3s-uninstall.sh
docker rm -vf $(docker ps -a -q)
docker rmi -f $(docker images -a -q)
```
