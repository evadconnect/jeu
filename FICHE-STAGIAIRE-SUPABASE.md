# Fiche stagiaire · Externaliser les données du jeu vers Supabase

**Projet :** EVAD Le Jeu (`evadconnect/jeu`, fichier `index.html`)
**Mission :** sortir les données **Solutions**, **ICI** et **Compétences** du HTML pour les gérer dans **3 tables Supabase**, puis les rebrancher dans le jeu.
**Durée estimée :** 3 à 5 jours.

---

## 1. Contexte

Le jeu est un **fichier HTML sans build** (aucune dépendance, tout en JavaScript inline) et sera **hébergé en ligne sur `jeu.evad.org`**. Aujourd'hui, les cartes du jeu sont écrites en dur dans `index.html` sous forme de tableaux/objets JavaScript :

- `SOLUTIONS` (les solutions à installer) : ligne ~821
- `ICI` (les indicateurs de mesure) : ligne ~909
- `COMPETENCES` (compétences Bâtisseur) : ligne ~927 et `PCOMP` (compétences Pilote) : ligne ~871

Le but est que l'équipe puisse **ajouter ou modifier une carte sans toucher au code**, via Supabase.

Le jeu sait déjà parler à Supabase : le formulaire de feedback fait un `INSERT` REST direct (voir `EVAD_STATS_SUPABASE`, ligne ~1226). Tu vas t'appuyer sur le même mécanisme, mais en **lecture**.

### Chargement des données

Le jeu étant en ligne, **Supabase est la source de vérité** : au démarrage, les 3 tables sont chargées depuis Supabase, puis le jeu se lance. Pas besoin de mode hors-ligne. Gère juste proprement le cas d'erreur réseau (message clair + éventuel bouton « réessayer ») pour ne pas laisser un écran vide si l'appel échoue.

---

## 2. Ce que tu dois livrer

1. Un script SQL (`supabase/schema.sql`) qui crée les 3 tables (avec le **cycle de validation**, voir §6) + les politiques RLS.
2. Un script SQL (`supabase/seed.sql`) qui insère les données actuelles (reprises depuis le HTML), au statut `valide`.
3. La modification de `index.html` : au démarrage, le jeu charge les 3 tables depuis Supabase (**uniquement les entrées validées**) avant de se lancer, avec une **gestion d'erreur propre** si l'appel échoue.
4. **Une interface de validation pour le Conseil Régénératif (CR)** : une page où les membres du CR se connectent, voient les propositions en attente, et les valident ou les rejettent (voir §6).
5. Une courte doc (`supabase/README.md`) : comment se connecter, où sont les clés, comment ajouter une carte, comment le CR valide.

---

## 3. Accès & sécurité

- **Projet Supabase** existant : `https://lmhhrccmgebztioesmik.supabase.co` (tu y as déjà accès).
- Pour la **lecture publique** depuis le jeu : on utilise la **clé anon / publishable** (comme pour le feedback). Elle peut vivre dans le HTML, c'est fait pour.
- Pour **écrire/administrer** (créer les tables, insérer) : passe par le dashboard Supabase (SQL Editor) ou une clé `service_role`. **La `service_role` ne doit JAMAIS être commitée ni mise dans le HTML.**
- Active la **RLS** sur les 3 tables avec une policy **lecture seule** pour le rôle `anon` (voir §5). Aucune écriture publique.

---

## 4. Modèle de données

Types Postgres. Pour les listes simples de chaînes (ex. `['terre','toit']`), on utilise `text[]` : l'API REST de Supabase les renvoie directement en tableaux JSON, ce qui colle au format attendu par le JS.

### Table `ici`

L'indicateur de mesure attaché à une solution.

| colonne | type | exemple | note |
|---|---|---|---|
| `id` | `text` PK | `eau` | clé métier, réutilisée par `solutions.ici_id` |
| `nom` | `text` | `Eau de pluie récupérée` | |
| `icone` | `text` | `💧` | emoji |
| `unite` | `text` | `m³` | |
| `referentiels` | `text[]` | `{ODD 6,ESRS E3,PCAET}` | langages couverts |

Source dans le HTML : objet `ICI` (8 entrées). Les référentiels possibles (`ODD`, `ESRS`, `PCAET`, `RSE`) sont détaillés dans l'objet `REFS` : si tu veux, tu peux en faire une petite table de référence `referentiels(code, nom, libelle)`, mais ce n'est pas obligatoire pour la V1.

### Table `competences`

Deux familles cohabitent, distinguées par `profil` :
- **`batisseur`** : compétences du profil Bâtisseur (objet `COMPETENCES` dans le HTML). Servent à débloquer des missions. Champ clé : `couvre` (liste de solutions concernées, ou `{*}` pour toutes).
- **`pilote`** : compétences du profil Pilote (objet `PCOMP` dans le HTML). S'achètent en Graines, boostent la production, et **débloquent** certaines solutions.

| colonne | type | exemple | s'applique à |
|---|---|---|---|
| `id` | `text` PK | `jardiner` / `maraichage` | tous |
| `profil` | `text` | `batisseur` / `pilote` | tous |
| `nom` | `text` | `Jardiner` | tous |
| `icone` | `text` | `🌱` | tous |
| `description` | `text` | `+1🌰/saison par solution Alimentation...` | pilote |
| `cout` | `int` | `6` | pilote (en Graines) |
| `couvre` | `text[]` | `{potager,serre,mur-vegetal,phyto}` | batisseur (ids de solutions, ou `{*}`) |
| `couleur` | `text` | `#2e6b47` | pilote (facultatif, couleur d'affichage) |

> Le lien « telle solution nécessite telle compétence Pilote » (objet `SOL_LOCK` dans le HTML) est porté par la colonne `solutions.competence_requise` ci-dessous, pas ici.

### Table `solutions`

Le cœur du jeu. Une carte = un objet de `SOLUTIONS`, enrichi par `SOL_COUT` (coût) et `SOL_LOCK` (verrou de compétence).

| colonne | type | exemple | note |
|---|---|---|---|
| `id` | `text` PK | `potager` | clé métier |
| `nom` | `text` | `Potager en pleine terre` | |
| `icone` | `text` | `🥬` | emoji |
| `categorie` | `text` | `alimentaire` | une des 7 catégories (voir tableau ci-dessous) |
| `complexite` | `text` | `simple` | `simple`, `moderee` ou `complexe` (voir tableau ci-dessous) |
| `description` | `text` | `Légumes de saison en circuit court.` | |
| `capacites` | `text[]` | `{terre}` | espaces acceptés : `terre`, `toit`, `piece`, `mur` |
| `vadance` | `int` | `4` | impact promis à l'installation |
| `graines` | `int` | `3` | production de Graines / saison |
| `cout` | `int` | `3` | coût d'installation en Graines (ex-`SOL_COUT`) |
| `ici_id` | `text` FK → `ici(id)` | `biodiv` | indicateur mesurable associé |
| `flux_in` | `text[]` | `{compost,eau}` | entrées (pour les boucles circulaires) |
| `flux_out` | `text[]` | `{legumes,biodechets}` | sorties |
| `climat` | `text` null | `fraicheur` | parade canicule, sinon `null` |
| `protege` | `text` null | `gel` | aléa protégé : `gel`, `secheresse`, `tempete`, sinon `null` |
| `dormant` | `boolean` | `true` | ne produit pas en hiver |
| `hiver` | `boolean` | `false` | actif l'hiver (ex-`winter`) |
| `competence_requise` | `text` null FK → `competences(id)` | `maraichage` | verrou Pilote (ex-`SOL_LOCK`), sinon `null` |
| `ordre` | `int` | `10` | ordre d'affichage (facultatif) |

**Relations :**
- `solutions.ici_id` → `ici.id`
- `solutions.categorie` → un des 7 codes ci-dessous (`text` + `check`, ou une table `categories` si tu veux)
- `solutions.competence_requise` → `competences.id` (famille `pilote`)
- `competences.couvre[]` référence des `solutions.id` (famille `batisseur`)

### Catégories (7) et complexité (3)

Taxonomie officielle des solutions (valeurs autorisées de `categorie`) :

| code | libellé | icône |
|---|---|---|
| `eau` | Eau | 💧 |
| `energie` | Énergie | ⚡ |
| `construction` | Construction | 🧱 |
| `alimentaire` | Alimentaire | 🌱 |
| `dechets` | Déchets | ♻️ |
| `biodiversite` | Biodiversité | 🌿 |
| `social` | Social | 🤝 |

Niveau de complexité de mise en œuvre (valeurs autorisées de `complexite`) :

| code | libellé | pastille |
|---|---|---|
| `simple` | Simple | 🟢 |
| `moderee` | Modérée | 🟡 |
| `complexe` | Complexe | 🔴 |

### Re-mapping des 13 solutions existantes

Correspondance ancienne catégorie (jeu actuel, objet `CATS` à 4 valeurs) vers la nouvelle taxonomie, avec la complexité proposée. **Ce sont les valeurs à mettre dans le `seed.sql`.**

| id | nom | ancienne cat | **catégorie** | **complexité** |
|---|---|---|---|---|
| `potager` | Potager en pleine terre | alim | `alimentaire` | `simple` |
| `serre` | Serre bioclimatique | alim | `alimentaire` | `moderee` |
| `rucher` | Rucher partagé | alim | `biodiversite` | `moderee` |
| `poulailler` | Poulailler | alim | `alimentaire` | `simple` |
| `compost` | Composteur | circulaire | `dechets` | `simple` |
| `recup-eau` | Récupérateur d'eau de pluie | fraicheur | `eau` | `simple` |
| `mur-vegetal` | Mur végétalisé | fraicheur | `biodiversite` | `moderee` |
| `ombriere` | Ombrière solaire | fraicheur | `energie` | `complexe` |
| `phyto` | Phytoépuration | fraicheur | `eau` | `complexe` |
| `solaire` | Panneaux photovoltaïques | energie | `energie` | `moderee` |
| `atelier` | Atelier de réparation | circulaire | `dechets` | `moderee` |
| `recyclerie` | Recyclerie de quartier | circulaire | `dechets` | `simple` |
| `cuisine` | Cuisine partagée | alim | `alimentaire` | `moderee` |

Répartition : `alimentaire` (4), `dechets` (3), `biodiversite` (2), `eau` (2), `energie` (2). Les catégories `construction` et `social` n'ont pas encore de solution : elles serviront aux futures propositions validées par le CR.

> **À réconcilier côté jeu (`index.html`) :** le jeu tourne encore sur les 4 anciennes catégories. En plus du re-mapping ci-dessus, il faudra (1) mettre à jour `CATS` (libellés + couleurs des 7 catégories), (2) adapter les boosts `PCOMP` qui ciblent une catégorie (« +1🌰/saison par solution Alimentation/Énergie/Réparation » pointent vers `alim`/`energie`/`circulaire`), (3) décider de l'usage de `complexite` en jeu (badge d'affichage, ou effet sur le coût/la durée). Le champ `clim: 'fraicheur'` des solutions est indépendant de la catégorie : la parade canicule n'est pas impactée.

---

## 5. SQL de départ (à adapter)

Chaque table porte aussi **5 colonnes de validation** communes (le cycle est décrit en §6) :

```sql
-- Colonnes de validation à ajouter aux 3 tables :
--   statut text not null default 'brouillon'
--     check (statut in ('brouillon','en_revue','valide','rejete')),
--   propose_par text,                         -- qui a soumis (email libre ou uid)
--   valide_par  uuid references auth.users,   -- membre du CR qui a tranché
--   valide_le   timestamptz,
--   commentaire_cr text                       -- motif en cas de rejet / note
```

```sql
-- ICI
create table ici (
  id text primary key,
  nom text not null,
  icone text,
  unite text,
  referentiels text[] default '{}',
  statut text not null default 'brouillon' check (statut in ('brouillon','en_revue','valide','rejete')),
  propose_par text,
  valide_par uuid references auth.users,
  valide_le timestamptz,
  commentaire_cr text
);

-- COMPETENCES (pilote + batisseur)
create table competences (
  id text primary key,
  profil text not null check (profil in ('pilote','batisseur')),
  nom text not null,
  icone text,
  description text,
  cout int,
  couvre text[] default '{}',
  couleur text,
  statut text not null default 'brouillon' check (statut in ('brouillon','en_revue','valide','rejete')),
  propose_par text,
  valide_par uuid references auth.users,
  valide_le timestamptz,
  commentaire_cr text
);

-- SOLUTIONS
create table solutions (
  id text primary key,
  nom text not null,
  icone text,
  categorie text check (categorie in ('eau','energie','construction','alimentaire','dechets','biodiversite','social')),
  complexite text check (complexite in ('simple','moderee','complexe')),
  description text,
  capacites text[] default '{}',
  vadance int default 0,
  graines int default 0,
  cout int default 3,
  ici_id text references ici(id),
  flux_in text[] default '{}',
  flux_out text[] default '{}',
  climat text,
  protege text,
  dormant boolean default false,
  hiver boolean default false,
  competence_requise text references competences(id),
  ordre int,
  statut text not null default 'brouillon' check (statut in ('brouillon','en_revue','valide','rejete')),
  propose_par text,
  valide_par uuid references auth.users,
  valide_le timestamptz,
  commentaire_cr text
);

-- RLS
alter table ici enable row level security;
alter table competences enable row level security;
alter table solutions enable row level security;

-- 1) Le jeu (anon) ne lit QUE le validé
create policy "jeu lit le valide - ici"         on ici         for select to anon using (statut = 'valide');
create policy "jeu lit le valide - competences" on competences for select to anon using (statut = 'valide');
create policy "jeu lit le valide - solutions"   on solutions   for select to anon using (statut = 'valide');

-- 2) Le CR (connecté) voit tout et peut valider / rejeter (mettre à jour)
create policy "CR lit tout - ici"         on ici         for select to authenticated using (true);
create policy "CR lit tout - competences" on competences for select to authenticated using (true);
create policy "CR lit tout - solutions"   on solutions   for select to authenticated using (true);

create policy "CR met a jour - ici"         on ici         for update to authenticated using (true) with check (true);
create policy "CR met a jour - competences" on competences for update to authenticated using (true) with check (true);
create policy "CR met a jour - solutions"   on solutions   for update to authenticated using (true) with check (true);
```

> Ici, tout compte **connecté** est considéré comme membre du CR (tu n'invites que le CR). Si tu veux verrouiller davantage, crée une table `membres_cr(user_id)` et remplace `using (true)` par `using (auth.uid() in (select user_id from membres_cr))`.

Exemples de `INSERT` (à générer pour toutes les lignes en reprenant le HTML) :

```sql
insert into ici (id, nom, icone, unite, referentiels) values
  ('eau', 'Eau de pluie récupérée', '💧', 'm³', '{ODD 6,ESRS E3,PCAET}');

insert into competences (id, profil, nom, icone, couvre) values
  ('jardiner', 'batisseur', 'Jardiner', '🌱', '{potager,serre,mur-vegetal,phyto}');

insert into competences (id, profil, nom, icone, cout, description, couleur) values
  ('maraichage', 'pilote', 'Maraîchage', '🌱', 6,
   '+1🌰/saison par solution Alimentation. Débloque la Serre bioclimatique.', '#2e6b47');

insert into solutions
  (id, nom, icone, categorie, complexite, description, capacites, vadance, graines, cout, ici_id, flux_in, flux_out, dormant, competence_requise, ordre, statut) values
  ('potager', 'Potager en pleine terre', '🥬', 'alimentaire', 'simple',
   'Légumes de saison en circuit court.', '{terre}', 4, 3, 3, 'biodiv',
   '{compost,eau}', '{legumes,biodechets}', true, null, 10, 'valide');
```

> **Important :** les données reprises du HTML existent déjà en prod, donc **insère-les au statut `valide`** (comme ci-dessus), sinon elles resteraient en `brouillon` et n'apparaîtraient pas dans le jeu. Si tu oublies, rattrape avec `update solutions set statut='valide';` (idem `ici`, `competences`).

---

## 6. Workflow de validation (Conseil Régénératif)

Le **Conseil Régénératif (CR)** doit valider chaque Solution, ICI ou Compétence avant qu'elle apparaisse dans le jeu. C'est le cœur de la démarche EVAD : « le vérifié bat le certifié ».

### Le cycle de statut

```
brouillon  ->  en_revue  ->  valide     (visible dans le jeu)
                    |
                    +------>  rejete     (avec un commentaire du CR)
```

- `brouillon` : en cours de rédaction (par l'équipe ou une proposition externe).
- `en_revue` : soumis au CR, en attente de décision.
- `valide` : approuvé par le CR. **Seul statut lu par le jeu.**
- `rejete` : refusé, avec un motif dans `commentaire_cr` (l'auteur peut corriger et re-soumettre).

Le passage `valide -> en_revue` (repasser en revue une carte modifiée) est autorisé aussi : une carte déjà en jeu qu'on veut retoucher redevient `en_revue` le temps d'une nouvelle validation.

### L'interface de validation (à développer)

Une **page séparée**, dans le même esprit que le jeu (HTML + JS, sans build), par exemple `validation.html`. Elle sera hébergée à part et **réservée au CR** (idée d'URL : `conseil.evad.org` ou `jeu.evad.org/validation`). Elle n'est **pas** livrée dans le jeu public.

Fonctionnalités attendues :

1. **Connexion CR via Supabase Auth**, de préférence par **lien magique par e-mail** (magic link) : le membre saisit son e-mail, reçoit un lien, se connecte. Pas de mot de passe à gérer, idéal pour des membres non techniques.
2. **File d'attente** : la liste des entrées `en_revue`, regroupées par type (Solutions / ICI / Compétences), avec un compteur.
3. **Fiche lisible** de chaque proposition : afficher son contenu de façon claire (nom, icône, description, catégorie, ICI associé, flux, etc.), pas un JSON brut, pour que le CR juge sur le fond.
4. **Deux actions** par entrée :
   - **Valider** -> `statut = 'valide'`, renseigne `valide_par = auth.uid()` et `valide_le = now()`.
   - **Rejeter** -> `statut = 'rejete'` + `commentaire_cr` obligatoire (le motif).
5. **Filtre** pour consulter aussi le `valide` et le `rejete` (historique), pas seulement la file d'attente.

L'écriture se fait par `PATCH` REST avec le **jeton de la session connectée** (pas la clé anon) : les policies RLS de mise à jour (§5) laissent passer un utilisateur `authenticated`, refusent un `anon`.

```js
// Exemple : valider une solution (session CR active)
await fetch(url + '/rest/v1/solutions?id=eq.' + id, {
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/json',
    'apikey': ANON_KEY,
    'Authorization': 'Bearer ' + session.access_token, // jeton du membre connecté
    'Prefer': 'return=minimal'
  },
  body: JSON.stringify({ statut: 'valide', valide_le: new Date().toISOString() })
});
```

> Pour la connexion, le plus simple est d'inclure le **client Supabase JS** (`@supabase/supabase-js`) sur la page de validation via un `<script type="module">` ESM depuis un CDN : `supabase.auth.signInWithOtp({ email })` pour le lien magique, puis `supabase.auth.getSession()`. Le jeu public, lui, reste sans dépendance (simple `fetch`).

---

## 7. Comment le jeu lit Supabase (modèle à réutiliser)

Le jeu écrit déjà dans Supabase avec ce pattern (voir `sendStat`, ligne ~1235) :

```js
fetch(url + '/rest/v1/feedback', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'apikey': ANON_KEY,
    'Authorization': 'Bearer ' + ANON_KEY,
    'Prefer': 'return=minimal'
  },
  body: JSON.stringify(row)
});
```

Pour **lire**, c'est un `GET` avec les mêmes en-têtes :

```js
const r = await fetch(url + '/rest/v1/solutions?select=*&order=ordre', {
  headers: { 'apikey': ANON_KEY, 'Authorization': 'Bearer ' + ANON_KEY }
});
const rows = await r.json(); // tableau d'objets
```

Pas besoin de filtrer sur `statut` dans la requête du jeu : la **RLS anon** (§5) ne renvoie déjà que les lignes `valide`. Le jeu ne voit jamais un brouillon ni un rejet.

---

## 8. Intégration côté jeu (`index.html`)

L'idée : au chargement, récupérer les 3 tables et **remplir** les structures du jeu, **avant** le premier rendu. Le jeu ne démarre qu'une fois les données prêtes.

Points d'attention :

1. **Une fois la base branchée, les tableaux en dur ne servent plus.** Tu peux les vider (ou les retirer) : la source de vérité est Supabase. Garde-en éventuellement un jeu minimal pour tes tests locaux, mais ce n'est plus la version de prod.
2. Les objets `SOL`, `SOL_COUT`, `SOL_LOCK` sont **dérivés** de `SOLUTIONS`. Après avoir chargé depuis Supabase, il faut **reconstruire** `SOL` (`Object.fromEntries(...)`) et reporter `cout` / `competence_requise` sur chaque solution (aujourd'hui fait lignes ~865-868). Regroupe cette logique dans une fonction pour la rejouer après le fetch.
3. **Mapping des noms** : la base utilise `ici_id`, `categorie`, `capacites`, `flux_in/out`, `competence_requise`, `protege` ; le JS attend `ici`, `cat`, `cap`, `flux:{in,out}`, `lock`, `protects`. Fais une petite fonction d'adaptation (base → format JS) pour ne pas avoir à renommer partout dans le code. Le champ `complexite` est nouveau (voir §4) : décide de son usage en jeu avec Romain.
4. Le point d'entrée du jeu (le `DOMContentLoaded` / la fonction d'init) doit devenir **async** : `await loadData()` avant de lancer l'écran d'accueil. Affiche un petit écran de chargement pendant le fetch, et un message d'erreur clair (avec bouton « réessayer ») si Supabase ne répond pas.

Squelette indicatif :

```js
const EVAD_DATA_SUPABASE = { url:'https://...supabase.co', anonKey:'sb_publishable_...' };

async function loadData(){
  const cfg = EVAD_DATA_SUPABASE;
  const h = { apikey: cfg.anonKey, Authorization: 'Bearer ' + cfg.anonKey };
  const [sol, ici, comp] = await Promise.all([
    fetch(cfg.url+'/rest/v1/solutions?select=*&order=ordre', {headers:h}).then(r=>r.json()),
    fetch(cfg.url+'/rest/v1/ici?select=*', {headers:h}).then(r=>r.json()),
    fetch(cfg.url+'/rest/v1/competences?select=*', {headers:h}).then(r=>r.json()),
  ]);
  applySolutions(sol);    // → remap + reconstruit SOL / SOL_COUT / SOL_LOCK
  applyIci(ici);
  applyCompetences(comp); // → répartit pilote / batisseur
}

// init
try {
  showLoader();
  await loadData();
  startGame();
} catch(e){
  showError('Impossible de charger les données du jeu.', e); // + bouton réessayer
}
```

---

## 9. Critères d'acceptation (Definition of Done)

- [ ] Les 3 tables existent avec la RLS : `anon` ne lit que `statut = 'valide'`, `authenticated` (CR) lit tout et peut mettre à jour.
- [ ] Toutes les données actuelles du HTML sont présentes dans Supabase au statut `valide` (13 solutions, 8 ICI, compétences Pilote + Bâtisseur).
- [ ] Le jeu chargé en ligne affiche **exactement** les mêmes cartes qu'avant (aucune régression visuelle).
- [ ] Modifier une ligne dans Supabase (ex. renommer une solution) se voit dans le jeu **après rechargement**, sans toucher au code.
- [ ] Si Supabase ne répond pas, le jeu affiche un **message d'erreur clair** (pas d'écran vide) avec une possibilité de réessayer.
- [ ] **Interface CR** : un membre se connecte (lien magique), voit les entrées `en_revue`, peut **valider** (elles apparaissent alors dans le jeu) et **rejeter** (avec motif).
- [ ] Un utilisateur **non connecté** ne peut **rien modifier** (les policies de mise à jour refusent `anon`).
- [ ] Aucune clé `service_role` dans le dépôt.
- [ ] Doc `supabase/README.md` livrée (connexion, ajout d'une carte, validation par le CR).
