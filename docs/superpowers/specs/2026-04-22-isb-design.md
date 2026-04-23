# ISB — Idea→System Builder
## Design Spec

| | |
|---|---|
| **Date** | 2026-04-22 |
| **Status** | Approved (brainstorm) — pending implementation plan |
| **Author** | Simon Cantin + Claude Opus 4.7 |
| **Repo** | `D:\ComfyUI-Intel\isb\` (sibling de `vinom\`, Git séparé) |
| **Codename** | ISB |
| **Phase** | 1 (dogfood) |

---

## 1. Vision & Purpose

ISB est un site web où n'importe qui décrit son idée de programme via un questionnaire adaptatif et reçoit, contre paiement, une **architecture logicielle complète validée par Claude Opus 4.7** (Mermaid diagram + brief PDF + JSON exportable).

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
- **Free preview** : questionnaire complet + architecture limitée à 3 fonctions visibles
- **$49.99 (one-shot)** : architecture complète sans limite + brief PDF deep + JSON export
- **Pas de code livré** dans ISB. Code = upsell manuel via FORGE workflow (hors spec).
- **Marge brute par génération** : ~96 % (tokens ~$1.80, prix $49.99)

## 3. Architecture globale

### Stack Phase 1
- **Next.js 15** (App Router, TypeScript)
- **Tailwind CSS** — UI rapide
- **@anthropic-ai/sdk** — appel direct Claude (pas via FORGE)
- **mermaid + Playwright** — diagramme rendu server-side
- **@react-pdf/renderer** — PDF brief
- **Zod** — validation schema partout
- **Zustand + localStorage** — state questionnaire (résiste au refresh)
- **Filesystem** — stockage `./generated/<uuid>/`

### Pas de DB, pas d'auth, pas de Stripe en Phase 1
Tout est bypassé via stubs (voir Section 6).

### Routes Phase 1
```
/                      Landing minimal + CTA
/new                   Questionnaire adaptatif
/preview/<uuid>        Architecture limitée (3 fonctions)
/unlock/<uuid>         Bouton "Dogfood unlock" (Phase 1) → Stripe (Phase 2)
/result/<uuid>         Architecture complète + downloads
```

### Layout disque
```
isb/
├── app/                       # Next.js App Router
├── components/
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
- `questions.config.ts` exporte questions + lambdas `shouldUnlock(branch, answers)`
- `<Questionnaire />` itère sur le schéma déclaratif (un seul composant)
- Validation Zod par question, erreurs inline
- Progress bar `X / ~Y` qui se recalcule à chaque réponse
- Save & resume via `localStorage`

**Total** : 8 questions (simple) à ~22 questions (projet complexe). Temps : 3-12 min.

## 5. Pipeline IA (2-pass Claude validation)

### Pass 1 — Architecte Junior (Sonnet 4.6)
- **Input** : `questionnaire.json`
- **Modèle** : `claude-sonnet-4-6` ($3 / $15 par M tokens)
- **System prompt** caché (`cache_control: ephemeral`) : "Tu es un architecte logiciel senior. À partir de ce JSON, produis l'architecture en JSON structuré : modules, tech_stack, folder_structure, data_flow (Mermaid), mvp_plan (3 étapes), risks_assumptions."
- **Output** : JSON validé Zod
- **Coût/temps** : ~$0.30, 5-10s

### Pass 2 — Reviewer Senior (Opus 4.7)
- **Input** : `questionnaire.json` + `pass1.json`
- **Modèle** : `claude-opus-4-7` ($15 / $75 par M tokens)
- **System prompt** caché : "Tu reviens sur le travail d'un architecte junior. Critique, raffine, identifie ce qui manque. Produis l'architecture FINALE avec validation_notes, hidden_constraints, recommended_alternative."
- **Output** : JSON final, validé Zod
- **Coût/temps** : ~$1.50, 15-30s

### Économie par génération
| Poste | Coût |
|---|---|
| Pass 1 (Sonnet, cached) | ~$0.30 |
| Pass 2 (Opus, cached) | ~$1.50 |
| Mermaid render + PDF | $0 |
| **Total** | **~$1.80** |
| **Revenue** | **$49.99** |
| **Marge brute** | **~96 %** |

### Streaming UX
Les 2 passes streament via Server Actions :
```
[●○○]  "Drafting architecture..."        (Pass 1)
[●●○]  "Reviewing for completeness..."   (Pass 2)
[●●●]  "Architecture ready"              (render)
```

### Error handling
- Pass 1 fail → retry 1× avec prompt simplifié
- Pass 2 fail → fallback Pass 1 + bannière "Senior review unavailable"
- Les 2 fail → email auto + flag manuel

### Fichiers générés par UUID
```
/generated/<uuid>/
├── questionnaire.json
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
  // Phase 2 : NextAuth magic link via Resend
}

// lib/storage.ts — interface
interface Storage {
  saveQuestionnaire(uuid, data): Promise<void>;
  saveBrief(uuid, files): Promise<void>;
  getResult(uuid): Promise<Result | null>;
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
- `/api/stripe/webhook`
- `/admin` (dashboard Simon)
- `/dashboard` (historique user)
- `/auth/sign-in` (magic link)

### Schéma DB documenté dès Phase 1 (`docs/db-schema.sql`)
```sql
users(id uuid pk, email text unique, created_at timestamptz)
submissions(id uuid pk, user_id uuid fk, questionnaire jsonb, status text, created_at)
payments(id uuid pk, submission_id uuid fk, stripe_session_id text, amount_cents int, status text)
briefs(id uuid pk, submission_id uuid fk, pdf_url text, svg_url text, json_url text)
```

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
```

### Estimation Phase 1 → Phase 2
- Implémenter les bodies des 4 stubs
- Ajouter Prisma + migrations
- Ajouter routes `/admin`, `/dashboard`, `/auth/*`, webhook Stripe
- Enrichir `/` avec landing copy
- Connecter Stripe checkout + webhook
- **Aucune route ni composant existant à toucher.** Estimation : 3-5 jours.

## 7. Rendu Mermaid + PDF brief

### Mermaid → SVG/PNG (Playwright headless)
- Lance Chromium en pool warm
- Page HTML avec `<div class="mermaid">` + script Mermaid
- Récupère SVG via `page.locator('svg').innerHTML()`
- Screenshot PNG via `locator('svg').screenshot()`

**Fallback Phase 2 si déploy Vercel** : switch vers client-side Mermaid render (browser génère SVG, upload server). Pas de changement Phase 1.

### PDF brief — `@react-pdf/renderer`
Composants React déclaratifs → PDF.

**Structure (5 pages) :**
| Page | Contenu |
|---|---|
| 1. Cover | Nom du projet, idée 1-phrase, date, "Generated & validated by Claude Opus 4.7" |
| 2. Problem & User | Problème, utilisateur, contexte, urgence |
| 3. Architecture | PNG Mermaid embed + liste modules avec rôles |
| 4. Tech Stack | Frontend / Backend / DB / Infra / AI — chacun justifié 2-3 lignes |
| 5. MVP Plan + Risques | 3 étapes MVP + `hidden_constraints` + `recommended_alternative` (valeur Pass 2) |

**Style** : Inter / IBM Plex Mono, palette neutre, grille 8px.

### Page web `/result/<uuid>`
```
[Header : nom du projet]
[SVG zoomable de l'architecture]
[Tabs : Modules | Tech Stack | MVP Plan | Risks]
[Bouton : Download PDF]
[Bouton : Export JSON]
[Footer Phase 2 : "Want me to build this for you? → upsell"]
```

### Livrables au client
1. PDF brief (hero deliverable)
2. JSON export (pour devs)
3. PNG haute résolution architecture

Tous depuis `/result/<uuid>` (lien permanent UUID v4, bookmarkable).

## 8. Stratégie de tests

### Stack
- **Vitest** — unit + integration
- **Playwright** — E2E + Mermaid render
- **pdf-parse** — assertions PDF
- **MSW** — mock SDK Anthropic

### Pyramide

#### Unit (~50 tests)
- Logique branches (`shouldUnlock(branch, answers)` — pure)
- Zod schemas (questionnaire / pass1 / pass2 / final, valides + invalides)
- Utilitaires (UUID, paths, cache keys)
- Stubs Phase 1 (`verifyPayment` true, `getCurrentUser` DOGFOOD_USER)

#### Integration (~15 tests)
- Pipeline Claude avec SDK mocké (fixtures `__fixtures__/claude/*.json`)
- Storage roundtrip (FilesystemStorage + tmpdir)
- Mermaid render réel (Playwright headless)
- PDF generation : génère + parse via `pdf-parse` + assert sections clés

#### E2E (~3 tests)
1. Happy path : `/new` → 8 questions → preview → unlock → result → PDF download
2. Branche adaptive : type=Web + inputs=API → assert UI + Intégrations visibles
3. Error fallback : Pass 2 mock down → bannière "Senior review unavailable"

#### Live API smoke (manuel + nightly)
- E2E gated `RUN_LIVE_API=1` : vraie API, génère brief, vérifie coût < $3, valide Zod final
- Lancer avant chaque deploy Phase 2

### TDD
- Critique (branches, Zod, contrat pipeline) : test failing avant implémentation
- Rendu (Mermaid, PDF) : implémentation puis test (visual feedback nécessaire)

### Test fixtures
- `__fixtures__/questionnaires/` : 5 cas types (web simple / automation / SaaS / CLI / mobile)
- `__fixtures__/claude/` : pass1 + pass2 enregistrés pour ces 5 cas
- `npm run regen-fixtures` (manuel, consomme tokens)

### Manual QA dogfood (gate pour déclarer Phase 1 done)
Simon génère **5 briefs** sur ses propres idées et valide qualité. Si moins de 4/5 sont "utilisables sans retouche" → itérer prompts.

## 9. Acceptance criteria (Phase 1)

**Phase 1 est complete quand :**

1. ✅ `npm run dev` démarre l'app sur `localhost:3000`
2. ✅ Questionnaire 8 questions + branches fonctionne, save/resume via localStorage
3. ✅ Pipeline 2-pass Claude streame en live à l'écran
4. ✅ `/result/<uuid>` affiche l'architecture (SVG zoomable + tabs)
5. ✅ Download PDF fonctionne (5 pages, contient les sections attendues)
6. ✅ Download JSON fonctionne (validé Zod)
7. ✅ 4 stubs Phase 2 en place et testés (`payment`, `auth`, `storage`, `email`)
8. ✅ ≥ 68 tests verts (50 unit + 15 integration + 3 E2E)
9. ✅ Manual QA : 4/5 briefs Simon "utilisables sans retouche"
10. ✅ `.env.example` complet avec sections Phase 2 documentées
11. ✅ `docs/db-schema.sql` versionné

## 10. Non-goals (Phase 1)

Pour éviter scope creep :

- ❌ Pas d'auth utilisateur
- ❌ Pas de paiement Stripe
- ❌ Pas de DB (Postgres)
- ❌ Pas d'email transactionnel
- ❌ Pas de landing page commerciale (juste un CTA "Démarrer")
- ❌ Pas de dashboard admin
- ❌ Pas d'upsell "Build it for me"
- ❌ Pas de partage social du brief
- ❌ Pas de multi-langue (FR uniquement Phase 1)
- ❌ Pas de webhook entrant/sortant
- ❌ Pas de versionning du brief (1 génération = 1 PDF figé)

Tous ces items appartiennent à Phase 2 ou plus tard, et sont préparés via les hooks de la Section 6.

## 11. Open questions

Aucune. Tous les points ont été tranchés en brainstorm.

---

## Appendix : Décisions et raisonnement

| Décision | Raisonnement |
|---|---|
| Repo séparé de FORGE | User : "indépendant". Évite couplage Godot/Web. |
| Next.js (vs CLI vs HTML) | Phase 2 commerciale = web. Zéro réécriture. |
| 2-pass Claude (Sonnet → Opus) | Justifie le prix premium ($49.99). Sonnet pour structure rapide, Opus pour qualité finale. Marge ~96 %. |
| Pas de code livré ($49.99) | Calcul économique : custom build coûte 4-7h × $80-150/h + tokens. Non viable < $499. Architecture seule = auto, marge 96 %. |
| Questionnaire adaptatif | 8 base + 5 branches = 8 à ~22 questions selon complexité. Sweet spot conversion vs richesse du brief. |
| Phase 1 dogfood-first | Simon est le 1er user. Évite over-engineering. Validation avant commercialisation. |
| Filesystem storage Phase 1 | Pas de DB nécessaire. Interface `Storage` permet swap propre Phase 2. |
| Playwright Mermaid | Marche sans config en local. Phase 2 deploy : switch client-side si Vercel. |
