cat > README.md << 'EOF'
# ansible-infra

[![CI](https://github.com/Safidy21/ansible-infra/actions/workflows/ci.yml/badge.svg)](https://github.com/Safidy21/ansible-infra/actions/workflows/ci.yml)

Rôles Ansible réutilisables pour provisionner les serveurs de mes labs.
Premier rôle : **`docker`**, qui installe Docker Engine et Docker Compose
depuis le dépôt officiel, en s'adaptant automatiquement à la distribution
et à l'architecture de chaque serveur.

## Contexte

Dans mon homelab, j'avais écrit une première version de ce rôle
([dépôt GitLab](https://gitlab.com/homelab8362625/docker)), déployée par
GitLab CI sur des VMs Debian 12. Elle fonctionnait, mais restait liée à
un seul OS et à une architecture codée en dur.

Ce dépôt en est la réécriture, avec trois objectifs :
- **réutilisable** : un seul rôle pour plusieurs distributions, sans valeur propre à une machine ;
- **idempotent** : une deuxième exécution ne modifie rien (`changed=0`) ;
- **vérifié** : linting automatique à chaque Pull Request.

Il sert de base à mes autres labs (`monitoring-stack`, `mailcow-ansible-lab`).

## Architecture du lab

```
                         GitHub
              ┌────────────────────────────┐
  Pull Request│  GitHub Actions (CI)        │
  ───────────►│  yamllint → syntax-check →  │──► ✅ fusion autorisée dans main
              │  ansible-lint (production)  │
              └────────────────────────────┘

  VMware Workstation (réseau NAT)
  ┌───────────────────────────┐      SSH (clé dédiée)     ┌──────────────────────┐
  │ lab-srv                   │ ────────────────────────► │ lab-node01           │
  │ Debian 13 (trixie)        │                           │ Ubuntu 24.04 (noble) │
  │ control node Ansible      │                           │ cible distante       │
  │ + cible (connexion local) │                           └──────────────────────┘
  └───────────────────────────┘
```

## Lancer le lab

Prérequis sur le control node : `ansible-core`, un accès SSH par clé aux
cibles, et un utilisateur avec sudo sur chaque cible.

```bash
git clone https://github.com/Safidy21/ansible-infra.git && cd ansible-infra
ansible all -m ping
ansible-playbook playbooks/docker.yml
```

Pour l'utiliser sur vos propres serveurs : adaptez `inventory/hosts.ini`,
et remplacez les fichiers `inventory/host_vars/*/vault.yml` (chiffrés avec
mon mot de passe) par les vôtres, créés avec `ansible-vault create`.

## Ce que fait le rôle

1. Vérifie que la famille d'OS et l'architecture sont supportées (`assert`), sinon s'arrête proprement.
2. Charge les variables et les tâches de la famille d'OS détectée (`vars/<famille>.yml`, `setup-<famille>.yml`).
3. Supprime les paquets non officiels qui entrent en conflit (`docker.io`, `podman-docker`…).
4. Installe la clé GPG de Docker et génère la définition du dépôt APT (format deb822) à partir des faits de la machine.
5. Rafraîchit le cache APT uniquement si le dépôt a changé (handler + `flush_handlers`).
6. Installe Docker Engine, la CLI, containerd, Buildx et le plugin Compose.
7. Démarre et active le service, puis ajoute les utilisateurs choisis au groupe `docker`.

## Plateformes

| OS | Version | Architecture | Statut |
|---|---|---|---|
| Debian | 13 (trixie) | x86_64 | ✅ testé |
| Ubuntu | 24.04 (noble) | x86_64 | ✅ testé |
| Rocky / Alma / RHEL | 9 | x86_64 | ⏳ tâches préparées, non testées, non activées |

## Variables

| Variable | Défaut | Rôle |
|---|---|---|
| `docker_packages` | docker-ce, cli, containerd, buildx, compose | Paquets installés |
| `docker_repo_channel` | `stable` | Canal du dépôt Docker (`stable` ou `test`) |
| `docker_users` | `[]` | Utilisateurs ajoutés au groupe `docker` (équivaut à un accès root) |
| `docker_service_state` | `started` | État du service |
| `docker_service_enabled` | `true` | Démarrage automatique au boot |
| `docker_arch_map` | x86_64→amd64, aarch64→arm64 | Traduction architecture noyau → APT |

Dans ce lab, `inventory/group_vars/all.yml` ajoute au groupe `docker`
l'utilisateur de connexion Ansible (`ansible_user`).

## Structure

```
ansible-infra/
├── ansible.cfg                  # inventaire et chemin des rôles
├── .github/workflows/ci.yml     # CI : yamllint, syntax-check, ansible-lint
├── inventory/
│   ├── hosts.ini                # machines, groupées par OS
│   ├── group_vars/all.yml       # variables communes
│   └── host_vars/<machine>/     # vars.yml (clair) + vault.yml (chiffré)
├── playbooks/
│   └── docker.yml
└── roles/docker/
    ├── defaults/main.yml        # variables modifiables
    ├── vars/                    # main.yml, Debian.yml, RedHat.yml
    ├── tasks/                   # main.yml, setup-Debian.yml, setup-RedHat.yml
    ├── handlers/main.yml
    ├── templates/docker.sources.j2
    └── meta/main.yml
```

## Sécurité

- Aucune IP dans le dépôt : les noms d'hôtes sont résolus par `~/.ssh/config` sur le control node.
- Mots de passe sudo chiffrés avec Ansible Vault (AES-256), pattern `vars.yml` / `vault.yml`.
- Mot de passe Vault hors du dépôt : variable `ANSIBLE_VAULT_PASSWORD_FILE` en local, secret GitHub en CI.

## Ce que j'en retiens

Ansible et ses principes (idempotence, rôles, Vault) m'étaient familiers. Ce projet m'a surtout fait découvrir le linting Ansible et GitHub Actions.

- **Trois vérifications, trois niveaux** : yamllint contrôle la forme du YAML, `--syntax-check` vérifie qu'Ansible comprend la structure, ansible-lint
  applique les bonnes pratiques. Je les lance en local avant de pousser, et la CI rejoue exactement les mêmes.
- **ansible-lint m'a fait corriger du code qui fonctionnait** : en profil `production`, la règle `no-handler` a signalé que mon `register` +
  `when: is changed` réimplémentait un handler. Je l'ai remplacé par un handler suivi de `meta: flush_handlers`, car les handlers s'exécutent
  par défaut en fin de play, alors que l'installation des paquets dépendait du cache APT à jour.
- **Ces outils lisent leur configuration depuis le dossier courant** : lancé depuis `.github/workflows/`, ansible-lint ignorait mon `.yamllint`
  et prenait la clé `on:` du workflow pour un booléen. C'est la même cause que l'inventaire « non chargé » d'Ansible : je lance toujours depuis la racine.
- **GitHub Actions, comparé à GitLab CI** : mêmes concepts (déclencheurs, jobs, runners, secrets), mais plusieurs workflows dans `.github/workflows/`,
  des jobs parallèles par défaut que l'on ordonne avec `needs`, et surtout un code qui n'est pas cloné automatiquement : `actions/checkout` est obligatoire.
EOF
