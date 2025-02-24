## Kubernetes GitOps using Terraform/OpenTofu
- 3x Talos Control Plane VM Creation (Proxmox)
- 3x Talos Worker VM Creation (Proxmox)
- Talos Cluster configuration
- Cilium, Traefik, Longhorn, Sops, FluxCD etc.

Terraform base from [Stonegarden](https://blog.stonegarden.dev/articles/2024/08/talos-proxmox-tofu/) modified for my use.

This repo mostly follows the FluxCD Mono Repo structure.

Check my [Wiki](https://wiki.f9.casa) for more details! (WIP)

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

1. Download the repo `git clone https://github.com/fma965/f9-k8s.git`
2. Apply values to `tofu\proxmox.auto.tfvars`
3. Run `tofu plan`
4. Run `tofu apply`

Wait for the tofu plan to finish applying
(6 VM's and for the Kubernetes cluster to be configured)

5. Export required values
```bash
export KUBECONFIG=/home/scott/f9-k8s/tofu/output/kube-config.yaml
export SOPSKEY=/home/scott/f9-k8s/.sops/age/private.key
export GITHUB_TOKEN=[GITHUB_TOKEN]
```

6. `kubectl create namespace flux-system`
7. `kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=$SOPSKEY`
8. `flux bootstrap github --token-auth --owner=fma965 --repository=f9-k8s --path=clusters/home --personal`
9. `flux suspend kustomization infra-databases`
10. `flux suspend kustomization apps`

11. Using `kubectl port-forward service/longhorn-frontend :80 -n longhorn-system` port forward access to Longhorn WebUI
12. Configure longhorn backup target `s3://f9-k3s-longhorn@eu-west-2/` & `minio-longhorn-secret`
13. Restore the volumes from the backup target (UnRAID Garage S3)
14. Once restore has completed run the following commands
```bash
kubectl rollout restart deployment/mariadb -n mariadb
kubectl rollout restart deployment/postgresql -n postgresql
```
15. Finally run the following commands
```bash
flux resume kustomization infra-databases
flux resume kustomization apps
```