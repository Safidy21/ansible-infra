# ansible-infra

Rôles Ansible réutilisables pour provisionner les serveurs de mes labs.
Premier rôle : `docker`, qui installe Docker Engine et Docker Compose en
s'adaptant à la distribution et à l'architecture de chaque serveur.

## Contexte
Évolution d'une première version écrite pour mon homelab (dépôt GitLab),
réécrite pour être réutilisable sur plusieurs distributions et servir de
base à mes autres labs (monitoring-stack, mailcow-ansible-lab).

## Structure

ansible-infra/
├── ansible.cfg # configuration Ansible du projet
├── inventory/ # serveurs ciblés
├── playbooks/ # playbooks qui appliquent les rôles
└── roles/
└── docker/ # rôle d’installation de Docker


## Plateformes testées
_À venir._

## Variables
_À venir._

## Utilisation
_À venir._

## Ce que j'en retiens
_À venir._
