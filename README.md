# AnalyseSport-Football-Server

Serveur central pour l'application Android **AnalyseSport-IA**.

Collecte, normalise et stocke les donnees football a partir de sources Web publiques et legalement accessibles.

**Aucune API Football payante n'est utilisee.**

---

## Architecture

```
Application Android
       |
       v
AnalyseSport-Football-Server
       |
       v
  Collecteurs Web
       |
       v
  Normalisation
       |
       v
  Base de donnees (SQLite / PostgreSQL)
       |
       v
  API REST + WebSocket
       |
       v
  AnalyseSport-IA
```

L'application Android n'accede jamais directement aux sites sources.

---

## Prerequis

- **Node.js** >= 18.0.0
- **npm** >= 9.0.0

### Installer Node.js

```bash
# Avec nvm (recommande)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18
nvm use 18

# Ou directement depuis nodejs.org
# Telecharger et installer la version LTS
```

---

## Installation

```bash
# 1. Cloner ou decompresser le projet
cd AnalyseSport-Football-Server

# 2. Installer les dependances
npm install

# 3. Copier la configuration
#    NE JAMAIS commiter le fichier .env avec des secrets
cp .env.example .env

# 4. Adapter .env selon vos besoins
```

---

## Lancer le serveur

```bash
# Mode development (avec rechargement automatique)
npm run dev

# Mode production
npm start
```

Le serveur demarre sur **http://localhost:3000**

---

## Initialiser la base de donnees

```bash
# Creer les tables et peupler les sources
npm run db:init
npm run db:seed
```

La base SQLite est cree automatiquement dans `./data/analysesport.db` au premier demarrage.

---

## Lancer les collecteurs

```bash
# Executer tous les collecteurs actifs
npm run collector:run

# Executer un collecteur specifique
npm run collector:sofascore
npm run collector:zonstat
```

**ATTENTION** : Les collecteurs sont DESACTIVES par defaut. Voir la section "Activer une source".

---

## Tester le serveur

```bash
# Verifier que le serveur fonctionne
curl http://localhost:3000/api/health

# Liste des matchs
curl http://localhost:3000/api/matches

# Matchs du jour
curl http://localhost:3000/api/matches/today

# Matchs a venir
curl http://localhost:3000/api/matches/upcoming

# Rechercher une equipe
curl "http://localhost:3000/api/teams/search?name=Arsenal"

# Classements
curl http://localhost:3000/api/standings
```

---

## Connecter AnalyseSport-IA (Android)

Dans votre application Android, configurez l'URL de base :

```kotlin
val BASE_URL = "http://VOTRE_SERVEUR:3000"

// Exemple avec Retrofit
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

Endpoints disponibles :

| Methode | Route | Description |
|---------|-------|-------------|
| GET | `/api/health` | Statut du serveur |
| GET | `/api/matches` | Tous les matchs |
| GET | `/api/matches/live` | Matchs en direct |
| GET | `/api/matches/today` | Matchs du jour |
| GET | `/api/matches/upcoming` | Matchs a venir |
| GET | `/api/matches/:id` | Detail d'un match |
| GET | `/api/matches/:id/statistics` | Statistiques du match |
| GET | `/api/matches/:id/events` | Evenements du match |
| GET | `/api/teams` | Toutes les equipes |
| GET | `/api/teams/search?name=...` | Recherche d'equipe |
| GET | `/api/teams/:id` | Detail d'une equipe |
| GET | `/api/teams/:id/matches` | Matchs d'une equipe |
| GET | `/api/teams/:id/statistics` | Statistiques d'une equipe |
| GET | `/api/players` | Tous les joueurs |
| GET | `/api/players/:id` | Detail d'un joueur |
| GET | `/api/standings` | Classements |
| POST | `/api/gemini/:id/analyze` | Analyse Gemini d'un match |

WebSocket : `ws://VOTRE_SERVEUR:3000/ws`

---

## Deployer le serveur

### Avec Docker

```bash
# Construire et lancer
docker-compose up -d

# Voir les logs
docker-compose logs -f

# Arreter
docker-compose down
```

### Sans Docker

```bash
# Sur un serveur Linux
npm install --production
NODE_ENV=production npm start

# Avec pm2 (gestionnaire de processus)
npm install -g pm2
pm2 start src/server.js --name analysesport
pm2 save
pm2 startup
```

### PostgreSQL (production)

Pour utiliser PostgreSQL au lieu de SQLite :

```env
DB_TYPE=postgresql
DB_HOST=votre-serveur-postgresql
DB_PORT=5432
DB_NAME=analysesport
DB_USER=analysesport
DB_PASSWORD=votre-mot-de-passe-securise
```

---

## Ajouter une nouvelle source Web

1. Creer un nouveau collecteur dans `src/collectors/` :

```bash
cp src/collectors/generic.collector.js src/collectors/nouvelle-source.collector.js
```

2. Modifier la classe :

```javascript
class NouvelleSourceCollector extends BaseCollector {
  constructor() {
    super('nouvelle-source', 'https://www.nouvelle-source.com', {
      delayBetweenRequests: 5000,
    });
  }

  async collectMatchesToday() {
    // Implementer la collecte
    // Verifier robots.txt (automatique via BaseCollector)
    // Utiliser this.httpClient.get(url) pour les requetes
    // Utiliser cheerio pour le parsing HTML
    // Retourner les donnees normalisees
  }
}
```

3. Ajouter dans `src/config/index.js` :

```javascript
collectors: {
  // ... sources existantes
  'nouvelle-source': { enabled: process.env.COLLECTOR_NOUVELLE_SOURCE_ENABLED === 'true' },
},
```

4. Ajouter dans `src/database/seed.js` :

```javascript
const SEED_SOURCES = [
  // ... sources existantes
  { name: 'nouvelle-source', base_url: 'https://www.nouvelle-source.com', enabled: 0 },
];
```

5. Ajouter dans `src/collectors/runner.js` :

```javascript
const NouvelleSourceCollector = require('./nouvelle-source.collector');

// Dans _initCollectors() :
if (config.collectors['nouvelle-source'].enabled) {
  this.collectors.push(new NouvelleSourceCollector());
}
```

6. Ajouter dans `.env.example` :

```env
COLLECTOR_NOUVELLE_SOURCE_ENABLED=false
```

7. **IMPORTANT** : Verifier les conditions d'utilisation de la source et robots.txt avant d'activer le collecteur.

---

## Desactiver une source

### Via .env

```env
COLLECTOR_SOFASCORE_ENABLED=false
```

### Via la base de donnees

```sql
UPDATE sources SET enabled = 0 WHERE name = 'sofascore';
```

Le collecteur se desactive automatiquement si :
- La source retourne 403 (acces interdit)
- La source retourne 429 (rate limiting)
- robots.txt interdit la collecte
- La source est inaccessible

---

## Sources Web

Les sources actuellement configurees :

| Source | URL | Etat par defaut | Notes |
|--------|-----|-----------------|-------|
| SofaScore | sofascore.com | Desactive | Restreint l'automatisation. Verifier robots.txt |
| ZonStat | zonstat.com | Desactive | A verifier avant activation |
| Generic | (modele) | Desactive | Template pour nouvelles sources |

**REGLE ABSOLUE** : Si une source interdit clairement l'automatisation (robots.txt, conditions d'utilisation, protections anti-bot), le collecteur reste desactive.

---

## Regles importantes

1. **Aucune donnee fictive** : Le serveur ne cree jamais de faux resultats, faux joueurs, ou fausses statistiques.
2. **Aucune cle API Football** : Pas d'API-Football, SportMonks, ou API payante.
3. **Respect des sources** : robots.txt, conditions d'utilisation, limites de requetes.
4. **Donnees manquantes** : Si une donnee n'est pas disponible, retourner `null` ou "non disponible". Jamais de fabrication.
5. **Temps reel** : Une notification WebSocket n'est envoyee que si la donnee provient reellement d'une source disponible.
6. **Origine tracable** : Chaque donnee a sa source, son ID source, et ses dates de collecte/mise a jour.

---

## Licence

Projet prive – AnalyseSport-IA