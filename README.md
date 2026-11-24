# esgi-ci-cd

Exercices de CI/CD avec **GitHub Actions** réalisés dans le cadre du cours ESGI.

Le dépôt contient trois workflows, du plus simple au plus complet : valider le projet à chaque push, produire un artefact, puis construire, tester et déployer sur plusieurs environnements.

## Prérequis

- Node.js (les workflows utilisent la version `26.10.0`, et les versions 20 et 22 pour les tests)
- npm

## Installation

```bash
npm install
```

> En local on utilise `npm install`, dans les workflows on utilise `npm ci` (voir [plus bas](#npm-install-ou-npm-ci-)).

## Les workflows

Tous les fichiers se trouvent dans [.github/workflows/](.github/workflows/).

### Valider le projet ([validate-project.yml](.github/workflows/validate-project.yml))

- **Déclenchement** : à chaque `push`, sur toutes les branches.
- **Étapes** : récupération du code, installation de Node, cache npm, `npm ci`, `npm test`.
- **Résultat** : un résumé est affiché sur la page du run.

### Produire un artefact ([artefact.yml](.github/workflows/artefact.yml))

- **Déclenchement** : manuel, depuis l'onglet **Actions** (`workflow_dispatch`).
- **Étapes** : `npm ci`, `npm run build`, vérification que le dossier `dist/` existe.
- **Résultat** : le dossier `dist/` est sauvegardé sous le nom d'artefact `build-artifact`, téléchargeable depuis la page du run. Un résumé affiche la branche, le commit, les versions de Node et npm, et la taille du build.

### Exercice 3 : déployer et tester ([deployement-exercice-3.yml](.github/workflows/deployement-exercice-3.yml))

- **Déclenchement** : manuel, avec le choix de l'environnement cible (`staging` ou `production`).
- **Trois jobs enchaînés** avec `needs` :

```
build  ──►  tests (matrice ×4)  ──►  deploy
```

| Job | Rôle |
|---|---|
| `build` | Construit le projet et envoie `dist/` comme artefact `application`. |
| `tests` | Lance `npm test` sur 4 combinaisons : Node 20/22 × Ubuntu/Windows. Au maximum 2 tournent en même temps, et un échec n'annule pas les autres. |
| `deploy` | Récupère l'artefact et « déploie » sur l'environnement choisi, en utilisant sa variable `API_URL`. |

**Configuration nécessaire** : dans **Settings → Environments**, créer les environnements `staging` et `production`, avec dans chacun une variable `API_URL`.

## Notions abordées

### `npm install` ou `npm ci` ?

| | `npm install` | `npm ci` |
|---|---|---|
| Utilisation | En local | En CI |
| Versions installées | Selon `package.json` (plages comme `^4.3.3`) | Exactement celles de `package-lock.json` |
| Modifie `package-lock.json` | Oui, ça peut arriver | Jamais |

👉 Toujours commiter `package.json` **et** `package-lock.json` ensemble, sinon `npm ci` échoue.

### Cache ou artefact ?

- **Cache** (`actions/cache`) : garde les dépendances téléchargées (`~/.npm`) d'un run à l'autre pour **aller plus vite**. Si on le perd, le workflow est juste plus lent.
- **Artefact** (`upload-artifact` / `download-artifact`) : transmet **ce que le workflow produit** (`dist/`, rapports de tests…) d'un job à l'autre. Chaque job tourne sur une machine neuve, donc sans artefact le job `deploy` ne trouverait pas `dist/`.

### Variables ou secrets ?

- **`vars`** (ex. `vars.API_URL`) : valeur en clair, visible dans les logs. Pour une URL, un nom de région…
- **`secrets`** (ex. `secrets.API_TOKEN`) : valeur chiffrée et masquée (`***`) dans les logs. Pour un token, un mot de passe…

Les deux peuvent être définis au niveau du dépôt ou par environnement (`staging` / `production`).

### Autres mots-clés utilisés

| Mot-clé | Rôle |
|---|---|
| `needs: build` | Le job attend la réussite de `build`. Il ne transmet aucun fichier. |
| `if: always()` | L'étape s'exécute même si une étape précédente a échoué. |
| `strategy.matrix` | Lance le même job sur plusieurs combinaisons (versions, OS). |
| `permissions: contents: read` | Limite les droits du workflow à la lecture du dépôt. |
| `$GITHUB_STEP_SUMMARY` | Fichier dont le contenu s'affiche en résumé du run. `>` écrase le contenu, `>>` ajoute à la suite. |

## À savoir

Les workflows appellent `npm run build` et `npm test`. Ces scripts doivent être définis dans la section `scripts` de `package.json` pour que les jobs réussissent.
