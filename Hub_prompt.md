# Build Prompt — Patient Hub (`/hub`)

A complete, reusable specification for building a mobile-first **Patient Hub**
single-page app for a multi-speciality hospital. Adapted from the Apex Super
Speciality Hospital implementation. Replace every `[BRACKETED]` value with the
new client's details.

---

## What you're building

A mobile-first, single-page **Patient Hub** route at `/hub` for a
multi-speciality hospital. Patients open it from a QR code at reception,
WhatsApp/SMS links, or Instagram. It loads fast, looks clean and trustworthy,
and works perfectly on phones. The hub is one route with an internal screen
switcher (home ↔ review ↔ booking ↔ whatsapp ↔ pay ↔ website) animated with
framer-motion.

## Stack

- **TanStack Start v1** (React 19) — `createServerFn` for backend logic,
  `createFileRoute` for the `/hub` route with `head()` SEO metadata
- **Tailwind CSS v4** with semantic design tokens defined in `src/styles.css`
  (oklch colors, `font-hub` family loaded via `<link>` in `__root.tsx`)
- `framer-motion` for screen transitions, `sonner` for toasts,
  `lucide-react` for icons, `zod` for server validation

## File layout

```
src/
├── routes/hub.tsx                  # createFileRoute("/hub") with head() SEO
├── pages/PatientHub.tsx            # page shell: header, screen switcher, footer
├── lib/
│   ├── hub-config.ts               # ALL clinic config + specialty/treatment data + label maps
│   ├── hub-i18n.ts                  # EN/te STRINGS + t()/tf() helpers
│   ├── hub-review-library.ts        # ~100 ready-made fallback review drafts (deterministic picker)
│   ├── hub-review.functions.ts      # createServerFn → Groq AI (openai/gpt-oss-120b)
│   ├── hub-store.functions.ts       # createServerFn → appointments + click counts (in-memory; ready for Cloud)
│   └── hub-tracking.ts              # CRM external_form_submission helper (form_tracking integration)
└── components/hub/
    ├── shared.tsx                   # BackBar, Field, Section, FireClick type
    ├── HubSelect.tsx                # themed custom dropdown (native <option> can't be styled)
    ├── ReviewFlow.tsx               # the 4-step review generator + draft card + regenerate + post
    ├── ReviewDemoAnimation.tsx      # copy→paste→post modal popup
    ├── BookingFlow.tsx              # appointment form
    ├── WhatsAppFlow.tsx             # WhatsApp quick-start chips
    ├── PayFlow.tsx                  # payment form + presets
    └── WebsiteFlow.tsx              # website link + services accordion
```

---

## 1. Configuration — `src/lib/hub-config.ts`

A single `HUB_CONFIG` object the whole hub reads from. Everything
clinic-specific lives here so a new client just edits this file.

```ts
export type Lang = "en" | "te";
export type Specialty = "dental" | "ent" | "general" | "cosmetology" | "diabetes";

HUB_CONFIG = {
  clinic: {
    name,              // "Apex Super Speciality Hospital"
    fullName,          // "Dr. Viswanath's Apex Super Speciality Hospital"
    tagline,           // "Healthy Smiles, Trusted Care"
    city, state,
    address,           // full street address
    hours: { weekdays, sunday },
    phones: ["+91..."],             // display (tel: links)
    whatsappNumber: "91XXXXXXXXXX", // wa.me target, country code, no +
    logoUrl,                        // absolute https URL
    instagram, facebook, websiteUrl,
  },
  googleReviewLink,    // https://search.google.com/local/writereview?placeid=...
  googleMapsLink,      // https://www.google.com/maps/search/?api=1&query=...
  booking: { calendarEmbed, webhookUrl },   // iframe calendar URL + optional webhook
  payment: { link, presets: [{ label, amount }] },
  doctors: [{ id, name, specialty, short, photo, specialtyIds }],
  treatments: [...31 canonical English strings...] as const,   // ends with "Other"
  experienceChips: [...10 English strings...],
  trustPoints: [{ icon: "shield"|"award"|"monitor"|"wallet", title, desc }],
  rateLimit: { maxGenerations: 5, perHour: 1, maxRegenerations: 3 },
}
```

Also export:

- `SPECIALTIES: SpecialtyInfo[]` — `{ id, label, labelTe, icon, blurb }` per
  specialty
- `TREATMENT_SPECIALTY: Record<Treatment, Specialty>` — maps each treatment
  string → its specialty
- `specialtyForTreatment(t)`, `treatmentsBySpecialty(id)`,
  `treatmentLabel(t, lang)`, `chipLabel(c, lang)`, `sectionLabel(key, lang)`
- `TREATMENT_LABELS_TE`, `EXPERIENCE_CHIPS_TE`,
  `TREATMENT_SECTION_LABELS_TE` — Telugu label maps keyed by the canonical
  English string. **The AI payload and DB always use the canonical English
  keys; Telugu is display-only.**

### Treatment list (canonical English keys, grouped)

- **Dental:** Check-up & Cleaning (Scaling), Root Canal Treatment, Dental
  Implants, Braces / Aligners, Teeth Whitening, Tooth Extraction / Wisdom
  Tooth, Fillings, Crowns & Bridges, Dentures, Kids Dentistry, Smile Makeover
- **ENT:** Hearing & Balance Care, Sinus & Allergy Treatment, Throat & Voice
  Disorders, Ear Infections & Wax Removal, Nasal Obstruction / Deviated
  Septum
- **General Medicine:** Routine Health Checkup, Hypertension Management,
  Chronic Illness Care, Vaccinations & Immunization, Fever / Acute Illness
- **Cosmetology:** Skin Rejuvenation, Anti-Aging Treatment, Pigmentation &
  Acne Care, Hydra Facial, Advanced Hair Care, Skin Tightening
- **Diabetes:** Blood Sugar Monitoring, Diet & Nutrition Counseling, Diabetes
  Management, Complication Screening
- **Catch-all:** Other

### Experience chips (10)

Painless treatment, Friendly staff, Clean & hygienic hospital, Explained
everything clearly, Affordable pricing, Quick appointment, Very gentle
doctor, Modern equipment, Comfortable for kids, Great results.

---

## 2. i18n — `src/lib/hub-i18n.ts`

`STRINGS = { en: {...}, te: {...} }` mirroring every label the UI shows:

- greeting, card titles + subs (5 action cards)
- review step titles + hints, star labels
  (`["Poor","Okay","Good","Great","Excellent"]`),
- booking fields, WhatsApp chips, payment fields/purposes,
- website section, doctors/trust section, footer privacy
- `reviewRegenerateCount(n, max)` as a function

`t(lang, key)` returns the string; `tf(lang, key, ...args)` resolves
function-valued strings. **Keep the approved Telugu wording verbatim** — do
not paraphrase or auto-translate.

### Section labels (final wording)

| English (source) | Telugu |
|---|---|
| 1. What treatment did you receive? | 1. మీరు ఏ చికిత్స పొందారు? |
| Choose a department to explore available treatments. | అందుబాటులో ఉన్న చికిత్సల వివరాల కోసం ఒక విభాగాన్ని ఎంచుకోండి. |
| 2. How was your experience? | 2. మీ చికిత్స అనుభవం ఎలా ఉంది? |
| Tap a star to rate your experience. | మీ అనుభవానికి రేటింగ్ ఇవ్వడానికి నక్షత్రంపై నొక్కండి. |
| 3. What did you appreciate about your visit? | 3. ఆసుపత్రికి వచ్చినప్పుడు మీకు ఏ విషయాలు నచ్చాయి? |
| Select everything that describes your experience. | మీ అనుభవానికి సరిపోయే అన్ని అంశాలను ఎంచుకోండి. |
| 4. Would you like to share anything else? | 4. మీరు ఇంకా ఏమైనా చెప్పాలనుకుంటున్నారా? |
| Add a few words about your experience (optional). | మీ అనుభవం గురించి కొన్ని మాటలు రాయండి (తప్పనిసరి కాదు). |

---

## 3. Page shell — `src/pages/PatientHub.tsx`

- `useState<Lang>("en")`, `useState<Screen>("home")` where
  `Screen = "home"|"review"|"book"|"whatsapp"|"pay"|"website"`.
- `useServerFn(trackClick)` → a `fire(action)` callback that reads `?src=`
  from `window.location.search` and calls
  `track({ data: { action, src } })`.
- **Sticky header**: logo img + clinic name + tagline (left), `LangToggle`
  EN/తె pill (right). `max-w-md`, backdrop blur.
- **Main**: `max-w-md px-4 pb-24`, `AnimatePresence mode="wait"` switching
  screens with slide transitions.
- **Home screen** (`HomeScreen` + `ExtraSections`):
  - Gradient teal greeting banner with city/state + greeting.
  - 2-col grid of 5 action cards (5th spans `col-span-2`), each
    `min-h-[116px]`, icon chip + bold label + sub, hover lift,
    `active:scale-[0.98]`, fires `fire(cardClick)` then `onOpen(screen)`.
  - Call Us (`tel:`) + Get Directions (`googleMapsLink`, new tab) buttons,
    `min-h-[56px]`.
  - "Our Specialties" badges card + Address/Hours/Phone info card.
  - **ExtraSections**: Meet Our Doctors cards (photo, name, specialty, short)
    + Why Patients Trust Us 2×2 grid.
- **Footer**: dark `dental-ink` card — logo, address, hours, phone,
  Instagram/Facebook, privacy note ("We don't store your medical details on
  this page."), © year fullName.

---

## 4. Review generator — `src/components/hub/ReviewFlow.tsx`

Four `Section` cards in order (the `Section` helper renders a white
rounded-3xl card with title + optional hint):

### Step 1 — What treatment did you receive?

Hint: "Choose a department to explore available treatments."

A single drill-down card:

- `activeSpecialty` state (`null` = specialty list). Shows `SPECIALTIES`
  (the 5 specialties + an "Other" entry) as full-width buttons with a
  chevron, label, blurb, and a count badge for already-selected treatments in
  that specialty.
- Tapping a specialty sets `activeSpecialty` and shows its treatments as
  **multi-select chips** (`toggleTreatment`), with a "← All specialties"
  back link.
- **Switching specialties never clears prior selections** — `treatments`
  accumulates across all specialties.

### Step 2 — How was your experience?

- Five empty stars (`rating` starts at 0).
- Tapping star *n* fills 1..n gold (`fill-amber-400`).
- Shows label below ("Tap to rate" when 0, else
  `reviewStars[rating-1]`).
- Stars are `h-11 w-11`, centered.

### Step 3 — What did you appreciate about your visit?

- 10 multi-select experience chips (`toggleChip`), selected = teal/mint
  fill + "✓".

### Step 4 — Would you like to share anything else?

- Optional textarea, `maxLength={200}`, char counter.

### Generate / Regenerate / Post

- **Generate button**: `disabled` unless
  `treatments.length > 0 && rating >= 1 && !loading`. Calls `runGenerate`.
- **`runGenerate`**: increments `variationRef` (drives fresh wording +
  fallback rotation), calls `generateReviewDrafts` server fn with
  `{ treatment: treatments.join(", "), rating, chips, note, lang, specialty,
  variation }`. On success sets `drafts[0]` (one draft),
  `setShowPostPopup(true)`. On failure `toast.error`.
- **Draft card** (single): tag pill + editable `<textarea>` + Copy button
  (`navigator.clipboard.writeText` + "Copied! ✅" toast + 1.8s reset).
- **Regenerate**: `disabled` when `regens >= maxRegenerations || loading`,
  increments `regens`, calls `runGenerate`. Shows
  `reviewRegenerateCount(regens, max)`.
- **Post on Google**: pulsing-ring `dental-ink` button →
  `setShowPostPopup(true)`. Helper text below. If `rating <= 3`, a
  non-blocking "Want to tell us privately? Message us on WhatsApp" link
  (opens wa.me with a private-feedback message) — **the Google button stays
  visible (no review gating).**
- Renders
  `<ReviewDemoAnimation open={showPostPopup} lang draft={drafts[0]?.text} onClose>`.

---

## 5. Copy→Paste→Post popup — `src/components/hub/ReviewDemoAnimation.tsx`

A modal that opens **only after successful generation** (and from the Post
button). Bottom-sheet on mobile (`items-end`, `rounded-t-3xl`), centered
dialog on `sm:` (`sm:items-center`, `sm:rounded-3xl`, `sm:max-w-md`). Dark
backdrop with blur, click-to-close + Escape.

- **Looping animation** (reset on open): a simulated `MousePointer2` cursor
  moves through 3 scenes via a `PHASE_MS` timeline + `ORDER` array —
  `copyIdle→copyClick→copyDone→toGoogle→pasteIdle→pasteClick→typing→postIdle→postClick→success→reset`.
  Step pills 1/2/3 with done checks + connectors.
  - `CopyScene` (sample review card + copy button → checkmark)
  - `GoogleScene` (stylized "G", 5 gold stars, text area that streams the
    review in during `typing`)
  - `SuccessScene` (spring check + "Review posted!" + thanks)
- **"Copy & Post on Google"** button: `await
  navigator.clipboard.writeText(draft)` (the *real* generated draft) → on
  success shows "Copied!" + `window.open(HUB_CONFIG.googleReviewLink,
  "_blank", "noopener,noreferrer")`. Clipboard failure → `toast.error`.
  **Never auto-posts** — the patient pastes and posts manually on Google.
- **Cancel** button: closes popup, preserves the editable draft +
  regenerate state.

---

## 6. AI server function — `src/lib/hub-review.functions.ts`

`createServerFn({ method: "POST" })` validated with zod (`treatment` ≤400,
`rating` 1–5, `chips` ≤10, `note` ≤200, `lang`, `specialty`, `variation`).
**Keep it a thin wrapper** — the system prompt, user-prompt builder, parser,
and fallback picker live in `hub-review-library.ts` (imported), per the
server-fn splitting rule.

- **API**: `POST https://api.groq.com/openai/v1/chat/completions`, model
  `openai/gpt-oss-120b`, `temperature: 0.9`, `max_tokens: 400`,
  `Authorization: Bearer ${process.env.GROQ_API_KEY}` — read **inside**
  `.handler()`. Key registered as a project secret; never in the browser.

### System prompt (verbatim)

> You help patients express their genuine healthcare experience in their own
> words. Create a concise, natural first-person Google review draft using only
> the information explicitly provided by the patient. Preserve the patient's
> intended sentiment and rating. Never invent experiences, treatment outcomes,
> staff behavior, medical claims, prices, or promises. Do not exaggerate,
> manipulate, or force positive sentiment. The review must reflect the
> patient's actual experience rather than promote the hospital. Never include
> private medical information or personally identifiable information. Do not
> add keywords unnaturally, request incentives, or encourage a particular
> rating. If information is limited, write a simple, honest review without
> adding unsupported details.

### User prompt

- Language instruction (Telugu → "Write the ENTIRE review in Telugu script
  (తెలుగు లిపి)" with treatments/chips translated naturally; English
  otherwise)
- Hospital context; department + treatment + rating + chips (if any) +
  patient note (if any)
- Sentiment guidance (1–3 → honest/neutral, no forced positivity; 4–5 →
  genuinely positive, no exaggeration)
- Rules (no invented facts/outcomes/prices, no "best hospital"/"best
  doctor" SEO phrases, max 1–2 emoji, no incentives)
- Variation nudge for fresh wording on regenerate
- `"Return ONLY a single JSON object: {"text":"..."}"`

### Parse

Extract first valid `{"text": string}` (defensively also accept a
single-element array).

### Fallback

If no key, non-ok, or parse fails → `pickFallbackDraft({ treatment, rating,
specialty, offset: variation })` from the ~100-draft library (deterministic
by offset, respects rating so 1–3 stars get honest/constructive drafts).
Return `{ ok: true, drafts: [draft], fallback: true }`.

### Rate limit

In-memory `checkRateLimit(deviceId, 50)` per hour; if exceeded, return a
fallback draft.

---

## 7. Booking — `src/components/hub/BookingFlow.tsx`

Form: name, mobile (+91 prefix span, 10-digit `/^[6-9]\d{9}$/` validation),
treatment (`HubSelect` grouped by specialty with Telugu labels), preferred
doctor (`HubSelect`, optional, "No preference" first), preferred date
(`type=date`, min today), preferred time (`HubSelect`:
Morning/Afternoon/Evening + "No preference"), note textarea. Honeypot hidden
`company` field. Required consent checkbox ("I agree to be contacted about my
appointment"). Submit → `submitAppointment` server fn (in-memory store)
**and** `trackBookingSubmission(...)` (CRM, fire-and-forget) → success card
with WhatsApp confirm link. Below the form: the embedded
`booking.calendarEmbed` iframe (`h-[460px]`).

---

## 8. WhatsApp / Pay / Website flows

### WhatsApp — `src/components/hub/WhatsAppFlow.tsx`

- 4 quick-start chips (Book appointment, Treatment cost, Reports/prescription,
  Emergency) → prefill
  `wa.me/<whatsappNumber>?text=Hello <name>, <chip>.`
- "Open WhatsApp Chat" green button.

### Pay — `src/components/hub/PayFlow.tsx`

- name + mobile + amount (₹) + purpose select (Consultation/Treatment
  payment/Advance/Other) → "Pay Now" opens `payment.link`.
- Quick-pay preset grid.
- Accepted-methods + secure-receipt note.

### Website — `src/components/hub/WebsiteFlow.tsx`

- Opens `websiteUrl` in new tab + "Our Services" accordion: per-specialty
  card with an icon + blurb header, then expandable rows of treatments with
  one-line descriptions (EN + te maps in the file).

---

## 9. Shared UI — `src/components/hub/`

- `HubSelect`: themed custom listbox (trigger with chevron, scrollable
  `max-h-72` listbox, mint highlight + check on selected, grouped headings
  for treatments). Outside-click + Escape close.
- `BackBar` (back arrow + screen title), `Field` (uppercase label + input),
  `Section` (rounded-3xl white card with title/hint). `FireClick` type for
  the 5 click actions.

---

## 10. Persistence + tracking

- `hub-store.functions.ts`: `submitAppointment` (zod-validated, in-memory
  array, shapes mirror a future `public.appointment_requests` Cloud table),
  `trackClick` (5 actions + `?src=`), `getHubStats`. In-memory only until
  Cloud is enabled — then move to `public.appointment_requests` +
  `public.hub_clicks` with RLS `TO authenticated`/`TO anon` as appropriate.
- `hub-tracking.ts`: `trackBookingSubmission` POSTs an
  `external_form_submission` event to the CRM `form_tracking` integration
  endpoint with `first_name`/`phone` in `formData` + custom-field IDs for
  treatment/doctor/date/slot/note, reusing the site's `__vibe_sid` session
  cookie. **Connect the new project's own `form_tracking` integration via
  `request_integration` and use its tracking/location/project IDs + register
  its custom fields** — do not copy another project's IDs.

---

## 11. Design system — `src/styles.css`

Define semantic tokens in oklch:

```
--dental-teal, --dental-pine, --dental-mint, --dental-sand,
--dental-ink, --dental-body, --dental-muted, --dental-border,
--dental-bg, --dental-sel
```

Plus `--font-hub` (Plus Jakarta Sans, loaded via `<link>` in
`src/routes/__root.tsx` head, never `@import` in css). Use `font-hub`,
`rounded-3xl` cards, soft shadows, `min-h-[56px]` primary buttons,
`active:scale-[0.98]`. A `hub-input` utility class styles all inputs/selects
consistently. Mobile-first `max-w-md` container throughout.

---

## 12. Route + SEO — `src/routes/hub.tsx`

```ts
export const Route = createFileRoute("/hub")({
  head: () => ({
    meta: [
      { title: "<Hospital name> <City> – Patient Hub" },
      { name: "description", content: "..." },
      { name: "robots", content: "index,follow" },
      { name: "theme-color", content: "#1F5FBF" },
      { property: "og:title", content: "..." },
      { property: "og:description", content: "..." },
      { property: "og:type", content: "website" },
      { name: "twitter:card", content: "summary" },
    ],
  }),
  component: PatientHub,
});
```

Distinct from the home route's metadata.

---

## 13. Workflow when handed to a builder

1. Fill `HUB_CONFIG` with the new clinic's name, address, hours, phones,
   WhatsApp number, logo, social/website links, Google review + Maps links,
   booking calendar embed, payment link + presets, doctors, and the
   canonical treatment/chip lists.
2. Add the EN/te string tables (keep the approved Telugu wording).
3. Register `GROQ_API_KEY` as a project secret; set the fallback review
   library.
4. Connect the `form_tracking` integration and wire `hub-tracking.ts` to its
   IDs + custom fields.
5. Verify on `/hub`: language toggle flips every label + AI output language;
   review generates one editable draft; regenerate rotates wording;
   copy→paste→post popup copies the real draft then opens Google only on
   success; booking form validates + submits + fires CRM tracking;
   WhatsApp/Pay/Website open the right links; click counts + `?src=`
   recorded.

---

## Checklist of things to configure per client

- [ ] `HUB_CONFIG.clinic` — name, fullName, tagline, city, state, address,
      hours, phones, whatsappNumber, logoUrl, instagram, facebook, websiteUrl
- [ ] `HUB_CONFIG.googleReviewLink` — Google Place ID review link
- [ ] `HUB_CONFIG.googleMapsLink` — Maps directions link
- [ ] `HUB_CONFIG.booking.calendarEmbed` — embedded calendar iframe URL
- [ ] `HUB_CONFIG.payment.link` + `presets` — payment/checkout link + common
      amounts
- [ ] `HUB_CONFIG.doctors` — name, specialty, short, photo, specialtyIds
- [ ] `HUB_CONFIG.treatments` + `experienceChips` + `trustPoints` — canonical
      lists (or keep the 31-treatment multi-specialty set)
- [ ] `GROQ_API_KEY` — project secret for the AI review generator
- [ ] `form_tracking` integration — connect via builder, wire IDs + custom
      fields in `hub-tracking.ts`
- [ ] Telugu label tables in `hub-i18n.ts` + `hub-config.ts` — keep approved
      wording verbatim