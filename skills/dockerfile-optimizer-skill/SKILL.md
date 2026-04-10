---
name: dockerfile-optimizer
description: Analyze and optimize Dockerfiles for security, image size, build performance, and best practices. Use when asked to "audit Dockerfile", "optimize Docker", "review Dockerfile", "reduce image size", or "check Docker best practices".
argument-hint: <dockerfile-path-or-glob>
metadata:
  author: Abdel-Mounhim EL ISSAOUI
  version: '1.0.0'
---

# Dockerfile Optimizer

Analyse et optimise les Dockerfiles selon les bonnes pratiques Docker, la securite, la taille d'image et la performance de build.

## When to Apply

- Audit ou review d'un Dockerfile
- Optimisation de la taille d'image Docker
- Amelioration du cache des layers
- Verification de la securite d'une image
- Revue avant mise en production

## How It Works

1. Localiser les Dockerfiles (argument fourni ou `**/Dockerfile*` hors `node_modules`)
2. Lire chaque Dockerfile et les fichiers associes (`.dockerignore`, `docker-compose*.yml`)
3. Appliquer les regles d'audit par categorie
4. Generer un rapport avec score, findings et recommandations
5. Sauvegarder le rapport dans `/reports/dockerfile-audit-{timestamp}.md`

## Audit Rules

### 1. Securite (CRITICAL)

| ID     | Regle                 | Severite | Detection                                                                                                 |
| ------ | --------------------- | -------- | --------------------------------------------------------------------------------------------------------- |
| SEC-01 | `no-root-user`        | critical | Absence d'instruction `USER` non-root                                                                     |
| SEC-02 | `no-latest-tag`       | high     | Image de base avec tag `latest` ou sans tag                                                               |
| SEC-03 | `secrets-in-env`      | critical | Secrets/passwords dans `ENV` ou `ARG` (patterns: `PASSWORD`, `SECRET`, `TOKEN`, `API_KEY`, `PRIVATE_KEY`) |
| SEC-04 | `no-add-remote`       | high     | `ADD` avec URL distante (preferer `COPY` + `curl`/`wget` avec verification)                               |
| SEC-05 | `no-sudo`             | medium   | Utilisation de `sudo` dans `RUN`                                                                          |
| SEC-06 | `pin-versions`        | high     | Packages systeme installes sans version pincee (`apt-get install package` sans `=version`)                |
| SEC-07 | `no-setuid-setgid`    | medium   | Absence de suppression des binaires setuid/setgid                                                         |
| SEC-08 | `trusted-base-image`  | medium   | Image de base non officielle (pas de prefixe Docker Hub officiel)                                         |
| SEC-09 | `no-secrets-copy`     | critical | `COPY` de fichiers sensibles (`.env`, `*.pem`, `*.key`, `id_rsa`, `credentials*`)                         |
| SEC-10 | `healthcheck-present` | low      | Absence d'instruction `HEALTHCHECK`                                                                       |

### 2. Taille d'image (HIGH)

| ID      | Regle                   | Severite | Detection                                                                     |
| ------- | ----------------------- | -------- | ----------------------------------------------------------------------------- |
| SIZE-01 | `use-alpine`            | medium   | Image de base non-alpine quand une variante alpine existe                     |
| SIZE-02 | `multi-stage-build`     | high     | Absence de multi-stage build pour les applications avec etape de build        |
| SIZE-03 | `clean-package-cache`   | high     | `apt-get install` sans `rm -rf /var/lib/apt/lists/*` dans le meme `RUN`       |
| SIZE-04 | `no-dev-dependencies`   | medium   | `npm install` sans `--production` ou `npm ci --omit=dev` en production        |
| SIZE-05 | `minimize-layers`       | low      | Commandes `RUN` consecutives qui pourraient etre combinees                    |
| SIZE-06 | `remove-build-tools`    | medium   | Outils de build installes et non supprimes (gcc, make, g++, python)           |
| SIZE-07 | `dockerignore-exists`   | high     | Absence de `.dockerignore` dans le meme repertoire                            |
| SIZE-08 | `dockerignore-coverage` | medium   | `.dockerignore` ne couvre pas `node_modules`, `.git`, `*.log`, `dist`, `.env` |
| SIZE-09 | `no-unnecessary-files`  | low      | `COPY . .` sans `.dockerignore` adequat                                       |

### 3. Performance de build / Cache (HIGH)

| ID       | Regle                | Severite | Detection                                                                      |
| -------- | -------------------- | -------- | ------------------------------------------------------------------------------ |
| CACHE-01 | `copy-package-first` | high     | `COPY . .` avant `npm install` (invalide le cache a chaque changement de code) |
| CACHE-02 | `order-by-frequency` | medium   | Instructions qui changent souvent placees avant celles qui changent rarement   |
| CACHE-03 | `use-npm-ci`         | medium   | `npm install` au lieu de `npm ci` (non deterministe)                           |
| CACHE-04 | `lock-file-copied`   | high     | `package.json` copie sans `package-lock.json`                                  |
| CACHE-05 | `no-cache-bust`      | low      | `apt-get update` et `apt-get install` dans des `RUN` separes                   |

### 4. Bonnes pratiques (MEDIUM)

| ID    | Regle               | Severite | Detection                                                                           |
| ----- | ------------------- | -------- | ----------------------------------------------------------------------------------- |
| BP-01 | `use-workdir`       | medium   | `RUN cd /path` au lieu de `WORKDIR /path`                                           |
| BP-02 | `explicit-expose`   | low      | Port utilise sans instruction `EXPOSE` correspondante                               |
| BP-03 | `label-metadata`    | info     | Absence de `LABEL` (maintainer, version, description)                               |
| BP-04 | `no-shell-form-cmd` | medium   | `CMD` en shell form au lieu d'exec form (`CMD npm start` vs `CMD ["npm", "start"]`) |
| BP-05 | `single-cmd`        | low      | Plusieurs instructions `CMD` (seule la derniere est effective)                      |
| BP-06 | `arg-before-from`   | info     | `ARG` apres `FROM` qui devrait etre avant (pour parametrer l'image de base)         |
| BP-07 | `copy-over-add`     | medium   | `ADD` utilise pour une simple copie locale (preferer `COPY`)                        |
| BP-08 | `no-deprecated`     | medium   | Instructions deprecees (`MAINTAINER` → utiliser `LABEL maintainer=`)                |

## Scoring

Chaque Dockerfile demarre avec un score de 100 points :

| Severite | Penalite par finding |
| -------- | -------------------- |
| critical | -20                  |
| high     | -10                  |
| medium   | -5                   |
| low      | -2                   |
| info     | -1                   |

**Grades :**

- **A** (90-100) : Excellent, pret pour la production
- **B** (75-89) : Bon, ameliorations mineures recommandees
- **C** (60-74) : Acceptable, ameliorations necessaires
- **D** (40-59) : Insuffisant, optimisations importantes requises
- **F** (0-39) : Critique, refactoring necessaire

Score minimum : 0.

## Output Format

```markdown
# Dockerfile Audit Report

**Generated:** [Date]
**Files analyzed:** [count]

## Summary

| Dockerfile           | Score | Grade | Critical | High | Medium | Low | Info |
| -------------------- | ----- | ----- | -------- | ---- | ------ | --- | ---- |
| back-end/Dockerfile  | 65    | C     | 1        | 2    | 1      | 0   | 1    |
| front-end/Dockerfile | 80    | B     | 0        | 1    | 2      | 1   | 0    |

---

## Detailed Analysis: [Dockerfile Path]

**Base image:** node:20.20.0-alpine
**Total instructions:** 12
**Estimated layers:** 8
**Score:** 65/100 (C)

### Findings

#### CRITICAL

| ID     | Rule           | Line | Description                              | Fix                                    |
| ------ | -------------- | ---- | ---------------------------------------- | -------------------------------------- |
| SEC-03 | secrets-in-env | 14   | Password exposed via ENV DB_DEV_PASSWORD | Use Docker secrets or runtime env vars |

#### HIGH

| ID  | Rule | Line | Description | Fix |
| --- | ---- | ---- | ----------- | --- |
| ... | ...  | ...  | ...         | ... |

#### MEDIUM

...

#### LOW / INFO

...

### Optimized Dockerfile

[Provide the complete rewritten Dockerfile with all fixes applied and comments explaining each optimization]

---

## Global Recommendations

### Immediate Actions (Critical/High)

1. [Specific action]
2. ...

### Short-term Improvements (Medium)

1. ...

### Nice to Have (Low/Info)

1. ...

---

## .dockerignore Recommendations

[If missing or incomplete, provide a recommended .dockerignore content]
```

## Optimized Dockerfile Generation

Pour chaque Dockerfile analyse, generer une version optimisee qui :

1. **Applique le multi-stage build** quand pertinent (etape build + etape runtime)
2. **Reordonne les layers** pour maximiser le cache (fichiers qui changent rarement en premier)
3. **Ajoute un utilisateur non-root** avec les permissions necessaires
4. **Pine toutes les versions** (image de base, packages systeme, packages npm)
5. **Supprime les fichiers inutiles** apres installation
6. **Utilise exec form** pour CMD et ENTRYPOINT
7. **Ajoute HEALTHCHECK** si un serveur HTTP est expose
8. **Commente chaque optimisation** avec une reference a la regle

### Exemple de multi-stage build Node.js optimise

```dockerfile
# Stage 1: Dependencies
FROM node:20.20.0-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Stage 2: Build (si necessaire)
FROM node:20.20.0-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20.20.0-alpine AS runner
WORKDIR /app

# SEC-01: Non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# SIZE-01: Only production files
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

# SEC-10: Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3030/api/health/live || exit 1

USER appuser
EXPOSE 3030

# BP-04: Exec form
CMD ["node", "dist/index.js"]
```

## Usage

When invoked:

1. If a file/path argument is provided, analyze that specific Dockerfile
2. If no argument, find all Dockerfiles in the project (excluding `node_modules`)
3. Also check for associated `.dockerignore` files
4. Apply all rules from all categories
5. Generate the report in the format above
6. Save to `/reports/dockerfile-audit-{YYYY-MM-DD}.md`
7. Propose the optimized Dockerfiles with inline comments

## Associated Files to Check

For each Dockerfile found, also verify:

- `.dockerignore` in the same directory
- `docker-compose*.yml` for related service configuration
- `package.json` for Node.js projects (to check scripts, dependencies)
- `.gitlab-ci.yml` for CI/CD Docker build configuration
