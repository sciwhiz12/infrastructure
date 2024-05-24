
# Bootstrapping

Bootstrapping the Kubernetes (K8s) cluster from this repository is not too difficult. Roughly, the goal is to setup Argo CD on a fresh cluster, add this repository as a source, and load the `core` and `apps` applications.

## Sealed secrets

Because of the use of sealed secrets, bootstrapping the cluster from this repository requires having access to a backup of the most recent sealing key used for sealed secrets. Without that key, all sealed secrets in this repository must be *regenerated* anew for full operational capacity.

In that case, the cluster should be bootstrapped normally so the [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) controller is made available by Argo CD. After the controller is loaded, the regeneration of the sealed secrets may begin.

If the cluster is currently active, the sealing key can be obtained for backup through the following command:

```sh
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets.key
```

## Step-by-step guide

Before you begin, ensure you have [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl) and [`helm`](https://helm.sh/docs/intro/install/) installed. [`kubeseal`](https://github.com/bitnami-labs/sealed-secrets#kubeseal) should also be installed for interfacing with the sealed-secrets controller.

1. **Prepare the server node.** This guide assumes the use of [k3s](https://docs.k3s.io/), a lightweight and production-ready Kubernetes distribution.

    To install as a standalone cluster (or as first node in a cluster), run the following command:

    ```sh
    curl -sfL https://get.k3s.io | sh -
    ```

    Obtain the Kubernetes config from `/etc/rancher/k3s/k3s.yaml` (to use locally). Remember to ensure the cluster URL points to the actual cluster's URL. Configure `kubectl` to use that cluster (and by extension, so will `helm` and `kubeseal`.)

2. **Install Argo CD.** [Argo CD](https://argoproj.github.io/cd) is a declarative, GitOps continuous delivery tool for Kubernetes.

    Installing Argo CD is done using Helm, through the following commands:

    ```sh
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
    helm install argocd argoproj/argo-cd -n argocd --create-namespace
    ```

3. **Login to the Argo CD web interface.** After installing Argo CD, the command-line output should contain instructions on connecting to the web interface. For reference, the following are the same steps:

    To access the web interface on your local machine, forward the `8080` port from the `argocd-server` service, through the following command in *another* terminal:

    ```sh
    kubectl port-forward service/argocd-server -n argocd 8080:80
    ```

    The web interface has a default account with the username `admin`, and a password stored in the `argocd-initial-admin-secret` secret (under the `password` key). Run the following to obtain the password:

    ```sh
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 --decode
    ```

4. **(Optional) Restore the sealing key for sealed secrets.** If the sealing key is available, restoring the key before loading up the controller avoids having to restart the controller later.

    The sealing key should be stored in a form of an (non-sealed) `Secret` containing `tls.crt` and `tls.key`. Assuming the sealing key is stored under `sealed-secrets.key`, restore the secrets by applying the manifest, through the following command:

    ```sh
    kubectl apply -f sealed-secrets.key
    ```

5. **Add this repository in Argo CD.** This guide assumes this repository is hosted on GitHub as a private repository, and that it will be accessed through a [deploy key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys).

    First, generate a key pair to be used as the deploy key *without a passphrase*, by running the following:

    ```sh
    ssh-keygen -f ./argocd-key
    ```

    Add the deploy key's public part (`argocd-key.pub`) by following [the instructions on GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#set-up-deploy-keys). Give it a descriptive name, such as "ArgoCD deployment".

    Obtain the SSH URL for the repository by navigating to the home screen of the GitHub repository, clicking the `Code` button, and selecting the `SSH` option.

    On Argo CD, navigate to `Settings` on the left menu, then `Repositories`. Click the `Connect Repo` button at the top bar, and input the repository URL and the deploy key's private part (`argocd-key`) in the indicated fields. Put the repository name in the `Name` field, and select `default` for the Project field.

6. **Configure initial settings in this repository.** There are certain settings which need adjustment before deploying from this repository.

    Under `core/values.yaml`, adjust the following values:

    - `domain` should be the full domain which the cluster will run under. (For example, `example.com`.)
    - `repository.url` should be the URL to the repository *exactly* as obtained in the previous step.
    - `firstInstall` should be set to `true`.

7. **Deploy the `core` and `apps` applications.** These two Argo Applications will load the rest of the applications and core services. `core` should be deployed first, and then `apps` after the core services settle down.

    On the `Applications` screen, click the `New App` at the top bar. Do the following:

    - Enter `core` (or `apps`) for the Application Name field.
    - Enter `default` for the Project Name field.
    - Select `Automatic` for the Sync Policy dropdown.
    - Tick the `Auto-Create Namespace` checkbox.
    - Select the previously-added repository for the Repository URL field.
    - Ensure the Revision field is set to `HEAD`.
    - Enter `core` (or `apps`) for the Path field.
    - Select the one available option for the Cluster URL field.

    Once finished, click the `Create` button at the top bar. Argo CD will then deploy the `core` (or `apps`) application, and all dependents.

    After creating the `core` Application and waiting for all applications to finish syncing, create the `apps` Application.

8. **Create a repository webhook.** This causes pushes to the repository to make Argo CD refresh and deploy changes quickly, instead of waiting for the polling timer. As before, this guide assumes the repository is hosted on GitHub.

    This requires that the Argo CD interface is deployed to a publicly accessible domain. (For example's sake, this guide will assume Argo CD is available at `argo.example.com`)

    To avoid denial-of-service by constantly refreshing Argo CD, the webhook will be secured using a random secret token, which is stored in the repository as a sealed secret. Obtain the value of the secret through whatever means available (either through `kubectl get secret`, or regenerating the sealed secret with a known value).

    Follow [the instructions on GitHub](https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks#creating-a-repository-webhook) to add a repository webhook, with the following notes:

    - The URL of the webhook is the Argo domain with the path `/api/webhook`. (For example, `argo.example.com/api/webhook`.)
    - The Content type field is set to `application/json`.
    - The Secret field should have the secret token as mentioned above.
    - Select `Just the push event` for the events that will trigger the webhook.
    - Ensure the `Active` checkbox and the `Enable SSL verification` radio button are selected.

    If the setup is correct, after a few moments, a green checkmark should appear next to the webhook.

9. **Reset the settings in this repository back to normal.**

    Change back the `firstInstall` key in `core/values.yaml` to `false`.

## Conclusion

Congratulations, the cluster is now setup correctly!

Remember to regularly backup the sealing key for the sealed secrets, to avoid having to regenerate all sealed secrets anew. Note that the sealing key is rotated around every three months, so the backup should be automated and safely stored away.

Of course, this setup needs to only be done for the first node in a cluster. Any additional nodes connected to the cluster will be managed automatically.

---

*P.S. This file is named as intended.*
