# docs/installation.md — Mise en place (révision : machine unique + Docker)

> Révision du 30/08/2026 — remplace la procédure multi-VM initiale de la Phase 1. Voir `docs/network.md` pour l'architecture.

## 1. Prérequis sur la machine Ubuntu 24.04 unique

- 8 Go de RAM minimum (Wazuh Indexer est le composant le plus gourmand), 4 vCPU, 60 Go d'espace disque libre.
- Docker Engine + plugin Compose (`docker compose version` doit fonctionner) :
  ```bash
  curl -fsSL https://get.docker.com | sudo sh
  sudo usermod -aG docker "$(whoami)"
  ```
- Accès Internet sortant (pour les paquets Wazuh/Suricata et les images de base Docker) — peut être retiré une fois l'installation terminée si une isolation complète est souhaitée pour les scénarios de démonstration (le trafic entre l'hôte et les conteneurs `endpoint-linux`/`webapp`/`attacker` ne nécessite aucun accès Internet).

## 2. Installation des services natifs du SOC

Suivre dans l'ordre les phases déjà documentées, **sans changement** par rapport à la version initiale (elles ciblaient déjà une seule machine, `soc-server`) :

1. `docs/wazuh.md` (Phase 2) — Wazuh Manager + Indexer + Dashboard.
2. `docs/suricata.md` (Phase 3) — **seule différence** : l'interface de capture devient le pont Docker `br-adaptivesoc` (créé à l'étape 3 ci-dessous) au lieu d'une interface VirtualBox — voir `docs/suricata.md` §2 mis à jour.
3. `docs/database.md` (Phase 14) — PostgreSQL.
4. `docs/backend.md` (Phase 13) — API FastAPI.
5. `docs/dashboard.md` (Phase 12) — Dashboard Next.js.
6. `docs/soc-security.md` (Phase 15) — TLS, nginx, secrets, backup.

## 3. Mise en place des conteneurs (cibles + attaquant)

```bash
cd infrastructure/docker
docker compose -f lab-topology.docker-compose.yml build
docker compose -f lab-topology.docker-compose.yml up -d
```

Cela crée le réseau `adaptive-soc-net` (172.20.0.0/24, pont `br-adaptivesoc`) et démarre les 3 conteneurs. Vérifier :

```bash
docker compose -f lab-topology.docker-compose.yml ps
docker network inspect adaptive-soc-net | grep -A2 '"Name": "br-adaptivesoc"'
ip a show br-adaptivesoc   # doit exister côté hôte, c'est l'interface que Suricata capturera
```

## 4. Enregistrement des agents Wazuh

Les images `endpoint-linux` et `webapp` installent déjà l'agent Wazuh au moment du build (`WAZUH_MANAGER=172.20.0.1`, la passerelle du pont Docker qui pointe vers l'hôte). Vérifier l'enregistrement depuis l'hôte :

```bash
sudo /var/ossec/bin/agent_control -l
```

Les deux agents (`endpoint-linux`, `webapp`) doivent apparaître. En cas d'échec (`Never connected`), vérifier que le Wazuh Manager écoute bien sur toutes les interfaces (pas uniquement `127.0.0.1`) pour recevoir les connexions entrantes depuis `172.20.0.0/24`.

## 5. Utilisation du conteneur attaquant

```bash
cd infrastructure/docker
docker compose -f lab-topology.docker-compose.yml exec attacker bash
# puis, depuis l'intérieur du conteneur :
nmap -sS -p 1-1000 172.20.0.21
hydra -l testuser -P /usr/share/wordlists/common-passwords.txt ssh://172.20.0.21
```

## 6. Résultats attendus

- `docker compose ps` montre les 3 conteneurs `Up`.
- `agent_control -l` montre `endpoint-linux` et `webapp` en statut `Active`.
- Un scan `nmap` depuis `attacker` déclenche une alerte Suricata (Phase 3) puis Wazuh (règle 100050, Phase 2) en quelques secondes.

## 7. Erreurs potentielles à surveiller

- **Le pont `br-adaptivesoc` n'apparaît pas côté hôte** : certains environnements Docker (rootless, ou drivers réseau alternatifs) ignorent `driver_opts.com.docker.network.bridge.name` — vérifier avec `docker network inspect`, et à défaut utiliser le nom de pont réellement attribué dans la configuration Suricata (§3 de `docs/suricata.md`).
- **Agents jamais enregistrés** : le pare-feu de l'hôte (UFW) doit autoriser les ports 1514/1515 depuis `172.20.0.0/24`, pas seulement depuis l'ancien `10.10.10.0/24` — `infrastructure/firewall/ubuntu-soc-ufw.sh` a été mis à jour en conséquence.
- **Build Docker échoue sur le téléchargement de l'image de base** : vérifier l'accès Internet sortant de l'hôte (nécessaire uniquement au moment du build, pas à l'exécution).
