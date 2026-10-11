# TD2-IaC — Industrialiser K3s avec Terraform, Ansible et Helm

TD IaC / DevSecOps (INSA Centre Val de Loire, 5A ASL-IA) : déploiement reproductible de **NovaShop**
(PostgreSQL + backend + frontend) sur une VM K3s, dans un DevStack (OpenStack).

```
Terraform  ──►  Ansible  ──►  Helm
(VM, SG, IP)    (K3s, kubeconfig)  (postgres, backend, frontend)
```

| Outil | Rôle | Dossier |
|---|---|---|
| Terraform | Crée le groupe de sécurité (22/80/443), la VM `k3s-<login>`, l'IP flottante et son association. Lit le socle partagé (`asl-net`, `public`, `asl-key`, image, flavor) via des *data sources*. | `terraform/` |
| Ansible | Prépare la VM (DNS, paquets), installe K3s, récupère le kubeconfig. | `ansible/` |
| Helm | Déploie 3 charts dans le namespace `td2` : `postgres` (StatefulSet + PVC), `backend` ×2, `frontend` ×2 + Ingress Traefik. | `helm/` |

## Arborescence

```
td2/
├── terraform/   providers.tf variables.tf data.tf security.tf cloud-init.yaml compute.tf outputs.tf example.tfvars
├── ansible/     ansible.cfg playbook.yml roles/{k3s-prereq,k3s,kubeconfig}/tasks/main.yml
└── helm/        postgres/  backend/  frontend/   (Chart.yaml, values.yaml, templates/)
```

## Utilisation (depuis l'hôte DevStack)

```bash
# 1. Infrastructure
cd terraform
cp example.tfvars terraform.tfvars      # puis renseigner son login
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.ini

# 2. Configuration de la VM
cd ../ansible
ansible-playbook playbook.yml           # relancer : changed=0 (idempotent)

# 3. Tunnel SSH vers l'API K3s (le port 6443 reste fermé dans le SG)
export FIP=$(cd ../terraform && terraform output -raw floating_ip)
ssh -f -N -L 6443:127.0.0.1:6443 ubuntu@"$FIP"
sed -i "s|server: https://$FIP:6443|server: https://127.0.0.1:6443|" kubeconfig
export KUBECONFIG=$PWD/kubeconfig

# 4. Application
cd ../helm
kubectl create namespace td2
helm install postgres ./postgres -n td2 --wait
helm install backend  ./backend  -n td2 --wait
helm install frontend ./frontend -n td2 --wait
curl -s http://$FIP/api/items
```

L'application est ensuite accessible sur `http://<IP flottante>`.
## Nettoyage

```bash
helm uninstall frontend backend postgres -n td2
kubectl -n td2 delete pvc --all && kubectl delete namespace td2
cd terraform && terraform destroy       # le socle partagé n'est jamais détruit
pkill -f "ssh.*6443:127.0.0.1:6443" || true
rm -f ../ansible/kubeconfig ../ansible/inventory.ini
```

## Sécurité

- Ne jamais versionner `*.tfstate`, `terraform.tfvars`, `inventory.ini` ni `kubeconfig` (voir `.gitignore`).
- Les mots de passe présents dans le code (`azerty`, Postgres dans `values.yaml`) sont ceux d'un **labo isolé** ; en production : Ansible Vault / Sealed Secrets.

## Auteur

LOPEZ Niels — 5A ASL-IA, INSA Centre Val de Loire
