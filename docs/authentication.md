
# Authentication

The cluster uses [Dex](https://github.com/dexidp/dex), an OpenID Connect (OIDC) provider, for authentication to its services including the cluster API server, which is hosted under `https://auth.sciwhiz12.dev`.

For user identity, Dex is configured to [authenticate using GitHub](https://dexidp.io/docs/connectors/github/) through an OAuth App, and only for users in the [@sciwhizlabs](https://github.com/sciwhizlabs) organization. This means that users not within the organization are not recognized by Dex.

Dex configures the `groups` claim to contain the team names which the user is a member of, in the format `<org>:<team slug>` (the slug can be seen in the URL of the GitHub team). This allows for fine-grained access to resources based on team membership.

## Cluster API Access

For access to the cluster API server, membership in the @sciwhizlabs/infrastructure team is required. Membership in the group grants access through the built-in `cluster-admin` group, i.e., unlimited administrator access to the cluster.

However, it should be noted that this method of access through Dex requires browser access on the same machine that `kubectl` is run on. If that is not possible because `kubectl` is running in a remote headless environment (and port-forwarding `8000` to the local machine is not possible), then other means of access to the cluster must be obtained.

To gain `kubectl` access to the cluster:

1. Install [kubelogin](https://github.com/int128/kubelogin?tab=readme-ov-file#setup). This will be the plugin that handles authenticating with Dex to issue the access tokens.

2. Run the following command to test access to the user:

    ```bash
    kubectl oidc-login setup --oidc-issuer-url=https://auth.sciwhiz12.dev --oidc-client-id=kubernetes-access
    ```

    The command will open up a browser window (or prompt with a link to open in a browser window) to continue authentication with GitHub.

    If the authentication succeeds, it should emit the issued token and some instructions to follow. (If one wishes to follow those instructions instead, start on step 5: "Set up the kubeconfig.")

3. Run the following command to setup `kubelogin` for the *current* context:

    ```bash
    kubectl config set-credentials oidc \
    --exec-api-version=client.authentication.k8s.io/v1beta1 \
    --exec-command=kubectl \
    --exec-arg=oidc-login \
    --exec-arg=get-token \
    --exec-arg=--oidc-issuer-url=https://auth.sciwhiz12.dev \
    --exec-arg=--oidc-client-id=kubernetes-access
    ```

    The `oidc` in the first line may be replaced with any username, particularly to avoid conflicts with existing usernames for accessing other clusters.
