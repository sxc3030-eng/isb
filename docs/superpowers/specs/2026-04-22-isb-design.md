# ISB — Idea→System Builder
## Design Spec (v2)

| | |
|---|---|
| **Date** | 2026-04-22 |
| **Status** | Approved (brainstorm v2) — pending implementation plan |
| **Author** | Simon Cantin + Claude Opus 4.7 |
| **Repo** | `D:\ComfyUI-Intel\isb\` (sibling de `vinom\`, Git séparé, **privé**) |
| **Codename** | ISB |
| **Phase** | 1 (dogfood) |

---

## 1. Vision & Purpose

ISB est un **outil web single-page** où n'importe qui décrit son idée de programme via un questionnaire adaptatif et reçoit, contre paiement ($49.99), une **architecture logicielle complète validée par Claude Opus 4.7** (Mermaid diagram + brief PDF + JSON exportable).

**UX cible** : style v0.dev / Lovable / Notion AI — un seul écran, transitions fluides, aucune navigation entre pages.

**Lien avec FORGE** : ISB est indépendant techniquement, mais sert de **machine d'acquisition** pour le workflow FORGE. Le brief généré par ISB est ensuite exploitable par Simon dans FORGE pour livrer du code custom (upsell séparé, manuel, hors scope ISB).

**Dual usage** :
- Externe : SaaS pour clients tiers (Phase 2)
- Interne : Simon dogfood l'outil pour structurer ses propres idées avant de coder (Phase 1)

## 2. Users & Business Model

### Cible
- Freelancers dev qui veulent un cahier des charges structuré
- Startups early-stage / entrepreneurs non-tech
- Builders IA, agences
- Simon lui-même (premier utilisateur)

### Modèle
- **Preview gratuit** : 3 modules visibles, reste teasé/flouté
- **$49.99 (one-shot)** : architecture complète + brief PDF + JSON export
- **Pas de code livré** dans ISB. Code = upsell manuel via FORGE workflow (hors spec)
- **Marge brute par génération payée** : ~96 % (tokens ~$1.90, prix $49.99)
- **Coût marketing par visiteur preview** : ~$0.10 (acceptable)

## 3. Architecture globale

### Stack Phase 1
- **Next.js 15** (App Router, TypeScript)
- **Tailwind CSS** — UI rapide
- **@anthropic-ai/sdk** — appel direct Claude
- **mermaid + Playwright** — diagramme rendu server-side
- **@react-pdf/renderer** — PDF brief (PDF taggué accessible)
- **Zod** — validation schema partout
- **Zustand + localStorage** — state machine + persistence client
- **Filesystem** — stockage `./generated/<uuid>/`

### Pas de DB, pas d'auth, pas de Stripe en Phase 1
Tout bypassé via stubs (voir Section 6).

### Routes
```
/                Outil unique (state machine côté client)
/r/<uuid>        Permalien partageable vers brief payé (bookmarkable)
```

### State machine (côté React, persistée localStorage)
```
IDLE
  ↓ user enters project name
NAMING
  ↓ click "Commencer"
ASKING                    ← itère sur questions (8 base + branches)
  ↓ last question answered
PREVIEWING                ← Sonnet "preview pass" → 3 modules visibles
  ↓ user clicks "Débloquer"
PAYWALL                   ← Phase 1 : bouton "Dogfood unlock"
                            Phase 2 : Stripe checkout inline
  ↓ payment confirmed
GENERATING                ← Pass 1 (Sonnet) + Pass 2 (Opus) streaming
  ↓ pipeline complete
COMPLETE                  ← architecture full + downloads
```

### Layout disque
```
isb/
├── app/
│   ├── page.tsx               # L'outil (state machine)
│   ├── r/[uuid]/page.tsx      # Permalien brief payé
│   ├── api/
│   │   ├── preview/route.ts   # Sonnet quick pass
│   │   ├── generate/route.ts  # Pass 1 + Pass 2 streaming
│   │   └── stripe/webhook/    # Phase 2 stub
│   └── layout.tsx
├── components/
│   ├── tool/                  # Composants par état (Naming, Asking, Preview, Paywall, Complete)
│   ├── questionnaire/
│   ├── diagram/               # SVG Mermaid renderer client
│   └── ui/                    # buttons, inputs, etc.
├── lib/
│   ├── claude.ts              # SDK wrapper (mockable)
│   ├── payment.ts             # stub Phase 1
│   ├── auth.ts                # stub Phase 1
│   ├── storage.ts             # FilesystemStorage Phase 1
│   ├── email.ts               # console stub Phase 1
│   ├── mermaid.ts             # Playwright render
│   └── pdf.tsx                # @react-pdf composants
├── questions.config.ts        # schéma déclaratif du questionnaire
├── prompts/
│   ├── preview.md             # Pass preview (Sonnet light)
│   ├── pass1.md               # Architecte junior (Sonnet)
│   └── pass2.md               # Reviewer senior (Opus)
├── generated/                 # outputs (gitignored)
├── __fixtures__/              # mocks Claude pour tests
├── __tests__/                 # Vitest
├── e2e/                       # Playwright
└── docs/
    ├── superpowers/specs/
    └── db-schema.sql          # Phase 2 prêt
```

## 4. Questionnaire adaptatif

### Q0 — Nom du projet (entrée)
- Champ texte unique sur écran d'accueil
- Required, 2-60 chars, validation Zod
- Persiste dans state, affiché en header tout au long du parcours

### 8 questions de base obligatoires
1. Idée en une phrase (texte libre)
2. Problème concret résolu (texte libre)
3. Utilisateur principal (texte libre)
4. Type de système (radio : Web / Mobile / API / Automation / CLI / Autre)
5. 3 actions principales (3 champs courts)
6. Format des inputs (multi-select : texte / fichier / API / formulaire / audio / autre)
7. Format des outputs (multi-select : dashboard / fichier / email / API / message / autre)
8. Échelle visée (radio : perso / MVP startup / SaaS scalable / enterprise)

### 5 branches conditionnelles

| Branche | Condition d'unlock | Questions |
|---|---|---|
| **UI/UX** | type ∈ {Web, Mobile} | Pages clés ; auth requise ; mobile-first |
| **Intégrations** | inputs contient "API" OU action mentionne "intégrer/sync" | Services (Gmail, Stripe, Notion, Slack…) ; API critiques |
| **Triggers** | type = Automation OU action mentionne "auto/cron" | Type (webhook / schedule / event) ; continu vs événementiel |
| **Scale** | échelle ∈ {SaaS scalable, enterprise} | Volume estimé ; multi-tenant ; SLA |
| **AI/IA** | action mentionne "AI/IA/learn/generate/predict" | Type (LLM / embeddings / vision) ; modèle préféré |

### Implémentation
- `questions.config.ts` exporte questions + lambdas `shouldUnlock(branch, answers)` (pures)
- `<Questionnaire />` itère sur le schéma déclaratif (un seul composant)
- Validation Zod par question, erreurs inline
- Progress bar `X / ~Y` qui se recalcule à chaque réponse
- Save & resume via `localStorage`
- Animations `framer-motion` entre questions (respect `prefers-reduced-motion`)

**Total** : 8 questions (simple) à ~22 questions (projet complexe). Temps : 3-12 min.

## 5. Pipeline IA (3-stages : Preview → Pass 1 → Pass 2)

### Stage 0 — Preview Pass (Sonnet 4.6, gratuit)
- **Trigger** : dernière question répondue
- **Modèle** : `claude-sonnet-4-6`, temperature 0.3
- **System prompt** caché : "Génère une architecture light : ≤ 3 modules nommés, 1 ligne de description chacun. Pas de tech stack détaillé, pas de MVP plan. Format JSON court."
- **Output** : 3 modules teasés, montrés à l'écran avec teaser "+5 autres modules", "+ tech stack", "+ MVP plan", "+ analyse risques" floutés
- **Coût/temps** : ~$0.05-0.15, 3-6s
- **Justification coût** : marketing accepté pour conversion

### Stage 1 — Architecte Junior (Sonnet 4.6) — APRÈS PAIEMENT
- **Input** : `questionnaire.json` complet
- **Modèle** : `claude-sonnet-4-6` ($3 / $15 par M tokens)
- **System prompt** caché (`cache_control: ephemeral`) : "Tu es un architecte logiciel senior. À partir de ce JSON, produis l'architecture en JSON structuré : modules, tech_stack, folder_structure, data_flow (Mermaid), mvp_plan (3 étapes), risks_assumptions."
- **Output** : JSON validé Zod
- **Coût/temps** : ~$0.30, 5-10s

### Stage 2 — Reviewer Senior (Opus 4.7)
- **Input** : `questionnaire.json` + `pass1.json`
- **Modèle** : `claude-opus-4-7` ($15 / $75 par M tokens)
- **System prompt** caché : "Tu reviens sur le travail d'un architecte junior. Critique, raffine, identifie ce qui manque. Produis l'architecture FINALE avec validation_notes, hidden_constraints, recommended_alternative."
- **Output** : JSON final, validé Zod
- **Coût/temps** : ~$1.50, 15-30s

### Économie complète
| Scénario | Tokens | Coût | Revenue |
|---|---|---|---|
| Preview only (visiteur sort) | $0.10 | $0.10 | $0 (marketing) |
| Preview + paiement → full pipeline | $0.10 + $1.80 | **$1.90** | **$49.99** |
| **Marge brute payés** | | | **~96 %** |

### Streaming UX
Pendant `GENERATING` état, les 2 passes streament via Server Actions :
```
[●○○]  "Drafting full architecture..."   (Pass 1)
[●●○]  "Senior review for completeness..." (Pass 2)
[●●●]  "Architecture ready"              (render)
```

### Error handling
- Preview fail → message "Génération preview indisponible, paye direct pour le brief complet" (revenue salvage)
- Pass 1 fail → retry 1× avec prompt simplifié
- Pass 2 fail → fallback Pass 1 + bannière "Senior review unavailable, partial refund offered" + flag manuel
- Tout fail après paiement → email auto Simon + remboursement Stripe auto Phase 2

### Fichiers générés par UUID (après paiement)
```
/generated/<uuid>/
├── questionnaire.json
├── preview.raw.json
├── pass1.raw.json
├── pass2.raw.json
├── final.json
├── architecture.svg
├── architecture.png
└── brief.pdf
```

## 6. Hooks Phase 2 (préparation sans implémentation)

But : **zéro réécriture** au passage commercial.

### Variable gate
```env
PHASE=1   # dogfood : tout bypassé
PHASE=2   # commercial : Stripe + auth + email actifs
```

### Modules à interface stable

```typescript
// lib/payment.ts
export async function verifyPayment(uuid: string): Promise<boolean> {
  if (process.env.PHASE === '1') return true;
  // Phase 2 : Stripe checkout.session.payment_status === 'paid'
}

// lib/auth.ts
export async function getCurrentUser(): Promise<User | null> {
  if (process.env.PHASE === '1') return DOGFOOD_USER;
  // Phase 2 : NextAuth magic link via Resend (optionnel — paywall fonctionne sans compte)
}

// lib/storage.ts — interface
interface Storage {
  saveSubmission(uuid, data): Promise<void>;
  saveBrief(uuid, files): Promise<void>;
  getBrief(uuid): Promise<Brief | null>;
}
// Phase 1 : FilesystemStorage  → ./generated/<uuid>/
// Phase 2 : PostgresStorage + S3FileStorage  (même interface)

// lib/email.ts
export async function sendBriefEmail(to, uuid, pdfPath): Promise<void> {
  if (process.env.PHASE === '1') { console.log('[email stub]', to); return; }
  // Phase 2 : Resend API
}
```

### Routes ajoutées Phase 2
- `/api/stripe/webhook` (active body)
- `/admin` (dashboard Simon : liste briefs payés, briefs échoués, stats)
- (`/auth/sign-in` optionnel — le paywall marche sans compte, juste email)

### Schéma DB documenté dès Phase 1 (`docs/db-schema.sql`)
```sql
submissions(id uuid pk, project_name text, questionnaire jsonb, status text, created_at timestamptz)
payments(id uuid pk, submission_id uuid fk, stripe_session_id text, amount_cents int, status text, created_at)
briefs(id uuid pk, submission_id uuid fk, pdf_url text, svg_url text, json_url text, generated_at)
emails(id uuid pk, submission_id uuid fk, to_email text, sent_at timestamptz)
```
Pas de table `users` Phase 2 initiale (paywall sans compte = friction min). Optionnel plus tard.

### `.env.example` complet dès Phase 1
```env
ANTHROPIC_API_KEY=
PHASE=1
NEXT_PUBLIC_BASE_URL=http://localhost:3000

# Phase 2 (laissés vides, documentés)
# DATABASE_URL=
# STRIPE_SECRET_KEY=
# STRIPE_WEBHOOK_SECRET=
# RESEND_API_KEY=
# AUTH_SECRET=
# S3_BUCKET=
# PRO_MODE_KEY=          # 32-char secret pour activation Pro via cookie
```

### Estimation Phase 1 → Phase 2
- Implémenter les bodies des 4 stubs
- Ajouter Prisma + migrations
- Ajouter `/api/stripe/webhook` body, route `/admin`
- Enrichir landing avec copy commercial
- Connecter Stripe checkout inline + webhook
- **Aucun composant existant à toucher.** Estimation : 3-5 jours.

## 7. Rendu Mermaid + PDF brief

### Mermaid → SVG/PNG (Playwright headless)
- Lance Chromium en pool warm (1-2 instances)
- Page HTML avec `<div class="mermaid">{diagram}</div>` + script Mermaid
- Récupère SVG via `page.locator('svg').innerHTML()`
- Screenshot PNG via `locator('svg').screenshot()`

**Phase 2 sur Fly.io** : Playwright marche dans containers, zéro changement.

### PDF brief — `@react-pdf/renderer`
Composants React déclaratifs → PDF **taggué accessible**.

**Structure (5 pages) :**
| Page | Contenu |
|---|---|
| 1. Cover | Nom du projet, idée 1-phrase, date, "Generated & validated by Claude Opus 4.7" |
| 2. Problem & User | Problème, utilisateur, contexte, urgence |
| 3. Architecture | PNG Mermaid embed (avec alt-text généré par Claude) + liste modules |
| 4. Tech Stack | Frontend / Backend / DB / Infra / AI — chacun justifié 2-3 lignes |
| 5. MVP Plan + Risques | 3 étapes MVP + `hidden_constraints` + `recommended_alternative` (valeur Pass 2) |

**Style** : Inter / IBM Plex Mono, palette neutre, grille 8px, contraste WCAG AA.

### Vue COMPLETE dans l'outil
État final de la state machine (toujours sur `/`) :
```
[Header : nom du projet · "Brief complet" · Export menu]
[SVG zoomable de l'architecture (avec <title>/<desc>)]
[Tabs ARIA : Modules | Tech Stack | MVP Plan | Risks]
[Bouton : Download PDF · Export JSON · Copy permalink]
[Footer Phase 2 : "Want me to build this for you? → upsell"]
```

### Permalien `/r/<uuid>`
Lien permanent vers le même contenu en lecture seule (bookmark, partage). Utilise les fichiers `/generated/<uuid>/`.

### Livrables au client
1. PDF brief (hero deliverable, taggué accessible)
2. JSON export (pour devs)
3. PNG haute résolution architecture

## 8. Stratégie de tests

### Stack
- **Vitest** — unit + integration
- **Playwright** — E2E + Mermaid render + axe-core a11y
- **pdf-parse** — assertions PDF
- **MSW** — mock SDK Anthropic

### Pyramide

#### Unit (~50 tests)
- Logique branches (`shouldUnlock(branch, answers)` — pure)
- State machine transitions (IDLE → NAMING → ASKING → ...)
- Zod schemas (questionnaire / preview / pass1 / pass2 / final)
- Utilitaires (UUID, paths, cache keys)
- Stubs Phase 1 (`verifyPayment` true, `getCurrentUser` DOGFOOD_USER)

#### Integration (~17 tests)
- Pipeline Claude 3-stages avec SDK mocké (fixtures `__fixtures__/claude/*.json`)
- Storage roundtrip (FilesystemStorage + tmpdir)
- Mermaid render réel (Playwright headless)
- PDF generation : génère + parse via `pdf-parse` + assert sections clés
- PDF taggué : assert présence des tags d'accessibilité
- localStorage persist + resume

#### E2E (~5 tests)
1. Happy path : nom → 8 questions → preview → unlock → result → PDF download (tout sur `/`)
2. Branche adaptive : type=Web + inputs=API → assert UI + Intégrations branches visibles
3. Refresh mid-questionnaire → reprend à la bonne question (localStorage)
4. Permalien `/r/<uuid>` : ouvre brief payé en mode lecture seule
5. Error fallback : Pass 2 mock down → bannière "Senior review unavailable"

#### a11y tests (axe-core dans Playwright, ~3 tests)
- État NAMING : zéro violation axe
- État ASKING (mid-questionnaire) : zéro violation axe
- État COMPLETE : zéro violation axe

#### Live API smoke (manuel + nightly)
- E2E gated `RUN_LIVE_API=1` : vraie API, génère brief, vérifie coût < $3, valide Zod final
- Lancer avant chaque deploy Phase 2

### TDD
- Critique (branches, state machine, Zod, contrat pipeline) : test failing avant implémentation
- Rendu (Mermaid, PDF) : implémentation puis test (visual feedback nécessaire)

### Test fixtures
- `__fixtures__/questionnaires/` : 5 cas types (web simple / automation / SaaS / CLI / mobile)
- `__fixtures__/claude/` : preview + pass1 + pass2 enregistrés pour ces 5 cas
- `npm run regen-fixtures` (manuel, consomme tokens)

### Manual QA dogfood (gate pour déclarer Phase 1 done)
Simon génère **5 briefs** sur ses propres idées et valide qualité. Si moins de 4/5 sont "utilisables sans retouche" → itérer prompts.

## 9. Acceptance criteria (Phase 1)

**Phase 1 est complete quand :**

1. ✅ `npm run dev` démarre l'app sur `localhost:3000`
2. ✅ State machine fonctionne : NAMING → ASKING → PREVIEWING → PAYWALL → GENERATING → COMPLETE
3. ✅ Questionnaire 8 base + branches fonctionne, save/resume via localStorage
4. ✅ Preview pass affiche 3 modules teasés en < 6s
5. ✅ Pipeline 2-pass full streame en live à l'écran après "unlock"
6. ✅ Vue COMPLETE affiche SVG + tabs + downloads
7. ✅ Download PDF fonctionne (5 pages, taggué accessible, contient sections attendues)
8. ✅ Download JSON fonctionne (validé Zod)
9. ✅ Permalien `/r/<uuid>` ouvre brief en lecture seule
10. ✅ 4 stubs Phase 2 en place et testés (`payment`, `auth`, `storage`, `email`)
11. ✅ ≥ 80 tests verts (52 unit + 18 integration + 7 E2E + 3 a11y)
12. ✅ **Lighthouse a11y score ≥ 95** sur `/` (états NAMING, ASKING, COMPLETE)
13. ✅ Manual NVDA test sur happy path : pas d'obstacle bloquant
14. ✅ Manual QA : 4/5 briefs Simon "utilisables sans retouche"
15. ✅ `.env.example` complet avec sections Phase 2 documentées
16. ✅ `docs/db-schema.sql` versionné

## 10. Non-goals (Phase 1)

Pour éviter scope creep :

- ❌ Pas d'auth utilisateur
- ❌ Pas de paiement Stripe (juste bouton dogfood unlock)
- ❌ Pas de DB (Postgres)
- ❌ Pas d'email transactionnel
- ❌ Pas de landing page commerciale (l'outil EST la landing)
- ❌ Pas de dashboard admin
- ❌ Pas d'upsell "Build it for me"
- ❌ Pas de partage social du brief (juste copy permalink)
- ❌ Pas de multi-langue (FR uniquement Phase 1)
- ❌ Pas de webhook entrant/sortant
- ❌ Pas de versionning du brief (1 génération = 1 PDF figé)
- ❌ Pas de mode dark (Phase 2)
- ❌ Pas de déploiement (juste localhost)

Tous ces items appartiennent à Phase 2 ou plus tard, et sont préparés via les hooks de la Section 6.

## 11. Open questions

Aucune. Tous les points ont été tranchés en brainstorm.

## 12. Accessibilité (WCAG 2.1 AA)

Cible non-négociable : **WCAG 2.1 niveau AA**. Standard légal Canada (AODA), USA (ADA), Europe (EAA). Critique pour vente entreprise/gouvernement.

### Questionnaire (surface critique)
- HTML natif (`<input>`, `<radio>`, `<checkbox>`) → a11y baseline gratuite
- `<fieldset>` + `<legend>` pour grouper questions multi-champs (ex: 3 actions)
- Labels explicites associés (`<label for>` ou wrap)
- Erreurs Zod inline + région `aria-live="polite"` (annoncées au lecteur d'écran)
- Progress bar : `role="progressbar"` + `aria-valuenow/min/max/text`
- Déverrouillage de branche annoncé : "3 questions supplémentaires ajoutées"
- Navigation clavier 100 % (Tab/Shift+Tab/Enter/Space/Esc)
- Focus visible (outline conservé)
- `aria-required="true"` sur questions obligatoires

### Vue COMPLETE
- SVG Mermaid : `<title>` + `<desc>` générés par Claude (description plain English du diagramme)
- Tabs : pattern ARIA tabs complet (`role="tablist"`, `aria-selected`, navigation flèches)
- Boutons Download avec texte descriptif ("Télécharger le brief PDF, 5 pages")

### PDF brief
- `@react-pdf/renderer` PDF taggué activé
- Alt-text sur PNG Mermaid embed
- Hiérarchie de titres (H1 → H2 → H3)
- `lang="fr"` sur le document
- Reading order logique

### Général
- `lang="fr"` sur `<html>`
- Skip-link "Aller au contenu" (premier élément focusable)
- `prefers-reduced-motion` respecté (anims framer-motion désactivées)
- Contraste 4.5:1 texte, 3:1 UI (palette validée Tailwind)
- Pas de couleur seule pour info (toujours doublé d'icône ou texte)
- Pas de capture-piège focus

### Tests automatisés
- **axe-core** dans Playwright → 3 tests (NAMING / ASKING / COMPLETE) zéro violation
- **Lighthouse a11y ≥ 95** dans CI (script `npm run lighthouse-ci`)
- **Manual NVDA** (gratuit Windows) sur happy path avant chaque release

## 13. Hosting & Deployment

### Phase 1 (dogfood)
**Local uniquement.** `npm run dev` sur la machine Windows de Simon. Aucune infra cloud.

### Phase 2 (commercial)

**Choix : Fly.io + Cloudflare DNS.**

#### Pourquoi Fly.io
- ✅ **Datacenter Montreal (`yul`)** — données restent au Québec, argument vente B2B/gouvernement Canada (cohérent avec posture FORGE → Thales/defense)
- ✅ Containers (Docker) → Playwright SSR marche zéro friction
- ✅ Postgres managé (`fly postgres create`) ~$5/mo
- ✅ ~$5-15/mo total pour trafic anticipé Phase 2 early
- ✅ Multi-région scaling possible si besoin (USA/Europe plus tard)
- ✅ Zero downtime deploys, healthchecks, auto-restart
- ❌ Plus de config initiale qu'un Vercel (1 jour vs 1h)

#### Domaine
- Acheter via **Cloudflare Registrar** (~$15/an, prix coûtant, pas de markup) ou Namecheap
- Suggestions à valider plus tard : `isb.app`, `briefarchitect.com`, `architectai.com`, ou nom de marque inventé
- Cloudflare DNS gratuit + protection DDoS basique gratuite

#### Stack déploiement
```
Fly.io app (Dockerfile)
├── Next.js standalone build (server.js)
├── Playwright + Chromium dans l'image
├── Volume mounted pour ./generated/<uuid>/  (ou S3 plus tard)
└── Healthcheck GET /api/health

Fly Postgres (managed, single instance Phase 2 early)

Cloudflare
├── DNS A record → Fly.io app IP
├── SSL automatique (Cloudflare Universal SSL)
└── Future : Workers pour edge caching si besoin
```

#### CI/CD
- GitHub Actions → `fly deploy` sur push `main`
- Preview deploys sur PRs (`fly deploy --strategy=immediate -a isb-preview-<sha>`)
- Secrets stockés dans Fly secrets (`fly secrets set ANTHROPIC_API_KEY=...`)

#### Estimations coûts mensuels Phase 2 early (≤ 100 visiteurs/jour)
| Poste | Coût |
|---|---|
| Fly.io app (1 GB RAM, shared CPU) | $5 |
| Fly Postgres (1 GB) | $5 |
| Domaine | $1.25 ($15/an) |
| Cloudflare | $0 |
| **Total infra** | **~$11/mo** |
| Tokens Claude (100 visiteurs × $0.10 preview + 5 paiements × $1.80) | ~$310 |
| Stripe fees (5 × $49.99 × 2.9 %+30¢) | ~$8.75 |
| **Total dépenses** | **~$330/mo** |
| Revenue (5 × $49.99) | $249.75 |
| **Net Phase 2 early** | **-$80/mo** (acceptable, scale jusqu'à break-even ~7 paiements/mois) |

À 10+ paiements/mois → profitable.

## 15. Pro Mode (usage interne Simon + tier premium Phase 2+)

### Objectif
Débloquer toutes les limites du free tier pour Simon (qui dogfood l'outil) et, plus tard, vendre comme tier supérieur aux clients sérieux.

### Activation

```typescript
// lib/pro-mode.ts
export function isProMode(): boolean {
  if (process.env.PHASE === '1') return true;          // dogfood = toujours pro
  // Phase 2 : cookie signé OU header avec PRO_MODE_KEY
  const cookieValue = cookies().get('pro_mode')?.value;
  return cookieValue === process.env.PRO_MODE_KEY;
}
```

**Phase 1** : auto-activé (Simon est seul utilisateur).
**Phase 2** : URL d'activation `/?pro=<KEY>` set le cookie, persistant 365 jours. Clé secrète dans `.env`.

### Différences feature par feature

| Feature | Standard | Pro Mode |
|---|---|---|
| Preview teaser | 3 modules visibles, reste flouté | **Tous modules visibles, pas de teaser** |
| Architecture profondeur | 5-8 modules | **10-15 modules + sous-modules détaillés** |
| Tech stack | Stack reco (1 par couche) | **Stack reco + 2 alternatives évaluées par couche** |
| MVP plan | 3 étapes simples | **Roadmap 3 phases × 3 étapes (MVP / Beta / GA)** |
| Risques | 3-5 risques principaux | **Tous risques + mitigations détaillées + niveau d'effort** |
| Section bonus Pro | — | **"Prompt for Claude" : prompt copy-paste prêt à donner à Claude Code/Cursor pour démarrer le build immédiatement** |
| Section bonus Pro | — | **"Cost & timeline estimate" : estimation tokens/temps pour build full** |
| Coût génération | ~$1.90 | **~$3-5** (Opus avec prompt étendu, plus de tokens output) |

### Pricing Phase 2 (proposé)
| Tier | Prix | Cible |
|---|---|---|
| Free preview | $0 | Lead capture |
| Standard | $49.99 | Mass market (entrepreneur, freelance) |
| **Pro** | **$199** | Sérieux (CTO, agence, projet enterprise) |
| Pro Annual Pass | $999/an | Power users (10+ briefs/an), inclut updates futurs |

### Implémentation
- Composant Pro Mode badge visible top-right quand actif
- Prompts dédiés `prompts/pass2-pro.md` (deeper instructions)
- Conditional rendering : preview teaser absent en Pro, sections bonus visibles en Pro
- Same Zod schema mais avec champs `pro_only` optionnels (`prompt_for_claude`, `cost_estimate`, `roadmap_phases[]`)

### `.env` ajouté
```env
PRO_MODE_KEY=<random-32-char-secret>   # Phase 2 : clé d'activation cookie
```

### Tests ajoutés (~5)
- Unit : `isProMode()` retourne true en Phase 1, dépend du cookie en Phase 2
- Unit : Zod schema accepte champs Pro optionnels
- Integration : pipeline Pro génère sections bonus
- E2E : URL `/?pro=<KEY>` set cookie + débloque vue
- E2E : sans cookie en Phase 2 → preview standard limité

## 14. Decision log

| Décision | Raisonnement |
|---|---|
| Repo séparé de FORGE, privé | User : "indépendant". IP (prompts Claude) à protéger. Vente du service, pas du code. |
| Next.js (vs CLI vs HTML) | Phase 2 commerciale = web. Zéro réécriture. |
| **Single-page UX** (state machine) | User : "comme un outil, une seule page". Style v0.dev/Lovable. Réduit friction navigation. |
| 3-stage pipeline (Preview + Pass 1 + Pass 2) | Preview = lead magnet à $0.10. Pass 1+2 = qualité premium à $1.80. Justifie $49.99. |
| Pas de code livré ($49.99) | Calcul économique : custom build coûte 4-7h × $80-150/h + tokens. Non viable < $499. Architecture seule = auto, marge 96 %. |
| Questionnaire adaptatif (8 base + 5 branches) | Sweet spot conversion vs richesse du brief. |
| Phase 1 dogfood-first | Simon est le 1er user. Évite over-engineering. Validation avant commercialisation. |
| Filesystem storage Phase 1 | Pas de DB nécessaire. Interface `Storage` permet swap propre Phase 2. |
| Playwright Mermaid (pas client) | Marche partout, container-friendly. Compatible Fly.io. |
| **Fly.io Montreal hosting Phase 2** | Compliance Canada/Quebec, cohérent avec posture FORGE. Containers = Playwright OK. |
| **WCAG 2.1 AA non-négociable** | Légal (AODA, ADA, EAA). Critique vente entreprise/gouvernement. |
| Pas d'auth Phase 2 initiale | Paywall sans compte = friction min. Auth optionnel plus tard si historique demandé. |
| **Pro Mode tier** | Sert dogfood Simon (Phase 1) + monétisation premium Phase 2+ ($199/brief, $999/an). Section bonus "Prompt for Claude" = bridge direct vers FORGE workflow. |
