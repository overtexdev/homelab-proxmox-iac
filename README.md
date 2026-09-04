# Homelab Proxmox - Infrastructure as Code

Provisionnement et configuration **entièrement automatisés** d'une plateforme d'infrastructure auto-hébergée, décrite en code avec **Ansible**. Ce dépôt transforme un hyperviseur Proxmox nu en une plateforme complète de supervision et d'intégration/déploiement continu, reproductible en quelques commandes.

Projet personnel pour approfondir l'administration système/réseau, le DevOps et les pratiques d'Infrastructure as Code.

---

## Aperçu

Le homelab s'appuie sur deux machines physiques :

- **Raspberry Pi** - héberge des services personnels en production (reverse proxy, DNS, gestionnaire de mots de passe, cloud personnel).
- **Proxmox VE** - hyperviseur qui héberge les conteneurs LXC de la plateforme d'infrastructure décrite ici.

Les deux machines sont interconnectées via un réseau mesh **Tailscale**, Proxmox jouant le rôle de *subnet router*.

L'objectif : ne plus jamais configurer cette infrastructure à la main. Tout est déclaré dans ce dépôt ; une commande suffit à la reconstruire à l'identique après un incident ou une réinstallation.

---

## Architecture

L'hyperviseur Proxmox héberge quatre conteneurs LXC, chacun avec un rôle précis :

| LXC | Hôte | Rôle | Détails |
|-----|------|------|---------|
| 100 | `uptime-kuma` | Supervision de disponibilité | Surveillance des endpoints HTTP des services (préexistant) |
| 101 | `prometheus-grafana` | Métriques & visualisation | Prometheus (collecte) + Grafana (dashboards) |
| 102 | `docker-registry` | Registre d'images privé | `registry:2`, stockage des images Docker internes |
| 103 | `ci-runner` | CI/CD | Runner GitHub Actions self-hosted |

La supervision couvre les deux machines physiques : un `node_exporter` est installé sur le Raspberry Pi **et** sur l'hôte Proxmox, tous deux scrapés par Prometheus et visualisés dans un dashboard Grafana unique.

---

## Stack technique

- **Ansible** - provisionnement et configuration (rôles idempotents, Ansible Vault pour les secrets)
- **Proxmox VE** - virtualisation par conteneurs LXC
- **Docker & Docker Compose** - exécution des services
- **Prometheus / Grafana / node_exporter** - observabilité
- **GitHub Actions (runner self-hosted)** - intégration et déploiement continus
- **Registry Docker privé** - distribution des images internes
- **Tailscale** - réseau mesh entre les machines

---

## Structure du dépôt

```
homelab-proxmox-iac/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── proxmox.yml         # Secrets chiffrés (Ansible Vault)
├── roles/
│   ├── lxc_provision/          # Création des conteneurs LXC sur Proxmox
│   ├── docker_setup/           # Installation de Docker dans les conteneurs
│   ├── registry/               # Déploiement du registre Docker privé
│   ├── monitoring/             # Stack Prometheus + Grafana
│   ├── node_exporter/          # Agent de métriques (Pi + Proxmox)
│   ├── github_runner/          # Runner GitHub Actions self-hosted
│   └── docker_insecure_registry/
└── playbooks/
    ├── provision_lxc.yml
    ├── setup_docker.yml
    ├── deploy_registry.yml
    ├── deploy_monitoring.yml
    ├── deploy_node_exporter.yml
    ├── deploy_runner.yml
    └── configure_insecure_registry.yml
```

---

## Prérequis

- Une machine de contrôle avec **Ansible** (testé sous WSL Ubuntu 24.04)
- Collections : `community.general`, `community.proxmox`, `community.docker`
- Bibliothèques Python : `proxmoxer` (>= 2.3), `requests`
- Un hôte **Proxmox VE** accessible, avec un **token API** dédié
- Accès SSH par clé aux conteneurs et aux hôtes cibles

```bash
ansible-galaxy collection install community.general community.proxmox community.docker
pip install proxmoxer requests
```

---

## Utilisation

Les secrets sont chiffrés avec Ansible Vault. Le fichier `.vault_pass` et les clés SSH ne sont **pas** versionnés.

```bash
# 1. Provisionner les conteneurs LXC sur Proxmox
ansible-playbook playbooks/provision_lxc.yml

# 2. Installer Docker dans les conteneurs
ansible-playbook playbooks/setup_docker.yml

# 3. Déployer les services
ansible-playbook playbooks/deploy_registry.yml
ansible-playbook playbooks/deploy_monitoring.yml

# 4. Installer les agents de métriques (mot de passe sudo requis sur le Pi)
ansible-playbook playbooks/deploy_node_exporter.yml -K

# 5. Autoriser le registry HTTP interne (runner + Pi)
ansible-playbook playbooks/configure_insecure_registry.yml -K

# 6. Déployer le runner CI/CD (token d'enregistrement passé en variable)
ansible-playbook playbooks/deploy_runner.yml -e "runner_token=XXXX"
```

Tous les playbooks sont **idempotents** : les rejouer ne recrée rien inutilement.

---

## Pipeline CI/CD

Le runner self-hosted permet une boucle de déploiement continue complète, illustrée dans le dépôt compagnon [`demo-api`](https://github.com/overtexdev/demo-api) :

1. Un `git push` déclenche le workflow GitHub Actions.
2. Le runner (LXC 103) **build** l'image Docker (build `arm64` via `buildx` + QEMU, pour cibler le Raspberry Pi).
3. L'image est **poussée** sur le registre privé (LXC 102).
4. Le runner se connecte **en SSH** au Raspberry Pi (clé dédiée en GitHub Secret).
5. Le Pi **tire** la nouvelle image et **redéploie** le conteneur.
6. Un **health check** valide le déploiement.

Le tout sans intervention manuelle, à partir d'un simple commit.

---

## Choix techniques & points d'attention

- **Idempotence** - chaque rôle peut être rejoué sans effet de bord.
- **Gestion des secrets** - token API Proxmox chiffré avec Ansible Vault ; clé de déploiement SSH isolée en GitHub Secret ; aucun secret en clair.
- **Build multi-architecture** - le runner est en `x86_64`, la cible (Pi) en `arm64` ; le pipeline build explicitement pour `linux/arm64`.
- **Registre HTTP interne** - le registre fonctionne en HTTP sur le réseau local ; les démons Docker et le builder BuildKit sont configurés pour l'accepter. En production, ce registre serait servi en HTTPS.
- **Contraintes matérielles** - allocations mémoire des LXC dimensionnées pour un hôte à ressources limitées.

---

## Améliorations envisagées

- Servir le registre privé en HTTPS.
- Rendre la configuration DNS des conteneurs entièrement déclarative.
- Ajouter un lint Ansible (`ansible-lint`) dans la CI.
- Étendre la supervision (alerting Grafana, sondes applicatives).

---

## Auteur

**overtex** - étudiant en BUT Informatique

Portfolio : [portfolio.overtex-vault.duckdns.org](https://portfolio.overtex-vault.duckdns.org) · GitHub : [@yvlovertex](https://github.com/yvlovertex)
