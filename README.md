# Argo CD Setup Notes

This repository now has an EKS cluster running, and Argo CD was installed in the cluster.

## 1. Create the Argo CD namespace

```bash
kubectl create namespace argocd
```

## 2. Install Argo CD HA manifests

```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.4/manifests/ha/install.yaml
```

## 3. Change the Argo CD server service from `ClusterIP` to `NodePort`

Edit the `argocd-server` service:

```bash
kubectl edit svc argocd-server -n argocd
```

Change:

```yaml
type: ClusterIP
```

to:

```yaml
type: NodePort
```

Then save the service and note the assigned NodePort.

## 4. Get the initial Argo CD admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```

Default username:

```text
admin
```

## 5. Log in to the Argo CD UI

Open the Argo CD UI in the browser using:

```text
http://<node-ip>:<nodeport>
```

Use:

- Username: `admin`
- Password: output from `argocd-initial-admin-secret`

## 6. Reset the password

After logging in, open the user info/profile console in the Argo CD UI and reset the admin password.

## 7. SSH into an EKS worker node

Use the EC2 key pair private key and connect to one of the worker node public IPs:

```bash
chmod 400 ~/.ssh/qamar-key-pair.pem
ssh -i ~/.ssh/qamar-key-pair.pem ec2-user@<node-public-ip>
```

## 8. Install the Argo CD CLI on the node

Download the Argo CD CLI binary:

```bash
wget https://github.com/argoproj/argo-cd/releases/download/v3.3.4/argocd-linux-amd64
mv argocd-linux-amd64 argocd
sudo chmod +x argocd
sudo mv argocd /usr/local/bin
```

Verify installation:

```bash
argocd version --client
```

## 9. Log in to Argo CD from the node using the CLI

Use the Argo CD server ClusterIP:

```bash
argocd login 10.100.79.219
```

Because the TLS certificate does not include the ClusterIP as an IP SAN, the CLI prompts for insecure access:

```text
WARNING: server certificate had error: error creating connection: tls: failed to verify certificate: x509: cannot validate certificate for 10.100.79.219 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? yes
Username: admin
Password:
'admin:login' logged in successfully
Context '10.100.79.219' updated
```

When prompted:

- Enter `yes` to proceed insecurely
- Use username `admin`
- Use the password from `argocd-initial-admin-secret` or the updated password if it was already reset

## 10. Add the GitHub repository in the Argo CD UI

Log in to the Argo CD UI, then:

1. Go to **Settings**.
2. Open **Repositories**.
3. Click **Connect Repo**.
4. Choose the connection method:

For a public GitHub repository:

- Select **VIA HTTPS**
- Enter the repository URL
- Click **Connect**

Example:

```text
https://github.com/<github-username>/<repo-name>.git
```

For a private GitHub repository:

- Select **VIA HTTPS**
- Enter the repository URL
- Enter GitHub username
- Enter GitHub personal access token/password
- Click **Connect**

After a successful connection, the repository appears in the Argo CD repositories list.

## 11. Create an Argo CD application using the UI

In the Argo CD UI:

1. Click **NEW APP**.
2. Fill in the application details.

Recommended values:

- Application Name: `solar-system`
- Project Name: `default`
- Sync Policy: `Manual` or `Automatic`

Under **Source**:

- Repository URL: select the connected GitHub repository
- Revision: branch name such as `main`
- Path: repository folder containing Kubernetes manifests, for example `solar-system`

Under **Destination**:

- Cluster URL: `https://kubernetes.default.svc`
- Namespace: target namespace where the app should be deployed

Then:

1. Click **Create**.
2. Open the application.
3. Click **Sync**.
4. Confirm the sync to deploy the manifests.

After sync completes, Argo CD shows the application status, health, and Kubernetes resources created from the GitHub repository.

## 12. Add the GitHub repository and create the application using the Argo CD CLI

After logging in with the Argo CD CLI, add the GitHub repository if needed:

For a public repository:

```bash
argocd repo add https://github.com/imhashmi8/gitops-argocd-k8s-deployment.git
```

For a private repository:

```bash
argocd repo add https://github.com/imhashmi8/gitops-argocd-k8s-deployment.git \
  --username <github-username> \
  --password <github-personal-access-token>
```

Create the target namespace if it does not already exist:

```bash
kubectl create namespace solar-system
```

Create the Argo CD application from the CLI:

```bash
argocd app create solar-system-app-2 \
  --repo https://github.com/imhashmi8/gitops-argocd-k8s-deployment.git \
  --path solar-system \
  --revision main \
  --dest-namespace solar-system \
  --dest-server https://kubernetes.default.svc
```

Sync the application:

```bash
argocd app sync solar-system-app-2
```

Check application status:

```bash
argocd app get solar-system-app-2
```

Example with an `nginx` application using automatic sync, auto-pruning, and self-healing:

```bash
argocd app create nginx-auto-sync \
  --repo https://github.com/imhashmi8/gitops-argocd-k8s-deployment.git \
  --path nginx \
  --revision main \
  --dest-namespace nginx \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

This configuration means:

- `--sync-policy automated`: Argo CD syncs changes from Git automatically
- `--auto-prune`: resources removed from Git are also removed from the cluster
- `--self-heal`: resources changed manually in the cluster are reconciled back to the Git state

## 13. Screenshots

The screenshots are stored in the `images/` folder.

### Argo CD applications overview

![Argo CD applications overview](images/argocd-applications-overview.png)

### Argo CD application details tree

![Argo CD application details tree](images/argocd-application-details-tree.png)

### Solar system application UI

![Solar system application UI](images/solar-system-app-ui.png)
