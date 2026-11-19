# Sécurité de TaskFlow

## Contrôles automatiques

Tous les contrôles tournent dans `.github/workflows/security.yml`, à chaque PR, à chaque push sur `main`, une fois par semaine (lundi) et à la demande.

| Contrôle | Outil | Ce qu'il protège | Quand |
|---|---|---|---|
| Secrets | gitleaks | Mots de passe, clés ou jetons oubliés dans le code ou l'historique Git | PR, push sur main, hebdomadaire |
| Dépendances (SCA) | npm audit | Bibliothèques avec une faille connue (CVE) ; bloquant sur la production, rapport complet en artefact | PR, push sur main, hebdomadaire |
| Code (SAST) | Semgrep | Erreurs de code dangereuses (injection SQL, par exemple), sans exécuter l'application | PR, push sur main, hebdomadaire |
| Application (DAST) | OWASP ZAP (baseline) | Failles visibles de l'extérieur sur l'application qui tourne (en-têtes manquants, par exemple) | PR, push sur main, hebdomadaire |

Pourquoi un déclencheur hebdomadaire ? Une dépendance saine aujourd'hui peut avoir une CVE publiée la semaine prochaine, sans que personne ne touche au code.

## En-têtes HTTP ajoutés dans `front/nginx.conf`

| En-tête | Pourquoi |
|---|---|
| Content-Security-Policy | Limite les sources autorisées (scripts, images, etc.) : protège contre l'injection de code (XSS). Le front compilé n'a pas de script ni de style inline, donc pas de `unsafe-inline`. |
| X-Content-Type-Options: nosniff | Empêche le navigateur de deviner le type d'un fichier. |
| Referrer-Policy | Limite les informations envoyées aux autres sites par les liens. |
| Permissions-Policy | Désactive caméra, micro et géolocalisation, dont l'application n'a pas besoin. |
| X-Frame-Options: DENY | Empêche d'afficher le site dans un cadre sur un autre site (clickjacking). |
| server_tokens off, X-Powered-By masqué | Ne pas annoncer les versions des logiciels. Cela n'empêche pas une attaque, mais évite de faciliter le repérage. |

## Registre des exceptions

Aucune alerte n'est ignorée en silence. Chaque exception est notée ici.

| Alerte | Outil | Décision | Pourquoi | Date de revue |
|---|---|---|---|---|
| Dépendances de test (vitest, vite, esbuild, tinypool, glob, minimatch) | npm audit | Corrigée | Présentes seulement en `devDependencies` (non exploitables en production), mais un correctif existait : mise à jour effectuée | 2026-11-19 |
| picomatch (high, ReDoS) via vitest, vite, tinyglobby, fdir | npm audit | Exception datée | Aucun correctif publié (l'audit ne propose qu'un retour arrière vers une ancienne version de vitest). Outil de test uniquement (`devDependencies`) : absent de l'image de production, non exploitable en ligne. Le job bloque sur les dépendances de production (`--omit=dev`) ; le rapport complet reste publié en artefact. Issue ouverte pour suivre la sortie d'un correctif | 2026-12-18 |
| Règle helmet_header_x_powered_by (Semgrep) sur app.js | Semgrep | Faux positif, ignoré (nosemgrep) | La règle se déclenche sur la ligne qui désactive l'en-tête X-Powered-By, c'est-à-dire sur la bonne pratique elle-même | 2026-12-18 |
| Règle helmet_header_x_powered_by (Semgrep) sur app.js | Semgrep | Faux positif, ignoré (nosemgrep) | La règle se déclenche sur la ligne qui désactive l'en-tête X-Powered-By, c'est-à-dire sur la bonne pratique elle-même | 2026-12-18 |
| body-parser (high) | npm audit | Corrigée | Dépendance d'Express, donc présente en production : `npm audit fix` | 2026-11-19 |

## Limites connues

- La baseline ZAP est un scan passif : elle n'attaque pas l'application et ne teste pas l'authentification.
- Semgrep ne trouve que les motifs couverts par ses règles publiques.
- gitleaks ne détecte pas un secret qui ne ressemble à aucun motif connu.
