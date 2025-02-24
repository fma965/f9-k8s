## Kubernetes GitOps using Terraform/OpenTofu
- 3x Talos Control Plane VM Creation (Proxmox)
- 3x Talos Worker VM Creation (Proxmox)
- Talos Cluster configuration
- Cilium, Traefik, Longhorn, Sops, FluxCD etc.

Terraform base from [Stonegarden](https://blog.stonegarden.dev/articles/2024/08/talos-proxmox-tofu/) modified for my use.

This repo mostly follows the FluxCD Mono Repo structure.

### Secrets
All `secret.enc.yaml` and `*-secret.enc.yaml` files are encrypted with SOPS using AGE.

I have included `secret.sample.yaml` and `*-secret.sample.yaml` files which contain placeholder values to enhance the usability.

Set your GitHooks to use `.githooks` with this command
`git config --local core.hooksPath .githooks/`
This makes sure any filename with `secret.yaml` in it's name is encrypted with SOPS

[AGE](https://github.com/FiloSottile/age) keys for SOPS to use need to be saved in the following structure from the root of this repo.
`.sops/age/private.key` 
`.sops/age/public.key` 

Windows users download [SOPS](https://github.com/getsops/sops/releases/latest) and place it somewhere in your `PATH` (e.g. C:\Windows) otherwise secret encryption will not work

Linux users follow the instructions at [SOPS](https://github.com/getsops/sops/releases/latest) to install SOPS to `/usr/local/bin/sops`

### Restore Steps (with existing volumes)
Ensure you have fluxcli, kubectl, talosctl and sops installed in a WSL or Linux installation

1. Download the repo `git clone https://github.com/fma965/f9-k8s.git`
2. Apply values to `tofu\proxmox.auto.tfvars`
3. Run `tofu init`
4. Run `tofu plan`
5. Run `tofu apply` and wait for completion
6. Export required values
```bash
export KUBECONFIG=/home/scott/f9-k8s/tofu/output/kube-config.yaml
export SOPSKEY=/home/scott/f9-k8s/.sops/age/private.key
export GITHUB_TOKEN=[GITHUB_TOKEN]
```

7. Run the following commands
```bash
kubectl create namespace flux-system`
kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=$SOPSKEY
```

8. Bootstrap the repo with FluxCD
```bash
flux bootstrap github --token-auth --owner=fma965 --repository=f9-k8s --path=clusters/home --personal
flux suspend kustomization infra-databases
flux suspend kustomization apps
```

9. Using `kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80` port forward access to Longhorn WebUI
10. Open your browser and navigate to http://localhost:8080.

11. In the UI, Click on "Backups", Select all volumes, "Restore from last backup".
    
12. Once restore has completed run the following commands
```bash
kubectl rollout restart deployment/mariadb -n mariadb
kubectl rollout restart deployment/postgresql -n postgresql
```
13. Finally run the following commands to resume the FluxCD deployment
```bash
flux resume kustomization infra-databases
flux resume kustomization apps
```

### Footnotes
Check my [Wiki](https://wiki.f9.casa) for more details! (WIP)