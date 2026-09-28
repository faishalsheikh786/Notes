# Understanding `~/.kube/config` and `~/.ssh/config`

A practical guide to how `kubectl`, SSH, and Git choose a destination and authenticate. Examples use two EKS learning clusters and separate GitHub/GitLab keys. Replace example names, account IDs, and paths with your own values.

## 1. The two folders at a glance

| File | Used by | Main question answered |
| --- | --- | --- |
| `~/.kube/config` | `kubectl` and Kubernetes clients | Which cluster should I contact, and how should I authenticate? |
| `~/.ssh/config` | SSH (and Git when using SSH remotes) | Which host should I contact, as which remote user, and with which key? |

`~` means your home directory. A name beginning with `.` is hidden by default. If your shell prompt ends in `.kube %` or `.ssh %`, `cat config` reads the config in that current directory; `cat ~/.kube/config` and `cat ~/.ssh/config` name the files explicitly.

These files are local client settings. Editing a label in either one does not rename an EKS cluster, change a GitHub account, or grant permissions on a remote system.

# Part A: Kubernetes kubeconfig

## 2. What a kubeconfig contains

A kubeconfig typically has `clusters`, `users`, `contexts`, and `current-context`:

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: dev-cluster
    cluster:
      server: https://EXAMPLE.gr7.us-east-2.eks.amazonaws.com
      certificate-authority-data: <existing base64 CA data>
users:
  - name: dev-aws-auth
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - --region
          - us-east-2
          - eks
          - get-token
          - --cluster-name
          - eks-learning-dev-cluster
          - --output
          - json
        interactiveMode: IfAvailable
        provideClusterInfo: false
contexts:
  - name: dev
    context:
      cluster: dev-cluster
      user: dev-aws-auth
      namespace: default
current-context: dev
```

This is illustrative: the CA placeholder and example server are not a working configuration. In your original file, the two actual cluster entries are `eks-learning` in `us-east-1` and `eks-learning-dev-cluster` in `us-east-2`. Your displayed `current-context` selected the latter. The cluster ARN strings used as entry names are local identifiers, not proof of the AWS identity you are signed in as.

### `apiVersion` and `kind`

`apiVersion: v1` and `kind: Config` describe the kubeconfig document format. They do not report your cluster's Kubernetes version. Likewise, `client.authentication.k8s.io/v1beta1` under `exec` describes the credential plugin response format, not the cluster version.

### `clusters`: address and server verification

Each `clusters[].name` is a local lookup name. Its `server` is the Kubernetes API endpoint, not an application, pod, or worker node. `certificate-authority-data` is the encoded CA certificate used by the client to verify the server's TLS certificate. The CA identifies a trusted server; it does not identify you to the server. The CA is a certificate, not a private key.

A cluster entry can alternatively use a `certificate-authority` file path. Keep the configured CA and server matched to the real cluster; do not invent either value merely to make the names readable.

### `users`: client authentication instructions

A kubeconfig `user` is a named set of client credentials or instructions. It is not necessarily a human user or a Kubernetes `User` object. In your EKS config, `exec.command: aws` tells `kubectl` to invoke the AWS CLI. The `args` correspond approximately to:

```bash
aws --region us-east-2 eks get-token --cluster-name eks-learning-dev-cluster --output json
```

The AWS CLI uses the credentials available to that process and returns a short-lived token. The config you showed does not specify `--profile`, so the normal AWS CLI credential selection applies, including environment variables and the configured profile. `env: null` means the kubeconfig exec block adds no special environment variables. `interactiveMode: IfAvailable` permits interactive input when available; `provideClusterInfo: false` means cluster metadata is not supplied to the plugin.

The entry's `name` is merely a pointer for contexts. Naming it `dev-aws-auth` does not create an AWS user called `dev-aws-auth`.

### `contexts`: join a cluster and a user

A context points to **one cluster entry**, **one user entry**, and optionally a default `namespace`. In the example, context `dev` refers to `dev-cluster` and `dev-aws-auth`. All these references must match their entry names exactly. With no context namespace, Kubernetes commands normally target the `default` namespace unless overridden by `-n`/`--namespace` or command specific behavior. `kubectl get pods -A` requests pods across namespaces, subject to permissions.

### `current-context`: the default choice

`current-context: dev` selects the context used when you do not provide `--context`. A command-level `--context qa` overrides it for that invocation. `kubectl config use-context qa` changes the local default. These operations do not move workloads between clusters.

## 3. What happens for `kubectl get pods`

1. `kubectl` loads kubeconfig: explicit `--kubeconfig` if given, otherwise `KUBECONFIG` if set, otherwise `~/.kube/config`. `KUBECONFIG` may name multiple files that are merged.
2. It selects the command's `--context`, or the file's `current-context`.
3. The selected context names a cluster and a user. Its namespace is used unless overridden.
4. The cluster entry supplies the API server URL and CA. HTTPS verifies the server against the CA.
5. The user entry obtains authentication material. In your case, `aws eks get-token` creates a token using your effective AWS credentials.
6. EKS verifies the token and identifies the AWS IAM principal. Access entries or a legacy `aws-auth` mapping allow that principal into the cluster according to the cluster's authentication mode.
7. EKS access policies and/or Kubernetes RBAC decide whether that principal can **list pods** in the selected namespace. If allowed, pods are returned; otherwise the API may respond `Forbidden`. Connection or credential failures produce different errors.

The `server` and CA establish **where** and **which server**; the authentication token establishes **who**; authorization rules establish **what that identity may do**. Having a kubeconfig does not itself grant cluster access.

## 4. Inspect and switch contexts

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config get-clusters
kubectl config get-users
kubectl config view --minify
kubectl config view --raw --minify   # May display credentials; keep output private
kubectl config use-context dev
kubectl --context qa get pods -n payments
kubectl auth can-i list pods -n default
aws sts get-caller-identity
```

`aws sts get-caller-identity` shows the AWS identity selected by your current AWS CLI environment. If a kubeconfig entry supplies its own exec environment or profile, reproduce that selection when checking identity. `kubectl auth can-i` asks the selected cluster about one permission; it does not list all AWS account permissions.

## 5. Friendly names for dev, QA, stage, prod

Use distinct context labels such as `myapp-dev-us-east-2`, `myapp-qa-us-east-2`, `myapp-stage-us-east-2`, and `myapp-prod-us-east-1`. A context label can be renamed without renaming its cluster or auth entry:

```bash
kubectl config rename-context \
  'arn:aws:eks:us-east-2:140173112512:cluster/eks-learning-dev-cluster' \
  myapp-dev-us-east-2
kubectl config get-contexts
```

`rename-context` also updates `current-context` if the renamed context was current. This modifies local kubeconfig. There is no corresponding `kubectl config rename-user` command. If you want to rename a user entry, back up the file, change `users[].name`, and change **every** context `user:` reference to match. `kubectl config set-context CONTEXT --user=NEW_NAME` updates one context pointer, but does not copy the old user's `exec` settings into a new entry. Avoid deleting the old entry until references and authentication work.

| Value | Readability change? | Consequence |
| --- | --- | --- |
| `clusters[].name` | Yes | Update every context `cluster:` reference. |
| `users[].name` | Yes | Update every context `user:` reference. |
| `contexts[].name` | Yes | Use `rename-context`; update scripts that use the old name. |
| `current-context` | Yes | Must equal an existing context name. |
| `server`, CA | Only for the real target cluster | Wrong values prevent or misdirect a connection. |
| `--region`, `--cluster-name` | Only for the real EKS cluster | Must match the cluster for token generation. |
| `namespace` | Yes | Changes the default namespace for that context. |

A label `prod` does not make a dev cluster production. Each environment requires its own real cluster information and appropriate access; some organizations also have several environments within one cluster, so use accurate names.

### Adding another real EKS cluster

After obtaining its cluster name, region, account access, and permissions:

```bash
aws eks update-kubeconfig --region us-east-2 --name REAL_QA_CLUSTER --alias myapp-qa-us-east-2
kubectl config get-contexts
```

This fetches actual endpoint and CA information and writes/updates kubeconfig. It may set the added context as current, so check before your next `kubectl` command. AWS access to describe the cluster and Kubernetes access to workloads are separate. With multiple AWS profiles, you can choose a profile when generating the file and configure an appropriate profile or role for later token generation; verify the resulting `users[].user.exec` settings before relying on them.

### Safe edits and troubleshooting

```bash
cp ~/.kube/config ~/.kube/config.backup
kubectl config view --minify
kubectl --context myapp-dev-us-east-2 get pods
```

Check `echo "$KUBECONFIG"` if changes seem to affect a different file. If an API endpoint cannot be reached, check network/VPN and cluster endpoint access. If you see a credential error, check the AWS CLI identity and token command. If you see `Forbidden`, check the EKS access entry or legacy mapping and the associated access policy/RBAC permissions. Avoid sharing `--raw` config output publicly; kubeconfig can contain secrets in other authentication setups.

# Part B: SSH configuration

## 6. Typical files in `~/.ssh`

| File | Purpose |
| --- | --- |
| `config` | Per-destination SSH connection settings. |
| `id_ed25519`, `id_ed25519_gitlab`, `ansible-lab` | Example private keys. Never paste or publish their contents. |
| Matching `.pub` files | Public keys to register with the account or install on a server. |
| `known_hosts` | Trusted host key records used to detect an unexpected server key. |
| `known_hosts.old` | An older backup of host key records; not the active default. |
| `agent/` | Local agent-related data, depending on the setup. |

A private/public key pair is linked mathematically. The public key can be registered with GitHub or GitLab; the private key stays on your Mac. SSH proves possession of the private key without sending it to the server. `known_hosts` works in the other direction: it helps your Mac verify the **server**. The server's host key is distinct from your account's public key.

## 7. Read your GitHub and GitLab host rules

Your screenshot shows GitHub using `~/.ssh/id_ed25519` and GitLab using `~/.ssh/id_ed25519_gitlab`:

```sshconfig
Host github.com
    HostName github.com
    User git
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host gitlab.com
    HostName gitlab.com
    User git
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519_gitlab
    IdentitiesOnly yes
```

| Directive | What it does |
| --- | --- |
| `Host` | Pattern/alias matched against the destination you type. Can contain multiple names or wildcards. |
| `HostName` | Actual DNS name or IP address to contact. |
| `User` | Remote SSH login name; GitHub and GitLab normally use `git`. |
| `IdentityFile` | Path to the private key SSH should try. |
| `IdentitiesOnly yes` | Limits offered authentication identities to configured ones, even if the agent holds more. |
| `AddKeysToAgent yes` | Allows SSH to load the key into the local SSH agent. |
| `UseKeychain yes` | Apple OpenSSH setting to use macOS Keychain for a key passphrase. |

Other useful directives include `Port 22` (remote SSH port), `ProxyJump bastion-alias` (connect through a jump host), `ServerAliveInterval 30` (send keepalive requests), and `ForwardAgent` (agent forwarding; enable only where specifically needed). A broad `Host *` block can set defaults. SSH config is not YAML; write `Host example`, not `Host: example`.

## 8. Host aliases and account usernames

Assume your GitHub account is `github-username`, your GitLab account is `gitlab-username`, and their respective public keys have been added to those accounts. You can use readable aliases:

```sshconfig
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host gitlab-work
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab
    IdentitiesOnly yes
```

Then test and clone:

```bash
ssh -T github-work
# GitHub's successful authentication greeting identifies github-username.
git clone git@github-work:github-username/my-repo.git

ssh -T gitlab-work
# GitLab's greeting identifies gitlab-username.
git clone git@gitlab-work:gitlab-username/my-repo.git
```

In `git@github-work:github-username/my-repo.git`:

- `git` is the SSH service login, not your personal account name.
- `github-work` is the local alias; `HostName` resolves it to `github.com`.
- `github-username/my-repo.git` is the repository namespace and name. A repository in an organization uses that organization's namespace instead.
- The public key registered with the service identifies which account is authenticating; repository permissions decide whether that account may clone or push.

`ssh -T` tests SSH authentication without requesting an interactive shell. Some Git hosting services return a nonzero shell exit status even after a successful greeting, because they do not provide shell access; read the message rather than using the exit code alone.

If you replace `Host github.com` with `Host github-work`, old remotes using `git@github.com:...` will no longer match that rule. Preserve both names if both URL styles are needed:

```sshconfig
Host github.com github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Check and update a Git remote if desired:

```bash
git remote -v
git remote set-url origin git@github-work:github-username/my-repo.git
```

## 9. What can you rename in SSH config?

| Value | Changeable? | What must also change? |
| --- | --- | --- |
| `Host github-work` | Yes | Use the alias in `ssh` or Git URLs, or keep the original host pattern too. |
| `HostName github.com` | Only for a different real destination | It must resolve to the intended server. |
| `User git` | Only if the destination expects another remote login | Do not put your GitHub/GitLab account username here for their normal SSH service. |
| `IdentityFile` path | Yes | Move/rename the actual private key to match; keep its public key paired. |
| Key filename | Yes | Update all SSH config references and key management scripts. |
| `Port` | Yes | Must match the remote server's listening port. |
| Directive names | No | `HostName`, `IdentityFile`, etc. are SSH's syntax. |

An EC2 host is a different example: `User ec2-user` may be correct for an Amazon Linux instance, and `IdentityFile` might point to `~/.ssh/ansible-lab`. The remote user depends on the AMI and server setup:

```sshconfig
Host ansible-lab-ec2
    HostName 203.0.113.10
    User ec2-user
    IdentityFile ~/.ssh/ansible-lab
    IdentitiesOnly yes
```

Then use `ssh ansible-lab-ec2`. The example IP is documentation-only and must be replaced with your real public or reachable private IP. EC2 network access, security groups, server-side account, and authorized public key must also be correct.

## 10. SSH authentication and server verification

1. You invoke `ssh github-work` or Git invokes SSH for an SSH-format remote URL.
2. SSH matches `Host github-work`, connects to `HostName github.com`, and uses `User git`.
3. Your client verifies the server's host key using `known_hosts` or an approved host-key verification method. A surprising host-key change should be investigated before accepting it.
4. Your client offers the public-key identity and proves it holds the configured private key. The private key itself is not sent.
5. GitHub or GitLab maps the public key to an account and checks repository authorization. Authentication success does not guarantee access to every repository.

An SSH key passphrase protects your private key on disk. The SSH agent can hold an unlocked key for reuse; macOS Keychain can assist with the passphrase. These do not change which remote account owns the public key.

## 11. Inspect and troubleshoot SSH safely

```bash
ls -la ~/.ssh
ssh -G github-work | grep -E '^(hostname|user|port|identityfile|identitiesonly) '
ssh -T github-work
ssh -vT github-work         # Detailed connection diagnostics; review before sharing
ssh-add -l                # Keys currently loaded in the agent
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

If SSH says `Permission denied (publickey)`, verify the matching public key is registered with the intended account, the configured private key exists, and the host alias matches the Git URL. If a key is rejected due to permissions, set appropriate local permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config ~/.ssh/id_ed25519 ~/.ssh/id_ed25519_gitlab
chmod 644 ~/.ssh/id_ed25519.pub ~/.ssh/id_ed25519_gitlab.pub
```

Apply permissions only to files that actually exist. Never disable host key checking simply to suppress a mismatch. Keep private keys, access tokens, and sensitive kubeconfig material out of repositories and public messages.

## 12. Final comparison

| Action | Kubeconfig | SSH config |
| --- | --- | --- |
| Select readable shortcut | Context `myapp-dev-us-east-2` | Host alias `github-work` |
| Choose remote server | Context points to cluster `server` | Alias points to `HostName` |
| Verify remote server | Cluster CA certificate | Server host key in `known_hosts` |
| Identify client | EKS token obtained with AWS credentials | Proof using the configured SSH private key |
| Grant actions | EKS access policy and/or Kubernetes RBAC | Server account/repository permissions |
| Choose default | `current-context` | The destination in your `ssh` command or Git remote URL |

Both files simplify choosing connections, but neither file alone grants access. Always verify the selected target before running a command that changes resources.
