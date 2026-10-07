# otel-agent

Configuration Docker Compose pour déployer un agent OpenTelemetry Collector (`otel-collector`) afin d'instrumenter une machine hôte et l'ensemble de ses conteneurs Docker.

Ce projet permet de collecter les métriques système et Docker ainsi que les logs des conteneurs, de les enrichir avec des métadonnées contextuelles (environnement, hôte, labels de conteneurs), puis de les exporter vers un serveur OpenTelemetry central.

---

## 📋 Prérequis

- **Docker** et **Docker Compose**
- Droits de lecture sur le socket Docker (`/var/run/docker.sock`)
- Droits de lecture sur le répertoire des logs de conteneurs (`/var/lib/docker/containers`)
- Connaître le GID du groupe `docker` sur la machine hôte pour autoriser l'agent à s'y connecter.

---

## 🚀 Installation & Démarrage

1. **Récupérer le dépôt** :

   ```bash
   git clone https://github.com/abes-esr/otel-agent.git
   cd otel-agent
   ```

2. **Configurer l'environnement** :
   Copiez le fichier d'exemple `.env-dist` sous le nom `.env` :

   ```bash
   cp .env-dist .env
   ```

   Éditez ensuite le fichier `.env` pour l'adapter à votre cible de déploiement (voir la section [Configuration](#⚙️-configuration-env) ci-dessous).

3. **Lancer l'agent** :
   ```bash
   docker compose up -d
   ```

---

## ⚙️ Configuration (.env)

Le fichier `.env` pilote la configuration du Docker Compose et de l'agent. Voici les variables majeures à renseigner :

| Variable                           | Description                                                                            | Exemple / Valeur par défaut                        |
| :--------------------------------- | :------------------------------------------------------------------------------------- | :------------------------------------------------- |
| `MEM_LIMIT`                        | Limite de mémoire vive allouée au conteneur.                                           | `5g`                                               |
| `CPU_LIMIT`                        | Limite de processeur (nombre de cœurs) allouée au conteneur.                           | `5`                                                |
| `OTEL_ENVIRONMENT`                 | Environnement d'exécution (ex: `local`, `dev`, `test`, `prod`).                        | `"local"`                                          |
| `OTEL_EXPORTER_OTLP_ENDPOINT_GRPC` | URL gRPC du collecteur OpenTelemetry central.                                          | `http://diplotaxis7-dev.v212.abes.fr:4317`         |
| `OTEL_DOCKER_GROUP_ID`             | Le GID du groupe `docker` sur la machine hôte. (Essentiel pour lire le socket Docker). | _À récupérer avec `cat /etc/group \| grep docker`_ |
| `OTEL_METRICS_EXPORTER`            | Type d'exportateur pour les métriques.                                                 | `otlp`                                             |
| `OTEL_LOGS_EXPORTER`               | Type d'exportateur pour les logs.                                                      | `otlp`                                             |
| `OTEL_TRACES_EXPORTER`             | Type d'exportateur pour les traces.                                                    | `otlp`                                             |

---

## 🛠️ Architecture & Collecte

L'agent s'appuie sur l'image officielle `otel/opentelemetry-collector-contrib`. Sa configuration est définie dans [volumes/otel-collector/config.yaml](file:///c:/Projets/otel-agent/volumes/otel-collector/config.yaml).

### 1. Récepteurs (`receivers`) & Extensions

- **`docker_stats`** : Se connecte à `/var/run/docker.sock` et extrait les métriques des conteneurs. Il extrait également des variables d'environnement utiles (`OTEL_VERSION`, `OTEL_SERVICE_NAME`, `SPRING_PROFILES_ACTIVE`, `OTEL_NAMESPACE`) ainsi que des labels Docker Compose (`com.docker.compose.service`, `com.docker.compose.project`) pour les associer en tant que labels de métriques.
- **`docker_observer` & `receiver_creator` (avec `filelog`)** : Observe les conteneurs Docker via le socket et génère dynamiquement un récepteur `filelog` pour chaque conteneur actif afin de lire les logs JSON dans `/var/lib/docker/containers/`.
- **`host_metrics`** : Collecte périodiquement (toutes les 30s) les métriques physiques de la machine hôte : CPU, mémoire, disque et réseau.

### 2. Processeurs (`processors`)

- **`resource/containers`** : Associe le nom du conteneur en tant que `service.name` et injecte l'environnement (`deployment.environment`) ainsi que le nom de la machine (`host.name`).
- **`transform/containers`** : Normalise le nom du conteneur et du service en retirant le slash initial (`/`) issu de l'API Docker.
- **`resource/host`** : Associe le nom de l'hôte aux attributs globaux pour les métriques de la machine elle-même.
- **`batch`** : Regroupe les métriques et logs avant envoi pour réduire la charge réseau et optimiser les performances.

### 3. Exportateurs (`exporters`)

- **`otlp`** : Exporte les données collectées (métriques et logs) en gRPC (insecure) vers l'endpoint configuré (`OTEL_EXPORTER_OTLP_ENDPOINT_GRPC`).

---

## 📈 Supervision & Maintenance

- **Vérifier l'état de l'agent** :
  ```bash
  docker compose ps
  ```
- **Consulter les journaux (logs) de l'agent** :
  ```bash
  docker compose logs -f --tail=100
  ```
- **Arrêter l'agent** :
  ```bash
  docker compose down
  ```

---

## ⚖️ Licence

Ce projet est distribué sous la licence libre **CeCILL Version 2.1**.
Consultez le fichier [LICENSE](file:///c:/Projets/otel-agent/LICENSE) pour en savoir plus sur vos droits d'utilisation, de modification et de distribution.
