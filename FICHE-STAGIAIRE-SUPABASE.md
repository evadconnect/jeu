# Fiche stagiaire · Externaliser les données du jeu vers Supabase

**Projet :** EVAD Le Jeu (`evadconnect/jeu`, fichier `index.html`)
**Mission :** sortir les données **Solutions**, **ICI** et **Compétences** du HTML pour les gérer dans **3 tables Supabase**, puis les rebrancher dans le jeu.
**Durée estimée :** 3 à 5 jours.

---

## 1. Contexte

Le jeu est un **fichier HTML autonome** (aucun build, aucune dépendance, tout en JavaScript inline). Aujourd'hui, les cartes du jeu sont écrites en dur dans `index.html` sous forme de tableaux/objets JavaScript :

- `SOLUTIONS` (les solutions à installer) : ligne ~821
- `ICI` (les indicateurs de mesure) : ligne ~909
- `COMPETENCES` (compétences Bâtisseur) : ligne ~927 et `PCOMP` (compétences Pilote) : ligne ~871

Le but est que l'équipe puisse **ajouter ou modifier une carte sans toucher au code**, via Supabase.

Le jeu sait déjà parler à Supabase : le formulaire de feedback fait un `INSERT` REST direct (voir `EVAD_STATS_SUPABASE`, ligne ~1226). Tu vas t'appuyer sur le même mécanisme, mais en **lecture**.

### Contrainte non négociable

Le jeu doit **rester jouable hors-ligne**. L'appel Supabase est une **amélioration progressive** : si le réseau ou Supabase ne répond pas, le jeu retombe sur les données embarquées (celles déjà dans le HTML). On ne casse jamais l'expérience si Supabase est indisponible.

---

## 2. Ce que tu dois livrer

1. Un script SQL (`supabase/schema.sql`) qui crée les 3 tables + les politiques RLS.
2. Un script SQL (`supabase/seed.sql`) qui insère les données actuelles (reprises depuis le HTML).
3. La modification de `index.html` : au démarrage, le jeu charge les 3 tables depuis Supabase et remplace les données embarquées, **avec fallback** si l'appel échoue.
4. Une courte doc (`supabase/README.md`) : comment se connecter, où sont les clés, comment ajouter une carte.

---

## 3. Accès & sécurité

- **Projet Supabase** existant : `https://lmhhrccmgebztioesmik.supabase.co` (demande à Romain l'accès au dashboard).
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
| `categorie` | `text` | `alim` | une de : `fraicheur`, `alim`, `energie`, `circulaire` (voir objet `CATS`) |
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
- `solutions.categorie` → valeur de `CATS` (garde ça en simple `text` + `check`, ou fais une table `categories` si tu veux)
- `solutions.competence_requise` → `competences.id` (famille `pilote`)
- `competences.couvre[]` référence des `solutions.id` (famille `batisseur`)

---

## 5. SQL de départ (à adapter)

```sql
-- ICI
create table ici (
  id text primary key,
  nom text not null,
  icone text,
  unite text,
  referentiels text[] default '{}'
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
  couleur text
);

-- SOLUTIONS
create table solutions (
  id text primary key,
  nom text not null,
  icone text,
  categorie text check (categorie in ('fraicheur','alim','energie','circulaire')),
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
  ordre int
);

-- RLS : lecture publique seule
alter table ici enable row level security;
alter table competences enable row level security;
alter table solutions enable row level security;

create policy "lecture publique ici"        on ici         for select to anon using (true);
create policy "lecture publique competences" on competences for select to anon using (true);
create policy "lecture publique solutions"   on solutions   for select to anon using (true);
```

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
  (id, nom, icone, categorie, description, capacites, vadance, graines, cout, ici_id, flux_in, flux_out, dormant, competence_requise, ordre) values
  ('potager', 'Potager en pleine terre', '🥬', 'alim',
   'Légumes de saison en circuit court.', '{terre}', 4, 3, 3, 'biodiv',
   '{compost,eau}', '{legumes,biodechets}', true, null, 10);
```

---

## 6. Comment le jeu lit Supabase (modèle à réutiliser)

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

---

## 7. Intégration côté jeu (`index.html`)

L'idée : au chargement, tenter de récupérer les 3 tables, puis **écraser** les données embarquées avant le premier rendu. Si ça échoue, on garde les données embarquées.

Points d'attention :

1. **Garde les tableaux actuels comme fallback.** Ne supprime pas `SOLUTIONS` / `ICI` / `COMPETENCES` / `PCOMP` du HTML : ils restent la version hors-ligne.
2. Les objets `SOL`, `SOL_COUT`, `SOL_LOCK` sont **dérivés** de `SOLUTIONS`. Après avoir chargé depuis Supabase, il faut **reconstruire** `SOL` (`Object.fromEntries(...)`) et reporter `cout` / `competence_requise` sur chaque solution (aujourd'hui fait lignes ~865-868). Regroupe cette logique dans une fonction pour pouvoir la rejouer après le fetch.
3. **Mapping des noms** : la base utilise `ici_id`, `capacites`, `flux_in/out`, `competence_requise`, `protege` ; le JS attend `ici`, `cap`, `flux:{in,out}`, `lock`, `protects`. Fais une petite fonction d'adaptation (base → format JS) pour ne pas avoir à renommer partout dans le code.
4. Le point d'entrée du jeu (le `DOMContentLoaded` / la fonction d'init) doit devenir **async** : `await loadData()` avant de lancer l'écran d'accueil. Prévois un timeout court (2-3 s) pour ne pas bloquer si Supabase traîne.

Squelette indicatif :

```js
const EVAD_DATA_SUPABASE = { url:'https://...supabase.co', anonKey:'sb_publishable_...' };

async function loadData(){
  const cfg = EVAD_DATA_SUPABASE;
  if(!cfg.url || !cfg.anonKey) return; // pas de config → fallback embarqué
  try {
    const h = { apikey: cfg.anonKey, Authorization: 'Bearer ' + cfg.anonKey };
    const [sol, ici, comp] = await Promise.all([
      fetch(cfg.url+'/rest/v1/solutions?select=*&order=ordre', {headers:h}).then(r=>r.json()),
      fetch(cfg.url+'/rest/v1/ici?select=*', {headers:h}).then(r=>r.json()),
      fetch(cfg.url+'/rest/v1/competences?select=*', {headers:h}).then(r=>r.json()),
    ]);
    if(Array.isArray(sol) && sol.length) applySolutions(sol);   // → remap + reconstruit SOL
    if(Array.isArray(ici) && ici.length) applyIci(ici);
    if(Array.isArray(comp) && comp.length) applyCompetences(comp); // → répartit pilote/batisseur
  } catch(e){
    console.warn('Supabase indisponible, données embarquées utilisées', e);
  }
}
```

---

## 8. Critères d'acceptation (Definition of Done)

- [ ] Les 3 tables existent avec RLS **lecture seule** anon activée.
- [ ] Toutes les données actuelles du HTML sont présentes dans Supabase (13 solutions, 8 ICI, compétences Pilote + Bâtisseur).
- [ ] Le jeu chargé en ligne affiche **exactement** les mêmes cartes qu'avant (aucune régression visuelle).
- [ ] Modifier une ligne dans Supabase (ex. renommer une solution) se voit dans le jeu **après rechargement**, sans toucher au code.
- [ ] **Test hors-ligne** : couper le réseau → le jeu fonctionne toujours avec les cartes embarquées.
- [ ] Aucune clé `service_role` dans le dépôt.
- [ ] Doc `supabase/README.md` livrée (connexion, ajout d'une carte).

## 9. Étapes conseillées

1. Lire ce fichier + repérer les 4 blocs de données dans `index.html` (`SOLUTIONS`, `ICI`, `COMPETENCES`, `PCOMP`) et leurs dérivés (`SOL`, `SOL_COUT`, `SOL_LOCK`).
2. Écrire `schema.sql`, l'exécuter dans le SQL Editor de Supabase.
3. Générer `seed.sql` depuis le HTML (un petit script Node/Python qui transforme les tableaux JS en `INSERT` fait gagner du temps et évite les fautes de recopie).
4. Vérifier la lecture via l'URL REST directement dans le navigateur.
5. Brancher `loadData()` + les fonctions `apply*` avec le remapping, tester en ligne.
6. Tester le fallback hors-ligne.
7. Écrire la doc, ouvrir une PR.

## 10. Questions à poser avant de démarrer

- Accès au dashboard Supabase (rôle éditeur) ?
- Confirme-t-on **une seule** table `competences` (avec `profil`) plutôt que deux tables séparées ?
- Faut-il aussi externaliser les **catégories** (`CATS`) et **référentiels** (`REFS`), ou on les garde en dur pour la V1 ?
