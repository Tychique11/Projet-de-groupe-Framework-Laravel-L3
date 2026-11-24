# CONTRAT.md — Gestion des PV de délibération (ENEAM)

> Source de vérité commune. Si quelque chose manque ou semble faux ici : on le dit dans le groupe, **Dev A** met à jour ce fichier, puis chacun s'adapte. Personne n'invente de son côté.

Stack : Laravel 13 (PHP 8.3+, à vérifier avec `composer why-not php`), MySQL 8, Blade + Vite, Breeze (Blade) sans inscription publique.
Package : `spatie/laravel-activitylog` uniquement (audit). Les rôles sont gérés « maison » : colonne `role` sur `users` + middleware + Policy + Gates (pas de `spatie/laravel-permission`).
Langue : interface en français, code (classes, méthodes, variables) en français pour le domaine métier, noms de tables en français pluriel snake_case comme ci-dessous.

## 1. Rôles

| Rôle (nom exact) | Qui | Droits |
|---|---|---|
| `da` | Directeur Adjoint (admin) | Tout : voir/uploader/modifier/supprimer/restaurer des PV partout ; créer/désactiver/supprimer les comptes CD ; gérer les référentiels ; voir le journal d'audit |
| `chef_departement` | Chef de département (CD) | Voir/uploader/modifier/supprimer (soft) les PV **uniquement des filières qui lui sont affectées** (table `filiere_user`). Aucun accès aux comptes, référentiels, corbeille, audit |

Le rôle est stocké dans `users.role` (valeurs de l'enum `RoleNom`). Il n'y a pas de table de permissions.
Gates (définies par Dev A dans `AppServiceProvider`, réservées au `da`) : `gerer-utilisateurs`, `gerer-referentiels`, `voir-audit`, `voir-corbeille`.
Droits sur les PV : uniquement via la **PvPolicy** (`viewAny`, `view`, `create`, `update`, `delete`, `restore`), qui vérifie aussi l'appartenance à la filière (voir §5).
Dans les vues : `@can('create', App\Models\Pv::class)`, `@can('update', $pv)`, `@can('delete', $pv)`, `@can('gerer-referentiels')`, `@can('voir-audit')`. Ne jamais tester le rôle à la main dans une vue.

## 2. Schéma de base de données (migrations exactes — Dev B)

Toutes les tables ont `id` (bigint) + `timestamps` sauf mention contraire.

**users** (table Laravel de base, modifiée) : `name`, `email` unique, `password`, `role` string(20) default `chef_departement`, `telephone` string(20) null, `is_active` boolean default true, `created_by` FK users null, `deleted_at` (softDeletes).

**departements** : `code` string(20) unique, `nom` string.

**filieres** : `departement_id` FK, `code` string(20), `nom` string, `cycle` string(20) (valeurs de l'enum `Cycle`, §4), `annee_debut` smallint null, `annee_fin` smallint null (renseigné si la filière est fermée), `actif` boolean default true, `deleted_at`. Unique `(code, cycle)`.

**sous_filieres** : `filiere_id` FK, `code` string(20), `nom` string, `actif` boolean default true, `deleted_at`. Unique `(filiere_id, code)`.

**annees_academiques** : `libelle` string(9) unique (ex. `2023-2024`), `annee_debut` smallint, `annee_fin` smallint.

**niveaux** : `code` string(10) unique (ex. `L1`, `M2`), `libelle` string, `cycle` string(20), `ordre` tinyint.

**semestres** : `code` string(4) unique (`S1`…`S10`), `numero` tinyint.

**niveau_semestre** (pivot, sans timestamps) : `niveau_id`, `semestre_id`, PK composite. (Sert à valider qu'un semestre est cohérent avec le niveau : L1→S1,S2 ; L2→S3,S4 ; L3→S5,S6 ; M1→S7,S8 ; M2→S9,S10 ; BTS1→S1,S2 ; BTS2→S3,S4 ; LP→S5,S6.)

**filiere_user** (pivot) : `user_id` FK, `filiere_id` FK, unique `(user_id, filiere_id)`, timestamps.

**pvs** :
`annee_academique_id` FK, `filiere_id` FK, `sous_filiere_id` FK null, `niveau_id` FK, `semestre_id` FK,
`groupe` string(10) null (classe/répartition : `A`, `B`, `C`, `D`…),
`session` string(12) default `normale` (`normale` | `rattrapage`),
`date_deliberation` date,
`fichier_path` string, `fichier_nom_original` string, `fichier_mime` string(100), `fichier_taille` unsignedBigInteger, `fichier_hash` char(64) (SHA-256, détection de doublon),
`observations` text null,
`created_by` FK users, `updated_by` FK users null, `deleted_by` FK users null,
`deleted_at` (softDeletes).
Index : `(filiere_id, annee_academique_id, niveau_id, semestre_id)` ; index sur `fichier_hash`.
Aucune FK vers `users` n'utilise `cascadeOnDelete` (on ne perd jamais l'auteur d'un PV).

**activity_log** : migration publiée par le package Spatie activitylog (déjà dans le repo, ne pas modifier).

Règles transversales : jamais de `forceDelete` sur `pvs`, `users`, `filieres`, `sous_filieres`. Les relations vers `users` utilisent `withTrashed()`.

## 3. Modèles (Dev A) — noms de relations à respecter

- `Pv` : `anneeAcademique()`, `filiere()`, `sousFiliere()`, `niveau()`, `semestre()`, `auteur()` (created_by), `modificateur()` (updated_by), `suppresseur()` (deleted_by). Scopes : `accessiblePar(User $u)`, `filtrer(array $criteres)` (clés : `annee_academique_id`, `filiere_id`, `sous_filiere_id`, `niveau_id`, `semestre_id`, `groupe`, `session`). Accesseur `fichier_taille_lisible`. Trait `SoftDeletes` + `LogsActivity`.
- `User` : `filieres()` (belongsToMany), `pvs()`, méthodes `estDa()`, `estChefDepartement()` (lisent la colonne `role`), `idsFilieresAutorisees()`. Traits `SoftDeletes`, `LogsActivity`.
- `Departement` (hasMany `filieres`), `Filiere` (belongsTo `departement`, hasMany `sousFilieres`, `pvs`, belongsToMany `users`), `SousFiliere`, `AnneeAcademique`, `Niveau` (belongsToMany `semestres`), `Semestre`.
- Un trait `App\Models\Concerns\TracksAuthor` remplit automatiquement `created_by`, `updated_by`, `deleted_by` depuis `auth()->id()` (events `creating`, `updating`, `deleting`).

## 4. Enums (Dev A) — `App\Enums`

`Cycle` (string backed) : `BTS`, `DTSS`, `LICENCE`, `LICENCE_PRO`, `MASTER`, `ANCIEN_CYCLE` (formations d'avant le LMD).
`SessionPv` : `normale`, `rattrapage`.
`RoleNom` : `da`, `chef_departement`.

## 5. Autorisations (Dev A)

`PvPolicy` : `viewAny` (rôle connecté actif), `view/update/delete` (DA : toujours ; CD : `pv.filiere_id ∈ user->idsFilieresAutorisees()`), `create` (DA : toujours ; CD : uniquement pour une filière autorisée → vérifié aussi dans la FormRequest), `restore` / `forceDelete` (DA seulement ; `forceDelete` retourne toujours `false`).
Middleware (écrits par Dev A, alias enregistrés dans `bootstrap/app.php`) : `actif` (déconnecte si `is_active = false`), `role:da` (`VerifierRole`, lit `users.role`) sur toute la zone `admin.*` sauf mention contraire.
Les contrôleurs appellent `$this->authorize(...)` et utilisent `Pv::accessiblePar(auth()->user())` pour toute liste.

## 6. Routes et contrôleurs (Dev A écrit les fichiers `routes/*.php` ; Dev B implémente les méthodes)

Tous les noms de route ci-dessous sont **figés**. Les vues utilisent uniquement `route('nom')`.

| Nom de route | Méthode + URL | Contrôleur@méthode | Accès |
|---|---|---|---|
| `dashboard` | GET `/` | `DashboardController@index` | connecté |
| `pv.index` | GET `/pv` | `PvController@index` | connecté (filtres en query string) |
| `pv.create` | GET `/pv/creer` | `PvController@create` | policy create |
| `pv.store` | POST `/pv` | `PvController@store` | policy create |
| `pv.show` | GET `/pv/{pv}` | `PvController@show` | policy view |
| `pv.apercu` | GET `/pv/{pv}/apercu` | `PvController@apercu` (flux inline, journalise « consulté ») | policy view |
| `pv.telecharger` | GET `/pv/{pv}/telecharger` | `PvController@telecharger` (journalise « téléchargé ») | policy view |
| `pv.edit` / `pv.update` | GET `/pv/{pv}/modifier` / PUT `/pv/{pv}` | `PvController@edit` / `@update` | policy update |
| `pv.destroy` | DELETE `/pv/{pv}` | `PvController@destroy` (soft delete) | policy delete |
| `pv.corbeille` | GET `/corbeille/pv` | `PvController@corbeille` | da |
| `pv.restaurer` | PATCH `/corbeille/pv/{id}/restaurer` | `PvController@restaurer` | da |
| `ajax.sous-filieres` | GET `/ajax/filieres/{filiere}/sous-filieres` | `Ajax\ReferentielController@sousFilieres` | connecté (JSON `[{id,code,nom}]`) |
| `ajax.semestres` | GET `/ajax/niveaux/{niveau}/semestres` | `Ajax\ReferentielController@semestres` | connecté (JSON `[{id,code}]`) |
| `ajax.niveaux` | GET `/ajax/filieres/{filiere}/niveaux` | `Ajax\ReferentielController@niveaux` | connecté (JSON, filtré sur le `cycle` de la filière) |
| `admin.utilisateurs.*` | resource `/admin/utilisateurs` (+ `admin.utilisateurs.basculer` PATCH `/{utilisateur}/basculer`, `admin.utilisateurs.restaurer`) | `Admin\UtilisateurController` | da |
| `admin.departements.*`, `admin.filieres.*`, `admin.sous-filieres.*`, `admin.annees.*` | resources `/admin/...` | `Admin\DepartementController`, `FiliereController`, `SousFiliereController`, `AnneeAcademiqueController` | da |
| `admin.audit.index` | GET `/admin/audit` | `Admin\AuditController@index` (filtres : utilisateur, date, événement, objet) | da |

Un CD n'a pas de sélecteur « filière globale » : son formulaire ne propose que ses filières. Le DA choisit d'abord la filière (ou toutes), puis affine.

## 7. Variables passées aux vues (Dev B → Dev C)

- `pv.index` : `$pvs` (LengthAwarePaginator de `Pv` avec relations chargées), `$criteres` (array des filtres actifs), `$annees`, `$filieres` (déjà restreintes à l'utilisateur), `$niveaux`, `$groupes` (`['A','B','C','D','E','F']`).
- `pv.create` / `pv.edit` : mêmes listes + `$pv` (edit seulement) + `$sessions` (`SessionPv::cases()`).
- `pv.show` : `$pv`.  `pv.corbeille` : `$pvs` paginés (avec `suppresseur`).
- `dashboard` : `$stats` = `['total_pv' => int, 'pv_par_filiere' => Collection, 'derniers_pv' => Collection<Pv>]`.
- `admin.utilisateurs.*` : `$utilisateurs`, `$utilisateur`, `$filieres` (pour affectation). `admin.audit.index` : `$activites` (paginé, avec `causer`, `subject`).
- Messages flash : `session('succes')`, `session('erreur')`. Erreurs de validation : `$errors` standard.
- Champs du formulaire PV (attributs `name` exacts) : `annee_academique_id`, `filiere_id`, `sous_filiere_id`, `niveau_id`, `semestre_id`, `groupe`, `session`, `date_deliberation`, `observations`, `fichier`.

## 8. Fichiers : qui possède quoi (on ne touche pas aux fichiers des autres)

| Dossier / fichier | Propriétaire |
|---|---|
| `app/Models/**`, `app/Enums/**`, `app/Policies/**`, `app/Providers/**`, `app/Http/Middleware/**`, `routes/**`, `bootstrap/app.php`, `composer.json`, `.env.example`, `config/activitylog.php`, `database/seeders/UtilisateurDaSeeder.php` (crée le compte DA initial), `database/seeders/DatabaseSeeder.php`, `app/Http/Controllers/Admin/UtilisateurController.php`, `app/Http/Controllers/Admin/AuditController.php`, `tests/Feature/Autorisation*`, `CONTRAT.md`, `.github/**` | **Dev A** (chef d'équipe / fondations) |
| `database/migrations/**`, `database/factories/**`, `database/seeders/**` (sauf ceux de Dev A), `app/Http/Controllers/**` (sauf ceux de Dev A), `app/Http/Requests/**`, `app/Services/**`, `config/filesystems.php` | **Dev B** (Keryann : données & contrôleurs) |
| `resources/views/**`, `resources/css/**`, `resources/js/**`, `lang/fr/**`, `docs/**` (MCD, UML, captures) | **Dev C** (Mouchfiq : vues & modélisation) |

Besoin de modifier un fichier d'un autre ? On le demande dans le groupe avec la modif exacte souhaitée ; le propriétaire la fait.

## 9. Stockage des fichiers PV

Disque privé `pv` (racine `storage/app/private/pv`), **jamais** `public`. Chemin : `{annee}/{filiere_code}/{uuid}.{ext}`. Types acceptés : PDF, JPG, PNG ; taille max 20 Mo. Accès uniquement via `pv.apercu` / `pv.telecharger` (contrôleur + policy). Service : `App\Services\PvStockageService` (`enregistrer(UploadedFile): array`, `supprimerDefinitivement` n'existe pas).

## 10. Git

Branches : `main` (protégée, rien n'y est poussé directement), `develop` (intégration), `feat/<initiales>/<sujet>` (ex. `feat/kc/migrations`). PR vers `develop`, 1 relecture minimum (par quelqu'un d'autre que l'auteur). Commits en français, courts, à l'impératif : `Ajoute la migration pvs`. `git pull --rebase origin develop` avant chaque push. `.env` jamais commité. Après chaque merge dans `develop`, `php artisan migrate:fresh --seed` doit passer sur la machine de chacun.

Ordre de livraison : (0) Dev A pousse le squelette → (1) en parallèle : migrations+seeders (B), modèles+policies+routes (A), vues avec données factices (C) → (2) contrôleurs branchés (B), vues branchées (C) → (3) intégration, tests d'autorisation, recette.
