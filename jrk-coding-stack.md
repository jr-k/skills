---
name: jrk-coding-stack
description: Conçoit, migre et révise des applications avec Laravel, PostgreSQL, Inertia.js, React TypeScript et styled-components. À utiliser pour créer une application full-stack, remplacer Next.js ou Supabase, préserver une parité fonctionnelle, structurer un backend Laravel typé, construire les images Docker dev/prod et publier sur GHCR.
---

# JRK Coding Stack

## Objectif

Construire des applications monolithiques modernes avec :

- Laravel pour le backend, le domaine, l’authentification et les accès aux données ;
- PostgreSQL comme base de données ;
- Inertia.js comme frontière HTTP entre Laravel et React ;
- React sans framework frontend additionnel, en TypeScript strict ;
- styled-components pour toute la présentation ;
- Docker Compose pour le développement et la production ;
- une image Docker multi-stage publiée sur GHCR par GitHub Actions.

Privilégier une architecture explicite, typée et pragmatique. Préserver le comportement et le design lors d’une migration avant d’améliorer l’architecture.

## Principes non négociables

1. Laravel est l’unique backend.
2. React ne porte aucune règle métier serveur.
3. Inertia transporte des props sérialisables et orchestre les navigations.
4. PostgreSQL est accédé par Laravel. Ne conserver aucun client Supabase, RLS Supabase, RPC Supabase ou dépendance `@supabase/*`.
5. Les contrôleurs sont des adaptateurs HTTP minces.
6. Les entrées HTTP sont validées par des `FormRequest`.
7. Les cas d’usage reçoivent des DTO `readonly`, pas des `Request`, `UploadedFile` ou tableaux implicites.
8. Les intégrations externes passent par des ports Application et des adaptateurs Infrastructure.
9. Les secrets sont lus depuis la configuration Laravel, jamais directement avec `env()` hors des fichiers `config/`.
10. Une migration de stack conserve routes, design, textes, statuts HTTP, redirections, cookies, sessions et fonctionnalités.

## Architecture Laravel

Utiliser cette structure :

```text
app/
├── Application/
│   └── <Domain>/
│       ├── Actions/
│       ├── Contracts/
│       ├── Data/
│       ├── Presenters/
│       └── Queries/
├── Domain/
│   └── <Domain>/
├── Http/
│   ├── Adapters/
│   ├── Controllers/
│   ├── Middleware/
│   └── Requests/
├── Infrastructure/
│   └── <Domain>/
├── Models/
└── Providers/
```

### Contrôleurs

Un contrôleur peut :

- autoriser l’action ;
- recevoir un `FormRequest` ;
- appeler une Query ou une Action ;
- retourner une réponse Inertia, JSON ou une redirection.

Un contrôleur ne doit pas :

- appeler `DB::` ;
- ouvrir une transaction ;
- contenir une règle métier ;
- normaliser un payload volumineux ;
- appeler directement Slack, n8n ou une autre API ;
- construire un grand arbre de props.

Déplacer l’adaptation HTTP dans un `FormRequest::payload()`. Déplacer les projections dans une Query ou un Presenter.

### Actions et Queries

- Une Action représente une mutation métier.
- Une Query représente une lecture.
- Une mutation multi-écriture utilise une transaction.
- Les validations concurrentes sont répétées sous verrou dans la transaction.
- Les appels réseau ne doivent pas rester sous verrou PostgreSQL, sauf nécessité d’atomicité documentée et vérifiée.
- Les effets externes sont déclenchés après commit lorsque le métier le permet.

### DTO et enums

- Utiliser des classes `final readonly`.
- Employer des enums backed pour les vocabulaires fermés persistés.
- Ajouter les casts Eloquent correspondants.
- Documenter précisément les formes de tableaux qui restent nécessaires avec PHPStan.
- Refuser les états impossibles dans le constructeur ou la factory du DTO.
- Ne jamais faire dépendre Application de `Illuminate\Http`.

### Ports et adaptateurs

Créer une interface dans `Application/<Domain>/Contracts` pour :

- API externes ;
- stockage de fichiers ;
- signatures ou chiffrement ;
- persistance spécialisée ayant une vraie frontière ;
- infrastructure ayant plusieurs implémentations possibles.

Implémenter le port dans `Infrastructure/<Domain>` et déclarer le binding dans `AppServiceProvider`.

Ne pas créer de repository générique pour chaque modèle. Eloquent reste acceptable dans les Actions et Queries simples.

### Eloquent et PostgreSQL

- Les migrations Laravel sont la source du schéma.
- Toute évolution utilise une nouvelle migration.
- Utiliser `timestamptz` pour les instants.
- Déclarer explicitement `$fillable` ou `$guarded`.
- Employer les contraintes PostgreSQL pour l’intégrité, pas pour cacher des règles métier.
- Éviter les N+1 avec eager loading ou lectures groupées.
- Paginer en SQL au lieu de charger un corpus entier.
- Ne pas sérialiser de données sensibles dans les props Inertia ou les journaux d’audit.

## Frontend React avec Inertia

Utiliser React TypeScript sans Next.js, Remix ou autre framework frontend.

```text
resources/js/
├── Components/
│   └── MyComponent/
│       ├── MyComponent.tsx
│       └── styled.ts
└── Pages/
    └── MyPage/
        ├── MyPage.tsx
        └── styled.ts
```

### Convention des composants

Pour `MyComponent` :

```tsx
import * as S from './styled';

export function MyComponent() {
    return (
        <S.MyComponentContainer>
            <S.Title>Contenu</S.Title>
        </S.MyComponentContainer>
    );
}
```

Dans `styled.ts` :

```tsx
import styled from 'styled-components';

export const MyComponentContainer = styled.section`
    display: grid;
    gap: 1rem;
`;

export const Title = styled.h2`
    margin: 0;
`;
```

Règles :

- toujours `import * as S from './styled';` ;
- le nœud racine est `S.<ComponentName>Container` ;
- aucun élément HTML brut dans le JSX : utiliser un styled component, y compris pour `div`, `span`, `button`, `summary` et les champs natifs ;
- aucun `className` ni prop `style` ;
- aucun fichier `index.ts` ;
- un dossier par composant ;
- 250 lignes maximum par composant ;
- extraire des sous-composants dans des sous-dossiers ;
- écrire chaque styled component sur plusieurs lignes ;
- ne pas condenser les déclarations CSS sur une ligne ;
- conserver TypeScript strict et valider les réponses réseau inconnues avant usage.

### Inertia

- Les Pages reçoivent des props typées.
- Les props volumineuses sont préparées côté serveur.
- Utiliser les navigations et formulaires Inertia pour les parcours applicatifs.
- Réserver `fetch` aux réponses JSON réellement nécessaires.
- Ne jamais exposer un modèle Eloquent entier si une projection explicite suffit.
- Ne jamais envoyer de secret ou de donnée personnelle inutile au navigateur.

## Authentification et autorisation

- Laravel possède la session applicative.
- Une intégration SSO comme Crossguard utilise Socialite ou un provider dédié.
- Lire les URLs, identifiants, secret, redirect URI, scopes et rôles directement depuis `config/services.php`.
- Ne pas créer de variables d’environnement qui se référencent en cascade quand des valeurs explicites sont attendues.
- Mapper les rôles SSO vers les profils applicatifs dans un service dédié.
- Répéter l’autorisation dans les Policies, middleware ou Actions sensibles.
- Le frontend masque les gestes interdits, mais le backend reste l’autorité.

## Fichiers

- Valider les uploads dans un `FormRequest`.
- Adapter `UploadedFile` à la frontière HTTP.
- Application reçoit un DTO de contenu ou un contrat de flux.
- Pour un gros fichier ou un ZIP, utiliser un flux ou un chemin encapsulé par un adaptateur ; ne pas charger puis recopier l’archive entière en mémoire.
- Vérifier extension, MIME, taille, erreurs de lecture et traversée de chemins ZIP.
- Fermer les ressources dans un bloc `finally`.

## Configuration

- `env()` est autorisé uniquement dans `config/*.php`.
- Le code utilise `config()`.
- `APP_URL` est l’origine canonique ; ne pas multiplier les variables équivalentes.
- Utiliser `APP_SCHEME` uniquement lorsqu’un reverse proxy ou le runtime exige de forcer le schéma.
- Fournir `.env.example` sans secret réel.
- Ne jamais versionner `.env`.

## Docker

### Développement

`docker-compose.yml` doit fournir :

- l’application Laravel ;
- PostgreSQL ;
- les volumes du code, de `vendor` et de `node_modules` ;
- Artisan et Vite accessibles depuis l’hôte ;
- un healthcheck PostgreSQL ;
- des ports configurables sans dépendre de `0.0.0.0` dans les URLs générées par Vite.

### Production

`docker-compose.prod.yml` doit :

- utiliser l’image GHCR immuable ;
- ne pas monter le code source ;
- ne pas lancer Vite ;
- configurer les healthchecks et dépendances ;
- transmettre uniquement la configuration nécessaire ;
- exécuter les migrations via une étape contrôlée.

### Dockerfile multi-stage

Prévoir au minimum :

1. une base PHP avec extensions ;
2. une étape Composer sans scripts avant la copie complète si Laravel n’est pas encore bootable ;
3. une étape Node pour compiler les assets ;
4. une cible `dev` avec outils de développement ;
5. une cible `prod` minimale avec vendor optimisé et assets compilés.

Éviter `composer install` avec les scripts Laravel avant que l’application, ses providers et son fichier Artisan soient copiés.

## GitHub Actions et GHCR

Créer un workflow qui :

1. récupère le dépôt ;
2. active Buildx ;
3. se connecte à `ghcr.io` avec `GITHUB_TOKEN` ;
4. construit la cible `prod` ;
5. publie un tag immuable basé sur le SHA ;
6. publie éventuellement le tag de branche ou `latest` selon la politique du projet ;
7. exploite le cache GitHub Actions ;
8. n’insère aucun secret applicatif dans l’image.

Accorder seulement :

```yaml
permissions:
    contents: read
    packages: write
```

## Migration depuis Next.js ou Supabase

Procéder écran par écran et flux par flux.

1. Inventorier les routes, props, actions, statuts, cookies, sessions, textes et cas limites.
2. Capturer la référence fonctionnelle avant toute réécriture.
3. Migrer le schéma et les données vers PostgreSQL.
4. Reproduire les règles métier dans Domain/Application.
5. Reproduire les pages avec Inertia et React sans changer le design.
6. Comparer les payloads et effets avant/après.
7. Supprimer seulement après parité les dépendances Next et Supabase.

Ne jamais considérer une page qui s’affiche comme une preuve de parité. Vérifier les mutations, états concurrents, exports, intégrations, permissions et données sensibles.

## Contrôles de livraison

Adapter les commandes au projet, puis exécuter au minimum :

```bash
composer validate --strict
composer analyse
vendor/bin/pint --test
php artisan migrate:fresh --force
php artisan route:list
php artisan schedule:list
npm run typecheck
npm run lint
npm run build
docker compose config
docker compose -f docker-compose.prod.yml config
docker build --target prod .
```

Avant livraison :

- aucune erreur Larastan au niveau maximal configuré ;
- aucune dépendance HTTP dans Application ;
- aucun accès direct à une API externe depuis un contrôleur ;
- aucune ancienne référence Next/Supabase active ;
- aucune donnée sensible dans les logs, audits ou props ;
- aucune régression de route, rôle, cookie, statut HTTP, redirection, texte ou design ;
- arbre Git propre et commit limité au périmètre.

## Anti-patterns

Refuser :

- contrôleur de plusieurs centaines de lignes ;
- service omniscient mélangeant SQL, métier, HTTP et présentation ;
- `app()` ou `resolve()` comme service locator ;
- tableau métier non typé traversant plusieurs couches ;
- `Request` dans Application ;
- accès `env()` dans un service ;
- styled-components en une ligne ;
- HTML brut ou styles inline dans les composants ;
- chargement intégral d’un ZIP volumineux en mémoire ;
- appel réseau sous verrou DB sans justification ;
- remplacement de stack qui simplifie ou supprime silencieusement une fonctionnalité.
