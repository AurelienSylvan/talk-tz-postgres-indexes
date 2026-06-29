# PostgreSQL Indexes — Notes de préparation

Un guide complet sur les index PostgreSQL : fonctionnement interne, types, stratégie de création et monitoring.

---

## Table des matières

1. [Rappels fondamentaux](#rappels-fondamentaux)
2. [MVCC : Multi-Version Concurrency Control](#mvcc)
3. [VACUUM : nettoyage des dead tuples](#vacuum)
4. [Le Query Planner](#query-planner)
5. [Les types d'index](#types-dindex)
6. [Index avancés et patterns](#index-avancés)
7. [Coûts et maintenance](#coûts-et-maintenance)
8. [Monitoring et diagnostics](#monitoring)
9. [Démos en live](#démos)
10. [Cheat-sheet](#cheat-sheet)

---

## Rappels fondamentaux

### Architecture disque : Heap files et pages

PostgreSQL stocke **toutes les données** dans des fichiers **heap** sur disque.

- **Heap file** : ensemble de **pages** de **8 KB** chacune (taille fixe depuis les débuts de PG)
- **Page** : contient plusieurs **tuples** (lignes), un header, et une Free Space Map
- **Tuple** : une ligne avec ses colonnes, headers internes (infos MVCC)
- **ctid** : identifiant physique unique d'un tuple = `(page_number, position_in_page)`. C'est ce que les index utilisent pour pointer vers une ligne.

```sql
-- Vérifier la taille physique d'une table
SELECT pg_size_pretty(pg_total_relation_size('orders')) AS table_size;

-- Voir la taille détaillée (heap + tous les indexes)
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

### Mémoire partagée : Shared buffers et write-ahead log

- **Shared buffers** (~25 % de la RAM) : cache des pages du heap et des index en mémoire
- **WAL (Write-Ahead Log)** : journal séquentiel sur disque. Tout changement est d'abord écrit au WAL avant d'être appliqué en mémoire.
- **Checkpointer** : process qui périodiquement force les pages modifiées du shared buffer vers le disque (« checkpoint »)

**Implication pour les index** : les modifications d'un index suivent la même protection WAL. Crash = aucun perte de données, mais les index peuvent avoir besoin d'un `REINDEX` dans de très rares cas.

### Concept de ctid

Un **ctid** (transaction ID, mais ici tuple ID) est l'**adresse physique** d'une ligne :
- Format : `(blockid, offsetid)` → ex. `(0, 1)` = page 0, position 1
- **Immuable** : une fois écrite, la position physique d'un tuple ne change pas (sauf VACUUM FULL qui recompresse)
- **C'est ce qu'un index stocke** : pas la ligne entière, juste le ctid. Au moment de la requête, PG doit **« fetcher la ligne du heap »** en utilisant ce ctid.

---

## MVCC : Multi-Version Concurrency Control

PostgreSQL implémente **MVCC** : plusieurs versions d'une même ligne peuvent coexister en mémoire/disque. Chaque transaction voit un **snapshot** cohérent.

### Comment ça marche

Chaque **tuple** contient des métadonnées invisibles à l'utilisateur :

- **xmin** : ID de la transaction qui a créé/inséré ce tuple
- **xmax** : ID de la transaction qui a supprimé ou mis à jour ce tuple (initialement 0 = vivant)
- **cmin/cmax** : positions de commandes au sein d'une transaction
- **infomask** : flags (« est-ce un phantom tuple », etc.)

**Exemple** :

```sql
-- INSERT
BEGIN;
  INSERT INTO users (name) VALUES ('Alice');
  -- Crée tuple : xmin=100 (txn ID), xmax=0
COMMIT;

-- UPDATE (par une autre transaction)
BEGIN;
  UPDATE users SET name='Alice Updated' WHERE name='Alice';
  -- Ancienne version : xmin=100, xmax=101 (txn qui l'a tuée) → DEAD
  -- Nouvelle version : xmin=101, xmax=0 → VIVANTE pour txn 101+
COMMIT;

-- DELETE
BEGIN;
  DELETE FROM users WHERE name='Alice Updated';
  -- Nouvelle version : xmin=101, xmax=102 → DEAD
COMMIT;
```

### Snapshots et visibilité

Chaque transaction reçoit un **snapshot** = liste des txn actives au moment du `BEGIN`.

**Logique de visibilité** :
- Si `xmin` de la ligne < snapshot's xmin (txn la plus vieille en cours) → la ligne est **visible** (quelqu'un d'avant l'a créée, donc on la voit)
- Si `xmax` ≠ 0 ET xmax < snapshot's xmin → la ligne a été supprimée avant le début de ma txn → **invisible**
- Sinon → invisible (encore en cours ou supprimée après mon snapshot)

**Conséquence majeure pour les index** :
- Un **index pointe sur un ctid** sans savoir si ce tuple est vivant ou mort
- Postgres doit **toujours vérifier la visibilité** après avoir trouvé une entrée dans l'index
- Les tuples morts restent dans l'index tant qu'un VACUUM ne les nettoie pas → **index bloat**

### Exemple pratique

```sql
-- Créer une table avec MVCC visible
CREATE TABLE test_mvcc (id SERIAL, value TEXT);
INSERT INTO test_mvcc (value) VALUES ('Alice'), ('Bob'), ('Carol');

-- Voir l'état interne (xmin, xmax, ctid)
SELECT ctid, xmin, xmax, value FROM test_mvcc;
-- ctid   | xmin | xmax | value
-- (0,1)  | 1000 | 0    | Alice
-- (0,2)  | 1000 | 0    | Bob
-- (0,3)  | 1000 | 0    | Carol

-- Dans une autre transaction :
UPDATE test_mvcc SET value='Alice Updated' WHERE id=1;

-- Relancer le SELECT => Alice affiche la nouvelle version (Alice Updated), xmin/xmax changent
-- L'ancienne version existe toujours en disque mais est marquée xmax=1001
```

---

## VACUUM : nettoyage des dead tuples

### Qu'est-ce que VACUUM ?

**VACUUM** est un processus qui parcourt le heap et les index pour **marquer les espaces libres** et **purger les entrées de tuples morts**.

- **VACUUM** standard : marque l'espace libre, purge les index, mais ne compresse pas le fichier (reste à la même taille disque)
- **VACUUM FULL** : compresse physiquement le fichier (réécrit tout), réorganise le heap, mais pose des locks exclusifs (table inaccessible en écriture)
- **ANALYZE** : remet à jour les statistiques (cardinalité, distribution) utilisées par le planner

### Processus de VACUUM

1. **Scan du heap** : parcourt chaque page, identify les tuples morts (xmax < txn la plus vieille)
2. **Purge des index** : pour chaque index, supprime les entrées pointant sur des tuples morts
3. **Mise à jour de la Free Space Map** : liste des pages avec de l'espace disponible pour les INSERT/UPDATE futurs
4. **Freeze** : les vieux tuples (xmin < X tuples en arrière) sont marqués « frozen » pour éviter de les re-checker à l'infini

### autovacuum

PostgreSQL lance **autovacuum** automatiquement sur une table quand trop de tuples sont morts.

Paramètres clés dans `postgresql.conf` :

```ini
# Activer autovacuum (défaut : on)
autovacuum = on

# Lancer autovacuum quand : (10 + 0.1 * nb_lignes) tuples sont morts
autovacuum_vacuum_threshold = 10          # nombre minimum
autovacuum_vacuum_scale_factor = 0.1      # fraction

# Pour une table avec 1M lignes : trigger quand 100010+ tuples sont morts
# Pour une table avec 100M lignes : trigger quand 10000010+ tuples sont morts
```

**Problème commun** : sur une table très grande (100M+ lignes) avec beaucoup de writes, `autovacuum` n'arrive pas à suivre → accumulation de dead tuples → dégradation performance et storage.

**Solutions** :
- Ajuster les paramètres `autovacuum_vacuum_scale_factor` plus agressif (ex. `0.01` au lieu de `0.1`)
- Lancer VACUUM manuellement en creux (tâche cron)
- Pour les tables très grosses très modifiées, envisager une stratégie de **partitioning** (diviser la table en sous-tables par plage de temps)

### VACUUM FULL vs VACUUM standard

| Aspect | VACUUM | VACUUM FULL |
|--------|--------|-------------|
| **Lock** | AccessExclusiveLock (writes bloquées, reads possibles) | AccessExclusiveLock (tout bloqué) |
| **Temps** | Rapide (quelques min pour une grosse table) | Lent (réécrit tout) |
| **Espace disque** | Utilise Free Space Map | Peut réclamer de l'espace temp (2x la table) |
| **Réinitialise ctid** | Non | Oui (les ctid changent !) |
| **Quand l'utiliser** | Toujours, régulièrement via autovacuum | Rarement, sur petites tables après énormes DELETE ou en maint. |

**Attention** : VACUUM FULL réécrit les ctid, ce qui invalide **tous les index** (les pointeurs ne valent plus). Postgres gérer ça automatiquement, mais c'est coûteux.

### Exemple

```sql
-- Voir l'état VACUUM d'une table
SELECT
  schemaname,
  relname,
  last_vacuum,
  last_autovacuum,
  vacuum_count,
  autovacuum_count,
  n_dead_tup,
  n_live_tup
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- Lancer VACUUM manuel
VACUUM ANALYZE orders;  -- Rapide, pas de bloquage complet

-- Voir la progression
SELECT * FROM pg_stat_progress_vacuum;

-- VACUUM FULL (déconseillé sauf si vraiment nécessaire)
-- VACUUM FULL orders;  -- ⚠️ Bloque la table en écriture
```

---

## Query Planner

### Comment PostgreSQL décide d'utiliser un index

Le **planner** (optimiseur de requête) estime le **coût** de différents plans d'exécution et choisit le moins cher.

**Processus** :
1. **Parsing** : vérifie la syntaxe SQL
2. **Rewrite** : applique les règles (VIEW, règles de sécurité, etc.)
3. **Planing** : génère plusieurs plans possibles, estime le coût, choisit le meilleur
4. **Execution** : exécute le plan choisi

### Statistiques : la fondation des décisions

Le planner se base sur les **statistiques** stockées dans `pg_statistics` :

- **n_distinct** : nombre de valeurs uniques dans une colonne
- **null_frac** : fraction de NULL
- **avg_width** : largeur moyenne d'une valeur
- **histogram_bounds** : distribution des valeurs (10 buckets par défaut)
- **correlation** : corrélation entre l'ordre physique du heap et l'ordre du index (crucial pour BRIN)

Ces stats sont mises à jour par **ANALYZE**.

```sql
-- Voir les stats d'une colonne
SELECT
  attname,
  n_distinct,
  null_frac,
  avg_width,
  correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'customer_id';

-- Mettre à jour les stats (peut changer le plan de requête!)
ANALYZE orders;

-- Ou spécifiquement une colonne
ANALYZE orders (customer_id);
```

### Coûts estimés vs réels

Le planner calcule un **coût estimé** basé sur :

- **seq_page_cost** (défaut 1.0) : coût de lire une page en scan séquentiel
- **random_page_cost** (défaut 4.0) : coût de lire une page en accès aléatoire (index)
- **cpu_tuple_cost** (défaut 0.01) : coût de traiter un tuple
- **effective_cache_size** (défaut 25% RAM) : combien de pages PG pense pouvoir garder en cache

**Implication** : si tu changes `random_page_cost` de 4.0 à 1.0 (ex. disque SSD), PG préfèrera les index. Sur un serveur avec peu de RAM, baisse `effective_cache_size`.

```sql
-- Voir les paramètres de coût actuels
SHOW seq_page_cost;
SHOW random_page_cost;
SHOW effective_cache_size;

-- Les changer (temporairement pour une session)
SET random_page_cost = 1.5;  -- SSD: abaisse ce chiffre
SET effective_cache_size = '8 GB';
```

### EXPLAIN et EXPLAIN ANALYZE

**EXPLAIN** affiche le plan prévu sans l'exécuter.
**EXPLAIN ANALYZE** exécute réellement et affiche les vraies stats.

```sql
-- Plan seul (rapide)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Plan + execution (slow, montre buffers touchés)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
```

**Points clés à lire dans le résultat** :

```
Seq Scan on orders  (cost=0.00..45231.00 rows=12 width=128)
                    (actual time=0.041..312.4 rows=12 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 2000003
Buffers: shared hit=12388 read=20844
```

- **cost=0.00..45231.00** : coût estimé d'démarrage..coût total
- **rows=12** : nombre de lignes estimées (vs `actual rows=12` exécutées)
- **Rows Removed by Filter** : combien de lignes rejetées (scan séquentiel = test chaque ligne)
- **Buffers: shared hit=12388** : pages déjà en cache, **read=20844** : pages lues du disque

**Index Scan** :

```
Index Scan using idx_orders_customer_id
           on orders  (cost=0.56..24.3 rows=12 width=128)
                     (actual time=0.041..0.09 rows=12 loops=1)
  Index Cond: (customer_id = 42)
  Buffers: shared hit=5
```

- **Index Cond** : condition appliquée directement par l'index (TRÈS efficace)
- **Filter** (absent) : s'il y a un Filter, c'est une condition testée APRÈS l'index → moins efficace

**Règle d'or** : comparer les `Buffers: read` et le temps d'exécution. Index = beaucoup moins de buffers lus.

---

## Les types d'index

### Vue d'ensemble

PostgreSQL offre **7 types d'index** :

| Type | Structure | Cas d'usage | Opérateurs |
|------|-----------|------------|-----------|
| **B-Tree** | Arbre équilibré avec feuilles chaînées | Défaut, =, <, >, BETWEEN, LIKE | =, <>, <, >, <=, >= |
| **Hash** | Table de hachage | Égalité pure | = |
| **GIN** | Index inversé (valeur → lignes) | Arrays, JSONB, full-text search | @>, @<, ? |
| **GiST** | Arbre de recherche généralisé | Géométrie, ranges, plus proches voisins | &&, @, <->, etc. |
| **SP-GiST** | Arbre partitionné | Texte (tries), géospatial partitionné | \@, ~ (regex), <-> |
| **BRIN** | Min/max par range de blocs | Données naturellement triées (timestamps) | =, <, >, <= >= |
| **Bloom** | Filtre de Bloom probabiliste | Multi-colonne égalité | = |

### B-Tree (Balanced Tree) — le standard

**Structure** :
- Arbre équilibré avec **pages internes** (branches) et **pages feuilles** (données)
- Chaque page contient jusqu'à ~100–400 clés selon la largeur
- Les **feuilles sont chaînées** (doubly-linked list) pour les scans de range

**Propriétés** :
- Recherche : O(log n) → une table 1M lignes = ~4 niveaux
- Range queries (BETWEEN, <, >) : super efficace (utilise la chaîne de feuilles)
- Ordering : ORDER BY utilise le B-Tree pour lire les données triées
- Insertions : O(log n) mais peut causer des **page splits** (coûteux en écriture)

**Quand l'utiliser** :
- Recherche d'égalité : `WHERE id = 42` ✅
- Range : `WHERE created_at > '2024-01-01'` ✅
- Prefix matching : `WHERE name LIKE 'Alice%'` ✅ (B-Tree scan + filter)
- Ordering : `ORDER BY age ASC` ✅ (index scan donne déjà les lignes triées)

**Limites** :
- Faible cardinalité : si une colonne a 2 valeurs (booléen), l'index est quasi inutile
- Dead tuples : accumulation du bloat après beaucoup d'UPDATE/DELETE

```sql
-- Créer un B-Tree (type par défaut)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Multi-colonne : l'ordre COMPTE
CREATE INDEX idx_orders_customer_date
  ON orders(customer_id, created_at DESC);

-- Avec INCLUDE (covering index, pas indexé mais embarqué dans les feuilles)
CREATE INDEX idx_orders_customer_name
  ON orders(customer_id)
  INCLUDE (customer_name);  -- PostgreSQL 11+

-- Partial index (index seulement sur certaines lignes)
CREATE INDEX idx_pending_orders
  ON orders(created_at)
  WHERE status = 'pending';
```

**Cas d'usage** :
- 90% des indexes : pK, FK, colonnes de filtre courant

---

### Hash — accès O(1) pur

**Structure** :
- Table de hachage : chaque clé est hashée → bucket → liste de tuples

**Propriétés** :
- Recherche : O(1) moyenne (très rapide pour égalité)
- Pas de support pour les ranges ou l'ordering
- Collision possible (mais rares avec bon hasher)

**Quand l'utiliser** :
- UNIQUEMENT pour `WHERE col = valeur` (égalité pure)
- Rarement : B-Tree fait aussi bien et supporte plus

**Limites** :
- Pas de BETWEEN, pas de <, >, pas de LIKE
- Pas d'INDEX SCAN + ORDER BY

```sql
CREATE INDEX idx_user_email USING HASH ON users(email);

-- ✅ Utilise l'index
SELECT * FROM users WHERE email = 'alice@example.com';

-- ❌ N'utilise pas le hash (pas de support for range)
SELECT * FROM users WHERE email LIKE 'alice%';
```

**Verdict** : utilise B-Tree à la place, c'est plus universel.

---

### GIN — Generalized Inverted Index

**Concept** : pour les colonnes contenant **plusieurs valeurs par ligne** (arrays, JSONB, vecteurs de recherche full-text).

**Structure** :
- Au lieu de : ligne → valeurs
- GIN : valeur → liste de lignes (posting list)

**Propriétés** :
- Recherche : ultra-rapide pour « contains » ou « overlaps »
- Update : très coûteux (doit réorganiser les posting lists) → à éviter sur tables en écriture intensive
- Taille : peut être grosse (posting lists)

**Opérateurs** :
- `@>` : array contains (« contient »)
- `<@` : is contained by
- `?` : array element exists
- `@@` : full-text match

**Quand l'utiliser** :
- Arrays : `WHERE tags @> ARRAY['postgres', 'index']`
- JSONB : `WHERE config @> '{"debug": true}'`
- Full-text search : `WHERE to_tsvector('english', body) @@ to_tsquery('index & postgres')`

**Limites** :
- Lent en UPDATE/INSERT → **données read-only ou append-only seulement**
- Bloat important si beaucoup de modifications

```sql
-- Index GIN sur array
CREATE INDEX idx_tags ON posts USING GIN(tags);

-- Index GIN sur JSONB
CREATE INDEX idx_metadata ON articles USING GIN(metadata);

-- Index GIN full-text
CREATE INDEX idx_fts ON articles
  USING GIN(to_tsvector('english', body));

-- Utiliser l'index
SELECT * FROM posts WHERE tags @> ARRAY['postgres'];
SELECT * FROM articles WHERE metadata @> '{"status": "published"}';
SELECT * FROM articles
  WHERE to_tsvector('english', body) @@ to_tsquery('index & performance');
```

**Configuration** :
```sql
-- Voir les stats GIN
SELECT
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname LIKE '%tags%';
```

---

### GiST — Generalized Search Tree

**Concept** : arbre de recherche **générique** pour des données complexes (géométrie, ranges, nearest-neighbor).

**Structure** :
- Arbre avec **bounding boxes** (enveloppes) à chaque nœud
- Pour la géométrie : rectangles englobants
- Pour les ranges : union des ranges

**Propriétés** :
- Recherche : O(log n) mais avec beaucoup de false positives (enveloppe peut contenir d'autres objets)
- Flexible : extensible pour n'importe quel type de données

**Opérateurs courants** :
- `&&` : overlaps (chevauchement)
- `@` : contained by
- `<->` : distance (nearest neighbor)
- `<<`, `>>` : strictly left/right

**Quand l'utiliser** :
- **Géospatial (PostGIS)** : `WHERE ST_DWithin(location, point, radius)`
- **Date ranges** : `WHERE during @> '[2024-01-01, 2024-01-31]'::daterange`
- **IP ranges** : `WHERE ip_range >> '192.168.1.5'::inet`

**Limites** :
- False positives : l'index retourne des candidats que la requête doit re-vérifier
- Clustering : utilise les données pour construire le tree (peut être inefficace si données très aléatoires)

```sql
-- Index GiST pour géométrie (PostGIS)
CREATE EXTENSION postgis;
CREATE INDEX idx_location ON places USING GIST(coordinates);

-- Requête : tous les lieux dans un rayon de 10km
SELECT name FROM places
  WHERE ST_DWithin(
    coordinates,
    ST_MakePoint(2.35, 48.85),
    10000
  );

-- Index GiST pour date ranges
CREATE TABLE reservations (
  id SERIAL,
  during daterange,
  room_id INT
);
CREATE INDEX idx_bookings ON reservations USING GIST(during);

-- Requête : réservations qui chevauchent une plage
SELECT * FROM reservations
  WHERE during && '[2024-06-01, 2024-06-15]'::daterange;

-- Contrainte d'exclusion (aucun chevauchement)
ALTER TABLE reservations
  ADD CONSTRAINT no_overlapping_bookings
    EXCLUDE USING GIST (room_id WITH =, during WITH &&);
```

---

### SP-GiST — Space-Partitioned GiST

**Concept** : comme GiST mais **partitionne l'espace** (au lieu de bounding boxes).

**Structures internes** :
- **Quadtree** : partition récursive du plan en 4 quadrants (géospatial)
- **Radix tree** : partition par bits (strings, rangées)
- **K-d tree** : pour espaces multidimensionnels

**Propriétés** :
- Plus compact que GiST (pas de bounding boxes chevauchantes)
- Plus rapide pour certain patterns (ex: range sur strings)

**Opérateurs** :
- `@` : overlaps
- `~` : regex (pour strings)
- `<->` : distance

**Quand l'utiliser** :
- **Recherche de texte/rangees** : `WHERE word ~ '^test'` (commence par)
- **Géospatial partitionné** : quadtrees pour points (plus compact que GiST)
- **Ranges** : trie pour IP ranges, date ranges

**Vs GiST** :
- SP-GiST : mieux pour **données espacées** (peu de chevauchement)
- GiST : mieux pour **données denses** ou **overlaps fréquents**

```sql
-- Index SP-GiST pour prefix search (texte)
CREATE INDEX idx_words ON dictionary USING SPGIST(word);

-- Requête : mots commençant par "test"
SELECT * FROM dictionary WHERE word ~ '^test';

-- Index pour IP ranges (quadtree géo)
CREATE INDEX idx_ip_range ON networks USING SPGIST(ip_range);

-- Requête : IP appartient à un range
SELECT * FROM networks WHERE ip_range >> '10.0.0.5'::inet;
```

**Contre GiST** : SP-GiST est souvent plus compact et rapide pour les données **naturellement partitionnées**, mais moins flexible (moins d'opérateurs customs).

---

### BRIN — Block Range INdex

**Concept** : index super compact basé sur **min/max par range de blocs**.

**Structure** :
- Divise le heap en **ranges de 128 pages** (par défaut, ~1 MB chacun)
- Stocke min et max pour chaque range
- Pour une query : « skip les ranges où aucun tuple ne peut matcher »

**Propriétés** :
- Taille : minuscule (~0.1% de la table pour 100M lignes) ✅
- Efficacité : ultra-rapide sur données **naturellement triées** (timestamps, séquences)
- Useless sur **données aléatoires** : min/max ranges s'entrelacent

**Quand l'utiliser** :
- Timestamps d'insertion (created_at) : données arrivent dans l'ordre → parfait pour BRIN
- Serial IDs (auto-increment) : toujours croissant
- Append-only logs : jamais de UPDATE/DELETE, juste INSERT

**Limites** :
- Inutile sur données aléatoires (random UUIDs, etc.)
- Nécessite une corrélation physique (colonne corrélée avec l'ordre d'insertion)

```sql
-- Index BRIN sur timestamp (append-only)
CREATE INDEX idx_events_created USING BRIN(created_at) ON events;

-- Voir la corrélation (clé pour BRIN)
SELECT correlation FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';
-- Si close to 1.0 → parfait pour BRIN
-- Si close to 0 → BRIN sera inutile

-- BRIN avec paramètres custom
CREATE INDEX idx_events_custom
  USING BRIN(created_at) ON events
  WITH (pages_per_range = 256);  -- 2 MB ranges au lieu de 1 MB
```

**Exemple tailles** :
- Table 100M lignes, heap ~15 GB
- B-Tree sur created_at : ~2.5 GB
- BRIN sur created_at : ~1 MB (données bien ordonnées)

---

### Bloom Filter Index

**Concept** : **filtre probabiliste** pour multi-colonne égalité. Requiert l'extension `bloom`.

**Structure** :
- Bit array : chaque valeur est hashée dans plusieurs positions
- False positives possibles (la valeur passe le filtre mais n'existe pas)
- False negatives impossibles (si la valeur n'est pas dans le filtre, c'est sûr qu'elle n'existe pas)

**Propriétés** :
- Très compact
- Rapide pour filtrer (vrai/faux en temps O(1))
- Mais Postgres doit vérifier le heap pour les false positives

**Quand l'utiliser** :
- Table large, beaucoup de colonnes, filtres arbitraires
- Ex : table de log avec 20 colonnes, requêtes varient sur n'importe quelle combo

**Limites** :
- Requires extension `bloom`
- False positives → pas de garantie exacte (Postgres filtre après)
- Rarement optimal comparé à indexes composés ciblés

```sql
-- Activer l'extension
CREATE EXTENSION bloom;

-- Index Bloom sur 3 colonnes
CREATE INDEX idx_logs_bloom ON logs
  USING BLOOM(user_id, action, status)
  WITH (length=80, col1=4, col2=4, col3=4);

-- Utilisation : n'importe quelle combo des 3 colonnes
SELECT * FROM logs WHERE user_id = 42 AND action = 'login';
SELECT * FROM logs WHERE status = 'error';
SELECT * FROM logs WHERE user_id = 42 AND status = 'error';
```

**Verdict** : plutôt une option avancée pour patterns très complexes. B-Tree / GIN suffisent pour 99% des cas.

---

## Index avancés et patterns

### Index d'expression

Index une **expression** plutôt qu'une colonne directe.

```sql
-- Index sur la fonction lower() (case-insensitive search)
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Requête doit utiliser la MÊME expression
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Index sur une opération arithmétique
CREATE INDEX idx_salaries_annual ON employees(salary * 12);

-- Index sur partie d'un string
CREATE INDEX idx_phone_prefix ON customers(SUBSTRING(phone, 1, 3));
```

**Important** : la requête doit utiliser **exactement la même expression** pour que l'index soit utilisé.

---

### 🚨 CLARIFICATION : INCLUDE vs Partial Indexes (deux choses différentes !)

**Ces deux concepts répondent à des problèmes différents. Très important de ne pas les confondre.**

#### INCLUDE — embarquer des colonnes dans l'index (pour SELECT)

**Problème résolu** : tu cherches par `email`, mais tu SELECT aussi `name` et `avatar_url`. Sans INCLUDE, Postgres doit :
1. Scanner l'index pour trouver l'email
2. Fetcher la ligne du heap pour récupérer `name` et `avatar_url`

**Solution** : ajouter `INCLUDE (name, avatar_url)` pour embarquer ces colonnes **directement dans les feuilles de l'index**.

```sql
-- SANS INCLUDE : index seulement sur email
CREATE INDEX idx_users_email ON users(email);

-- Requête : doit fetcher le heap pour name, avatar_url
EXPLAIN SELECT name, avatar_url FROM users WHERE email = 'alice@example.com';
-- → Index Scan on users + Heap Fetch

-- AVEC INCLUDE : name, avatar_url embarqués
CREATE INDEX idx_users_email_covering
  ON users(email)
  INCLUDE (name, avatar_url);

-- MÊME requête : pas de heap fetch !
EXPLAIN SELECT name, avatar_url FROM users WHERE email = 'alice@example.com';
-- → Index Scan (Index-Only Scan, no Heap Access)
```

**Important** : colonnes INCLUDE ne peuvent **PAS** être dans WHERE (elles ne sont pas indexées, juste stockées).

```sql
-- ✅ INCLUDE fonctionne ici
SELECT name FROM users WHERE email = 'alice@ex.com';

-- ❌ INCLUDE n'aide pas ici (name n'est pas indexée)
SELECT * FROM users WHERE name = 'Alice';
```

**Taille** : INCLUDE ajoute un peu à la taille de l'index (les colonnes sont stockées), mais bien moins qu'une colonne indexée.

---

#### Partial Index — indexer seulement certaines LIGNES (pour WHERE)

**Problème résolu** : 99% de tes ordres sont `completed`, mais tu ne requêtes QUE les `pending`. Pourquoi indexer les 99% que tu ne toucheras jamais ?

**Solution** : ajouter une clause WHERE pour indexer seulement les lignes qui matchent la condition.

```sql
-- SANS partial : index TOUS les 1M ordres
CREATE INDEX idx_orders_date ON orders(created_at);
-- Taille : ~500 MB

-- AVEC partial : index SEULEMENT les ordres pending (~1%)
CREATE INDEX idx_pending_orders
  ON orders(created_at)
  WHERE status = 'pending';
-- Taille : ~5 MB (100x smaller!)
```

**Important** : la clause WHERE du partial index doit **matcher la requête** sinon l'index n'est pas utilisé.

```sql
-- ✅ Partial index utilisé (WHERE status = 'pending')
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';

-- ❌ Partial index NON utilisé (on demande les 'completed', index contient les 'pending')
SELECT * FROM orders WHERE status = 'completed';
```

---

#### INCLUDE vs Partial : cas concret combiné

```sql
-- Cas réel : table users 1M lignes
-- 95% des utilisateurs sont actifs (deleted_at IS NULL)
-- Requêtes courantes : "trouver user par email actif, retourner name + avatar"

-- SANS INCLUDE, SANS PARTIAL
CREATE INDEX idx_users_email ON users(email);
-- Taille : 200 MB (contient les 100k deleted users inutiles + 900k actifs)
-- Requête : index scan + heap fetch

-- AVEC PARTIAL seulement (réduit les lignes indexées)
CREATE INDEX idx_active_users_email
  ON users(email)
  WHERE deleted_at IS NULL;
-- Taille : ~190 MB (enlève les 10% morts, mais doit quand même fetcher heap)

-- AVEC INCLUDE seulement (embarque les colonnes, mais garde tous les users)
CREATE INDEX idx_users_email_covering
  ON users(email)
  INCLUDE (name, avatar_url);
-- Taille : ~230 MB (+30MB pour name, avatar)
-- Requête : index-only scan (pas de heap fetch)

-- AVEC PARTIAL + INCLUDE (le meilleur !)
CREATE INDEX idx_active_users_email_covering
  ON users(email)
  INCLUDE (name, avatar_url)
  WHERE deleted_at IS NULL;
-- Taille : ~200 MB (petit et pas de heap fetch !)
-- Requête : index-only scan sur les 900k utilisateurs actifs
```

**Résumé des rôles** :

| Feature | Résout quoi ? | Impact sur l'index | Aide avec WHERE ? | Aide avec SELECT ? |
|---------|--------------|-------------------|------------------|-----------------|
| **INCLUDE** | Heap fetches inutiles | Agrandit l'index | ❌ Non | ✅ Oui (Index-Only Scan) |
| **Partial** | Index trop gros sur données peu queryées | Rétrécit l'index | ✅ Oui (si WHERE match) | ❌ Non directement |
| **Les deux** | Optimisation totale | Petit + rapide | ✅ Oui | ✅ Oui |

---

### Partial indexes (index conditionnel)

Index seulement certaines **lignes** via une clause WHERE.

**Super utile pour** :
- **Soft deletes** : uniquement les lignes non supprimées
- **Status queues** : uniquement les jobs non traités
- **Active users** : uniquement les comptes actifs

```sql
-- Soft delete : index seulement lignes vivantes
CREATE INDEX idx_active_users ON users(email)
  WHERE deleted_at IS NULL;

-- Queue de jobs non traités (le cas parfait)
CREATE INDEX idx_pending_jobs ON jobs(priority DESC, created_at)
  WHERE processed_at IS NULL;

-- Commandes pas encore expédiées
CREATE INDEX idx_pending_orders ON orders(customer_id)
  WHERE shipped_at IS NULL;
```

**Avantages** :
- Index beaucoup **plus petit** (seulement 1-5% des lignes souvent)
- Maintenance plus rapide (moins de tuples à mettre à jour)
- Coût d'écriture réduit

**Unique constraint partiel** :

```sql
-- Permet plusieurs utilisateurs avec même email, tant qu'un seul est actif
CREATE UNIQUE INDEX idx_active_email
  ON users(email)
  WHERE deleted_at IS NULL;
```

### Covering indexes avec INCLUDE (PostgreSQL 11+)

Ajoute des colonnes à la **feuille de l'index** sans les indexer (pas utilisées pour la recherche).

Permet un **index-only scan** = pas besoin de fetcher le heap.

```sql
-- Sans INCLUDE : index sur email, mais SELECT name, avatar nécessite heap fetch
CREATE INDEX idx_users_email ON users(email);

-- Avec INCLUDE : name et avatar embarqués dans l'index
CREATE INDEX idx_users_email
  ON users(email)
  INCLUDE (name, avatar_url);

-- Requête : aucun heap fetch !
SELECT name, avatar_url FROM users WHERE email = 'alice@example.com';
-- → Index Scan (no heap access)
```

**Avantage** : combine la recherche efficace (email) + données additionnelles (name, avatar) sans agrandir énormément l'index (INCLUDE utilise moins d'espace que colonnes indexées).

**Limitation** : colonnes INCLUDE ne peuvent pas être utilisées dans WHERE (seulement SELECT).

### Créer un index sans bloquer (CONCURRENTLY)

Par défaut, CREATE INDEX pose un ACCESS EXCLUSIVE LOCK = table inaccessible en écriture.

```sql
-- Standard : table bloquée quelques minutes
CREATE INDEX idx_large_table ON huge_table(col);

-- CONCURRENTLY : écritures continuent, mais plus lent
CREATE INDEX CONCURRENTLY idx_large_table ON huge_table(col);

-- Vérifier la progression
SELECT * FROM pg_stat_progress_create_index;
```

**Utiliser CONCURRENTLY en production** sur grosses tables.

### REINDEX : reconstruire un index bloaté

```sql
-- Reindex standard (ACCESS EXCLUSIVE LOCK)
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;

-- Reindex toute une table
REINDEX TABLE CONCURRENTLY orders;

-- Vérifier la progression
SELECT * FROM pg_stat_progress_create_index;
```

---

## Coûts et maintenance

### Coût de stockage

Chaque index = **structure séparée sur disque**, en parallèle du heap.

```sql
-- Voir les tailles d'index
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_indexes
JOIN pg_stat_user_indexes ON indexname = indexrelname
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Règle du pouce** :
- B-Tree sur INT : ~15-20% de la taille du heap
- B-Tree sur TEXT(50) : ~40-50% de la taille du heap
- GIN sur JSONB : ~50-80%
- BRIN : ~0.01% (minuscule)

**Implication** : 5 B-Tree indexes = table doublée en espace disque. À prévoir sérieusement sur serveurs avec stockage limité.

### Write amplification

Chaque INSERT/UPDATE/DELETE doit **aussi modifier tous les index**.

```sql
-- Exemple : 1 INSERT sur table avec 5 indexes
-- Postgres doit :
-- 1. Écrire au WAL
-- 2. Insérer dans le heap
-- 3. Insérer dans idx_1 → potentiellement page split
-- 4. Insérer dans idx_2 → potentiellement page split
-- 5. Insérer dans idx_3, idx_4, idx_5 → idem

-- Une insertion = 6 opérations disque (1 heap + 5 index)
-- Une mise à jour est encore pire (marque l'ancienne version, crée une nouvelle)
```

**Impact** : table avec 100k INSERT/sec + 5 indexes = 600k disque writes/sec = saturation disque rapide.

**Solutions** :
- Réduire le nombre d'index : ne garder que ceux utilisés
- GIN sur données read-only (très lent en INSERT si contenu change)
- Batch inserts : `INSERT INTO ... VALUES (...), (...), (...)` est plus efficace
- Envisager partitioning : inserts distribuées sur plusieurs sous-tables

### Index bloat

Lors d'UPDATE/DELETE, les entrées d'index **meurent** mais restent en place (occupent l'espace).

```sql
-- Mesurer le bloat avec pgstattuple
CREATE EXTENSION pgstattuple;

SELECT
  schemaname,
  indexname,
  (pgstattuple_approx(indexrelid)).dead_tuple_count AS dead_tuples,
  (pgstattuple_approx(indexrelid)).live_tuple_count AS live_tuples,
  ROUND(
    100.0 * (pgstattuple_approx(indexrelid)).dead_tuple_count /
      NULLIF((pgstattuple_approx(indexrelid)).live_tuple_count +
             (pgstattuple_approx(indexrelid)).dead_tuple_count, 0),
    2
  ) AS dead_ratio
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY dead_ratio DESC;
```

**Quand REINDEX** :
- Si dead_ratio > 30% = reindex conseillé
- Index bloaté ralentit les scans (doit lire pages inutiles)

### VACUUM et index cleanup

**VACUUM** nettoie aussi les index (supprime les entrées mortes).

- VACUUM standard : marque les espaces, nettoie les index → efficace
- VACUUM FULL : refait tout le heap + index, très lent mais récupère de l'espace

```sql
-- Lancer un VACUUM manuel (rapide)
VACUUM ANALYZE orders;

-- Vérifier ce qui a été nettoyé
SELECT
  schemaname,
  relname,
  last_vacuum,
  n_dead_tup,
  vacuum_count
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

---

## Monitoring

### Identifier les index inutilisés

```sql
-- Index jamais utilisés (candidates au DROP)
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Attention** : un index récemment créé peut avoir 0 scans. Laisser tourner quelques jours avant de décider.

### Identifier les tables avec beaucoup de seq scans

**Sequential scans** = indicateur qu'un index manque peut-être.

```sql
-- Tables avec beaucoup de seq scans vs index scans
SELECT
  schemaname,
  relname,
  seq_scan,
  idx_scan,
  n_live_tup,
  ROUND(100.0 * seq_scan / (seq_scan + idx_scan), 2) AS seq_scan_ratio
FROM pg_stat_user_tables
WHERE (seq_scan + idx_scan) > 0
  AND n_live_tup > 10000  -- Ignore les petites tables
ORDER BY seq_scan DESC
LIMIT 20;
```

**Interprétation** :
- `seq_scan_ratio` proche de 100% = utilise presque jamais les index (candidates pour ajouter un index)
- `seq_scan_ratio` = 5% = bon (index bien utilisé, seq scans pour small ranges)

### Coût des requêtes lentes

Utilise `pg_stat_statements` (extension) pour voir les requêtes les plus coûteuses.

```sql
-- Activer pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- Top 10 requêtes par temps cumulé
SELECT
  query,
  calls,
  total_time,
  mean_time,
  max_time,
  rows
FROM pg_stat_statements
WHERE query NOT ILIKE '%pg_stat_statements%'
ORDER BY total_time DESC
LIMIT 10;
```

### Configuration pour monitoring actif

Ajouter dans `postgresql.conf` :

```ini
# Activer log des requêtes lentes (> 1 sec)
log_min_duration_statement = 1000

# Collecter les plans de requêtes lentes automatiquement
auto_explain.log_min_duration = 1000

# (Requiert extension auto_explain)
shared_preload_libraries = 'auto_explain'
```

Puis relancer PostgreSQL et les requêtes > 1 sec seront loggées avec leurs plans EXPLAIN.

---

## Démos en live

### Démo 1 : Sequential Scan → Index Scan

```sql
-- 1. Créer une table de test
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INT,
  amount DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Remplir avec données (1M lignes)
INSERT INTO orders (customer_id, amount)
SELECT
  (random() * 10000)::INT,
  (random() * 1000)::DECIMAL(10, 2)
FROM generate_series(1, 1000000);

ANALYZE orders;

-- 3. Requête sans index : EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
-- Affiche : Seq Scan, 312 ms, 12388 buffers hit + 20844 read

-- 4. Créer un index
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- 5. Relancer la requête
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
-- Affiche : Index Scan, 0.1 ms, 5 buffers
```

**À montrer** :
- Le temps passe de 312 ms à 0.1 ms
- Les buffers hit/read diminuent drastiquement
- Le plan change de "Seq Scan" à "Index Scan"

### Démo 2 : GIN sur JSONB (read vs write)

```sql
-- 1. Table articles avec métadonnées JSONB
CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  title TEXT,
  metadata JSONB
);

-- 2. Insérer avec des métadonnées variées
INSERT INTO articles (title, metadata) VALUES
  ('Article 1', '{"author":"Alice","status":"published","tags":["postgres"]}'),
  ('Article 2', '{"author":"Bob","status":"draft","tags":["index","performance"]}'),
  ('Article 3', '{"author":"Alice","status":"published","tags":["postgres","tuning"]}');

-- Répéter 1M fois pour une vraie table
INSERT INTO articles (title, metadata)
SELECT
  'Article ' || i,
  jsonb_build_object(
    'author', (ARRAY['Alice', 'Bob', 'Carol'])[random()*3+1],
    'status', (ARRAY['published', 'draft'])[random()*2+1],
    'tags', jsonb_build_array(random()::TEXT, random()::TEXT)
  )
FROM generate_series(1, 1000000) i;

ANALYZE articles;

-- 3. Requête sans GIN index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM articles
WHERE metadata @> '{"author":"Alice"}';
-- Seq Scan, ~500 ms, beaucoup de buffers

-- 4. Créer GIN index
CREATE INDEX idx_articles_metadata USING GIN(metadata);

-- 5. Requête avec GIN
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM articles
WHERE metadata @> '{"author":"Alice"}';
-- Index Scan, ~10 ms (50x faster !)

-- 6. Mesurer le coût : insérer 10k lignes
\timing on

INSERT INTO articles (title, metadata) VALUES
  ('New 1', '{"author":"Alice","status":"published"}'),
  -- ... 10k times

-- Montrer que c'est beaucoup plus lent (GIN updates = lent)
```

### Démo 3 : BRIN sur timestamps (append-only)

```sql
-- 1. Table d'événements (append-only, timestamps croissants)
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  event_type TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Insérer 10M lignes chronologiquement
INSERT INTO events (event_type, created_at)
SELECT
  (ARRAY['login','logout','click','error'])[random()*4+1],
  NOW() - INTERVAL '1 day' * (10000000 - row_number() OVER ())
FROM generate_series(1, 10000000);

-- 3. Créer B-Tree index (volumineux)
CREATE INDEX idx_events_created_btree ON events(created_at);
SELECT pg_size_pretty(pg_relation_size('idx_events_created_btree'));
-- ~500 MB

-- 4. Créer BRIN index (minuscule)
CREATE INDEX idx_events_created_brin USING BRIN(created_at) ON events;
SELECT pg_size_pretty(pg_relation_size('idx_events_created_brin'));
-- ~1 MB (500x smaller!)

-- 5. Comparer les performances (range query)
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM events
WHERE created_at > NOW() - INTERVAL '1 day';
-- Avec BRIN : très rapide, peu de buffers (peut skipper ranges)
```

### Démo 4 : Trouver et supprimer un index inutilisé

```sql
-- 1. Créer un index qui ne sera jamais utilisé
CREATE INDEX idx_waste ON orders(amount);

-- 2. Laisser tourner la table quelques jours (ou simuler avec reset)
SELECT pg_sleep(1);

-- 3. Identifier l'index inutilisé
SELECT
  indexname,
  idx_scan,
  idx_tup_read,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE indexname = 'idx_waste';
-- idx_scan = 0, idx_tup_read = 0 → DROP !

-- 4. Supprimer
DROP INDEX CONCURRENTLY idx_waste;

-- 5. Vérifier la taille totale avant/après
SELECT pg_size_pretty(pg_total_relation_size('orders'));
```

---

## Cheat-sheet

### Quel index pour quel besoin ?

| Cas d'usage | Type | Colonne(s) | Exemple SQL |
|-----------|------|-----------|------------|
| PK, FK | B-Tree | col ou (col1, col2) | `CREATE INDEX ON orders(customer_id)` |
| Recherche texte simple | B-Tree | col | `WHERE name = 'Alice'` |
| Range (date, prix) | B-Tree | col | `WHERE created_at > '2024-01-01'` |
| Prefix search | B-Tree | col | `WHERE name LIKE 'Ali%'` |
| Soft delete | Partial B-Tree | col | `WHERE deleted_at IS NULL` |
| Status queue | Partial B-Tree | (priority, created_at) | `WHERE processed_at IS NULL` |
| Case-insensitive search | B-Tree expr | LOWER(col) | `WHERE LOWER(email) = 'alice@ex.com'` |
| Array contains | GIN | array_col | `WHERE tags @> ARRAY['postgres']` |
| JSONB contains | GIN | jsonb_col | `WHERE config @> '{"debug":true}'` |
| Full-text search | GIN | to_tsvector(col) | `WHERE to_tsvector('english', body) @@ to_tsquery('index & postgres')` |
| Géospatial | GiST | geom_col | `WHERE ST_DWithin(loc, point, 10000)` |
| Date ranges overlap | GiST | range_col | `WHERE during && '[2024-01-01,2024-01-31]'` |
| Prefix sur string | SP-GiST | col | `WHERE col ~ '^prefix'` |
| Timestamps append-only | BRIN | created_at | (données naturellement triées) |
| Serial IDs append-only | BRIN | id | (auto-increment) |
| Multi-colonne combo | Bloom | (col1, col2, col3) | (filtrage arbitraire) |

### Checklist avant d'ajouter un index

- [ ] EXPLAIN ANALYZE : la requête fait un seq scan ?
- [ ] La table a > 10k lignes ? (sur petit table, seq scan peut être plus rapide)
- [ ] La colonne a bonne cardinalité ? (pas booléen ou 3 valeurs uniquement)
- [ ] La requête utilise = ou < ou > ou BETWEEN ? (pas seulement LIKE sans préfixe)
- [ ] L'index sera-t-il utilisé souvent ? (vérifier les stats après création)
- [ ] Stockage OK ? (chaque index = 15-50% du heap)
- [ ] Write performance acceptable ? (chaque index = latence ajoutée)

### Commandes essentielles

```sql
-- Analyser une requête
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- Voir les index inutilisés
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- Voir les tables avec beaucoup de seq scans
SELECT * FROM pg_stat_user_tables
WHERE (seq_scan + idx_scan) > 0
ORDER BY seq_scan DESC;

-- Mesurer le bloat
CREATE EXTENSION pgstattuple;
SELECT (pgstattuple_approx(indexrelid)).dead_tuple_count FROM pg_indexes ...;

-- VACUUM manuel
VACUUM ANALYZE table_name;

-- Reindex sans bloquer
REINDEX INDEX CONCURRENTLY idx_name;

-- Voir les stats actuelles
SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
```

---

## Ressources

- **Use the Index, Luke!** (use-the-index-luke.com) : libre, excellente explication indexing
- **PostgreSQL official docs, Chapter 11** : référence définitive
- **The Art of PostgreSQL** (Dimitri Fontaine) : indexing strategy approfondie
- **pgAdmin, pgBadger, pgDash** : outils de monitoring PostgreSQL

---

## Annexe : Configuration postgresql.conf pour performance

```ini
# ===== Shared Memory & Cache =====
shared_buffers = 256MB          # ~25% RAM pour serveur dédié
effective_cache_size = 2GB      # ~50% RAM disponible

# ===== Planner Costs =====
seq_page_cost = 1.0             # SSD: baisse ce paramètre (défaut 1.0)
random_page_cost = 4.0          # SSD: diminue à 1.5-2.0 pour boost index
cpu_tuple_cost = 0.01

# ===== VACUUM & Maintenance =====
autovacuum = on
autovacuum_vacuum_threshold = 10
autovacuum_vacuum_scale_factor = 0.1      # Agressif sur grosses tables : 0.01
autovacuum_analyze_scale_factor = 0.05

# ===== WAL & Durability =====
wal_buffers = 16MB
checkpoint_completion_target = 0.9
wal_level = replica               # Required for streaming replication

# ===== Logging for Diagnostics =====
log_min_duration_statement = 1000  # Log queries > 1 sec
log_statement = 'all'              # (Verbose, but useful for audits)
shared_preload_libraries = 'auto_explain,pg_stat_statements'
auto_explain.log_min_duration = 1000

# ===== Connection Management =====
max_connections = 200
```

Restart PostgreSQL après changements :
```bash
pg_ctl restart -D /var/lib/postgresql/16/main
# ou sur systèmes systemd
systemctl restart postgresql
```

---

**Notes finales** :

1. **Toujours EXPLAIN d'abord** — ne crée jamais un index sans vérifier que ça aide vraiment
2. **Monitor régulièrement** — `pg_stat_user_indexes`, `pg_stat_statements`, VACUUM logs
3. **Les données dictent le design** — append-only → BRIN, arrays → GIN, géospatial → GiST
4. **Moins d'index = plus rapide** — chaque index ralentit les writes
5. **VACUUM = ami** — un autovacuum bien configuré peut doubler la performance

Bon talk ! 🚀
