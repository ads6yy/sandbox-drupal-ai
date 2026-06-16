# Sandbox Drupal AI

Ce dépôt est un **bac à sable** pour expérimenter la suite **AI de Drupal** en conditions réelles : assistance éditoriale, enrichissement automatique des médias et un **chatbot RAG** (« IpsoBot ») qui répond à partir du contenu du site.

Le corpus de démo met en scène un intranet RH fictif (**Novelia Conseil**). Tout est piloté par **config exportée** et **variables d'environnement** — aucun secret n'est versionné.

![Drupal](https://img.shields.io/badge/drupal-%230678BE.svg?style=for-the-badge&logo=drupal&logoColor=white)
![PHP](https://img.shields.io/badge/php-%23777BB4.svg?style=for-the-badge&logo=php&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![MariaDB](https://img.shields.io/badge/mariadb-%23003545.svg?style=for-the-badge&logo=mariadb&logoColor=white)
![Nginx](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)

## 🛠️ Tech Stack

* **CMS :** Drupal Core 11
* **Langage :** PHP 8.4
* **Base de données :** MariaDB 12 (recherche vectorielle native — type `VECTOR` + index HNSW)
* **Serveur web :** Nginx
* **Environnement :** Docker Compose
* **Dépendances :** Composer
* **IA :** module `drupal/ai` + provider **Gemini** (chat, vision, embeddings)
* **Observabilité :** Langfuse
* **Thème admin :** Gin

## 🚀 Quick Start

### Prérequis

* [Docker](https://docs.docker.com/get-docker/) et [Docker Compose](https://docs.docker.com/compose/install/) installés
* Une **clé API Google Gemini** (pour les fonctions IA)

### Démarrage

```bash
# Copier et remplir le fichier d'environnement (au minimum GEMINI_API_KEY)
cp deploy/.env.example deploy/.env

# Construire et lancer la stack (MariaDB + PHP-FPM + Nginx)
cd deploy && docker compose up -d --build

# Installer les dépendances PHP
docker compose exec php composer install

# Installer Drupal à partir de la configuration versionnée
docker compose exec php vendor/bin/drush site:install --existing-config --yes

# Vider le cache
docker compose exec php vendor/bin/drush cache:rebuild
```

Le site est accessible sur `http://localhost:8703`.

```bash
# Obtenir un lien de connexion admin
docker compose exec php vendor/bin/drush user:login
```

> ℹ️ `site:install --existing-config` reconstruit le site (providers IA, index RAG, assistant IpsoBot, médias…) à partir de `config/sync/`. La connexion base et les réglages sensibles sont lus depuis l'environnement — voir ci-dessous.

### Commandes Drush utiles

```bash
# Exporter la config après modifications dans l'interface
docker compose exec php vendor/bin/drush config:export --yes

# Réindexer le corpus RAG (recherche sémantique)
docker compose exec php vendor/bin/drush search-api:index rh_corpus

# Activer un nouveau module
docker compose exec php composer require drupal/<module>
docker compose exec php vendor/bin/drush pm:enable --yes <module>
docker compose exec php vendor/bin/drush cache:rebuild
```

## 🔑 Variables d'environnement

Définies dans `deploy/.env` (modèle : `deploy/.env.example`) et injectées dans le conteneur PHP. `settings.php` les lit — **rien n'est codé en dur ni versionné**.

| Variable | Rôle | Requis |
|:---------|:-----|:------:|
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` / `DB_HOST` | Connexion MariaDB | ✅ |
| `DB_ROOT_PASSWORD` | Mot de passe root MariaDB (init conteneur) | ✅ |
| `HASH_SALT` | Sel de hachage Drupal (sessions, tokens) | recommandé |
| `GEMINI_API_KEY` | Clé du provider IA Gemini | ✅ (IA) |
| `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` / `LANGFUSE_BASE_URL` | Observabilité Langfuse | optionnel |

> La clé Gemini est stockée côté Drupal via le module **Key** (provider « environment ») : la config exportée ne contient qu'une *référence* à la variable, jamais la valeur.

## 📁 Structure du projet

```
├── composer.json / composer.lock   # Drupal core + modules contrib
├── config/sync/                     # Export complet de la configuration (YAML)
├── deploy/                          # Stack Docker Compose
│   ├── docker-compose.yml
│   ├── db/                          # Config MariaDB
│   ├── nginx/                       # Config Nginx
│   └── php/                         # Dockerfile + php.ini
└── web/
    ├── sites/default/settings.php   # 100 % env-based, versionné, sans secret
    └── modules/custom/
        └── sandbox_tweaks/          # Ajustements UI (z-index chatbot, token CSRF…)
```

## 🤖 Fonctionnalités IA

| Brique | Modules | Rôle |
|:-------|:--------|:-----|
| **Cœur + provider** | `ai`, `gemini_provider`, `key` | Abstraction des opérations IA, clés sécurisées |
| **Éditorial** | `ai_ckeditor`, `ai_content_suggestions` | Générer / reformuler / résumer ; suggestions de titre, ton, lisibilité |
| **Traduction** | `ai_translate` | Traduction de contenu en un clic (HTML préservé) |
| **Médias** | `ai_automators`, `ai_image_alt_text` | Tags, descriptions et alt text auto par vision |
| **RAG** | `search_api`, `ai_search`, `ai_vdb_provider_mariadb` | Recherche sémantique, vecteurs stockés dans MariaDB |
| **Chatbot** | `ai_assistant_api`, `ai_chatbot` | Assistant « IpsoBot » + widget de chat |
| **Observabilité** | `langfuse` | Traces, tokens, coûts de chaque appel IA |

### Le RAG en bref

Le contenu est découpé en *chunks*, vectorisé par Gemini (`gemini-embedding-001`, 3072 dim) et stocké dans **MariaDB** (type `VECTOR` natif + index HNSW — pas de base vectorielle externe). À chaque question, IpsoBot recherche les passages les plus proches par le *sens* puis rédige une réponse **sourcée**, avec un prompt anti-hallucination. Détails techniques : voir la documentation locale (ci-dessous).

## 📚 Documentation

* [Module Drupal AI](https://www.drupal.org/project/ai)
* [MariaDB Vector](https://mariadb.org/projects/mariadb-vector/)
* [Langfuse](https://langfuse.com/docs)
* [Drupal User Guide](https://www.drupal.org/docs/user_guide/en/index.html)

> 📁 Les supports de présentation et la documentation technique approfondie (RAG, schémas, notes) vivent dans `docs/` (dossier **local, non versionné**).

## 📅 Roadmap

| Fonctionnalité | Description | Statut |
|:---------------|:------------|:-------|
| **Éditorial augmenté** | CKEditor IA + suggestions de contenu | ✅ Fait |
| **Traduction IA** | `ai_translate` multilingue (FR/EN) | ✅ Fait |
| **Médias intelligents** | Tags, descriptions, alt text par vision | ✅ Fait |
| **RAG + IpsoBot** | Recherche sémantique MariaDB + chatbot sourcé | ✅ Fait |
| **Observabilité** | Traces Langfuse (Cloud EU) | ✅ Fait |
| **Transcription audio** | Speech-to-text sur médias audio | ⏳ À faire |
| **Workflows ECA** | IA déclenchée par événements (modération, traduction auto) | ⏳ À faire |
| **Self-hosted** | Provider local (Ollama) + Langfuse self-hosted | ⏳ À faire |

## 📄 Licence

Ce projet est sous licence [GNU General Public License v2.0 ou ultérieure](http://www.gnu.org/licenses/old-licenses/gpl-2.0.html).

Drupal et son logo sont des marques déposées. Voir la [politique des marques Drupal](https://www.drupal.com/trademark).
