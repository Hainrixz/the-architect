# Contributing / Contribuir

Thanks for being here. This repo is a prompt library, not an application — contributions are words,
and words that go into an autonomous builder's context have to be exact.

Every section below is bilingual: English first, **ES** after.

*Gracias por estar aquí. Este repo es una librería de prompts, no una aplicación — las
contribuciones son texto, y el texto que entra al contexto de un constructor autónomo tiene que ser
exacto. Cada sección es bilingüe: inglés primero, **ES** después.*

---

## Read this first: do not send us your blueprint

**Do not open a pull request containing a blueprint you generated, or the project you built from
it.**

This is the single most common PR we get. Two of the three pull requests this repo has received were
exactly that — one of them was an entire generated Android app, source tree and all.

We understand completely why it happens. The Architect just spent an hour designing something with
you, the output is genuinely good, and this is the repo you were looking at while it happened. It
feels like where the work belongs.

It isn't, and here is the honest reason: **the blueprint is your output, not ours.** It describes
your product, your business, your data model, your pricing. We would be taking custody of something
that is yours — including whatever ended up in it that you would not have chosen to publish. And
merging it would help nobody: nothing in this repo reads `blueprints/`, so a merged blueprint sits
there as dead weight, and the next contributor sees a precedent that fills the repo with other
people's product specs.

**Where blueprints belong:** in your own project repo. `./blueprints/` in your working directory is
already the default output location, and `blueprints/` is in this repo's `.gitignore` precisely so a
blueprint generated inside a clone never gets committed by accident.

**If you want to share it,** link it from an issue. A real blueprint someone actually built from is
useful to look at — just not as a merged file.

**The contribution hiding in your blueprint.** If something went wrong, that *is* worth a PR or an
issue, and it is the highest-value thing you can send us:

| What you noticed | What to open |
|---|---|
| It classified your project as the wrong shape | An issue with your opening brief, verbatim, and the shape it should have picked |
| A pinned version was wrong or already broken | An issue naming the package, the pin, and what it should be |
| A build step had no real definition of done | An issue quoting the step |
| Your project type has no shape | A PR adding one — see below |
| A decision you had to make yourself, twice | A PR adding or extending a capability |

**ES — Lee esto primero: no nos mandes tu blueprint.**

**No abras un pull request con un blueprint que generaste, ni con el proyecto que construiste desde
él.** Es el PR más común que recibimos: dos de los tres PRs de este repo fueron exactamente eso — uno
era una app de Android completa, con todo su código.

Entendemos perfectamente por qué pasa. The Architect acaba de diseñar algo contigo durante una hora,
el resultado es bueno de verdad, y este es el repo que tenías abierto mientras ocurría. Se siente
como el lugar donde va el trabajo.

No lo es, y la razón honesta es esta: **el blueprint es tu salida, no la nuestra.** Describe tu
producto, tu negocio, tu modelo de datos, tus precios. Estaríamos quedándonos con algo que es tuyo —
incluyendo lo que haya terminado ahí dentro que no habrías elegido publicar. Y no le serviría a
nadie: nada en este repo lee `blueprints/`, así que un blueprint fusionado queda como peso muerto, y
el siguiente contribuidor ve un precedente que llena el repo con specs de producto ajenas.

**Dónde van los blueprints:** en el repo de tu propio proyecto. `./blueprints/` en tu directorio de
trabajo ya es la ubicación por defecto, y `blueprints/` está en el `.gitignore` de este repo justo
para que un blueprint generado dentro de un clon nunca se commitee por accidente.

**Si quieres compartirlo,** enlázalo desde un issue. Un blueprint real del que alguien construyó algo
es útil de ver — solo que no como archivo fusionado.

**La contribución que sí está escondida en tu blueprint:** si algo salió mal — clasificó tu proyecto
en la shape equivocada, una versión fijada estaba rota, un paso no tenía definición de terminado, tu
tipo de proyecto no existe como shape — *eso* sí es un PR o un issue, y es lo más valioso que puedes
mandarnos.

---

## The three rules that govern every file

1. **Version numbers live only in `knowledge/runtime-tracks/`.** Nowhere else. Not in a shape, not in
   a capability, not in a template, not in a command.
2. **Every build step has an observable, machine-checkable "Done when".**
3. **Never invent a skill name or an install command.**

Everything below is an application of these three.

**ES — Las tres reglas que gobiernan cada archivo:** (1) los números de versión viven solo en
`knowledge/runtime-tracks/`; (2) todo paso de construcción tiene un "Done when" observable y
verificable por una máquina; (3) nunca inventes el nombre de una skill ni un comando de instalación.

---

## The 3-axis split — understand this before you write anything

v1 rotted because every archetype file hardcoded its own stack table. Refreshing one library meant
editing thirteen files, so in practice most of them stayed wrong. v2 separates three orthogonal
things:

| Axis | Directory | Answers | Version pins |
|---|---|---|---|
| **Shape** | `knowledge/shapes/` (14) | *What is it?* | **Never** |
| **Runtime track** | `knowledge/runtime-tracks/` (5) | *What is it written in?* | **Only here** |
| **Capability** | `knowledge/capabilities/` (19) | *What does it do?* | **Never** |

Before you write, decide which axis your contribution belongs to. Most rejected PRs are a shape file
that quietly contains a runtime track, or a capability that is really a shape.

**ES:** v1 se pudrió porque cada arquetipo tenía su propia tabla de stack fija; refrescar una
librería significaba editar trece archivos. v2 separa tres cosas ortogonales: **shape** (qué es),
**runtime track** (en qué está escrito — el único lugar con versiones), y **capability** (qué hace).
Antes de escribir, decide a qué eje pertenece tu contribución.

---

## Contributing a new shape

A shape is a project type. Add one when a real project genuinely does not fit any of the 14 — not
when it is a variation of one that exists.

### The file

`knowledge/shapes/<kebab-name>.md`, 90–140 lines, stack-agnostic, no version numbers. These headings,
in this order:

```markdown
# Shape: {Human Name}

> {One sentence: what this is and who builds it.}

Last verified: YYYY-MM-DD

## Is this your project?

**Yes if:** {3-5 concrete signals, in the words a user would actually say}
**No if:** {at least 3 signals, each pointing at the correct shape by filename}

## Default runtime track

**{track}** — see `knowledge/runtime-tracks/{file}.md`. {One line why.}
Alternatives: {track} when {condition}.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|

## Data model

{Entities and relationships. Names only — no SQL dialect, no ORM syntax.}

## Directory structure

## Build order

{8-14 numbered steps, each one sitting of work, each with a "Done when".}

## Pitfalls

## Skills for the build phase

## See also
```

### The classification reciprocity rule

**If your shape's "No if" points at shape X, then X's "No if" must point back at yours.**

This is not a style preference. An audit of v2's classification graph found 20 of 60 edges were
one-way, which meant the interview reliably resolved to about 8 of 14 shapes and three shapes were
nearly unreachable — a user could describe exactly the thing a shape exists for and never land on it.
One-way edges are how a shape becomes dead code.

So a PR adding a shape touches **at least four files**: your new shape, and the shape files of every
neighbour you claimed. Say in the PR description which edges you added in both directions.

`saas-webapp.md` is the declared sink for ambiguous briefs and carries an exit to every other shape,
so it almost always needs an entry for yours.

### Register it

A shape nobody can reach does nothing. Add your signal row to:

- `skills/architect/SKILL.md` — the DISCOVERY classifier table
- `CLAUDE.md` — the clone-mode classifier table
- `questions/phase-1-discovery.md` — the discovery signals
- `questions/phase-2-branches.md` — a deep-dive section for your shape
- `knowledge/runtime-tracks/<track>.md` — the "Shapes that use this track" list

**ES — Contribuir una shape nueva.** Una shape es un tipo de proyecto; agrega una solo cuando un
proyecto real no encaje en ninguna de las 14, no cuando sea una variación de una existente. Archivo:
`knowledge/shapes/<nombre>.md`, 90–140 líneas, agnóstico de stack, sin versiones, con los encabezados
de arriba.

**Regla de reciprocidad:** si el "No if" de tu shape apunta a la shape X, el "No if" de X tiene que
apuntar de vuelta a la tuya. No es preferencia de estilo: una auditoría encontró que 20 de 60 aristas
eran de una sola dirección, y por eso la entrevista solo llegaba a unas 8 de las 14 shapes. Un PR que
agrega una shape toca **al menos cuatro archivos**. Y regístrala en `skills/architect/SKILL.md`,
`CLAUDE.md`, `questions/phase-1-discovery.md` y `questions/phase-2-branches.md`, o nadie llegará a
ella.

---

## Contributing a capability

A capability is a cross-cutting decision a project either needs or does not — auth, payments, an
agent loop, observability. It is not a project type and it is not a library.

`knowledge/capabilities/<kebab-name>.md`, 80–130 lines, no version numbers:

```markdown
# Capability: {Name}

> {One sentence: what problem this solves.}

Last verified: YYYY-MM-DD

## When a project needs this

{Concrete triggers from the interview — things a user says.}

## Decision matrix

| Option | Best for | Pros | Cons |
|---|---|---|---|

## Recommendation

{Name ONE default and say why. Then say when to deviate.}

## Data model additions

## Build steps this adds

{Numbered, each with a "Done when", ready to splice into a shape's build order.}

## Pitfalls

## See also
```

Two requirements beyond the template:

- **Link it from at least one shape's "Core capabilities" table.** An unlinked capability is
  unreachable — `availability-engine.md` sat orphaned through the whole v2 build because no shape and
  no classifier contained booking or scheduling language, so every booking brief quietly lost it.
- **Be opinionated.** A decision matrix that presents four options and no recommendation has moved
  the decision back onto the user, which is the one thing The Architect exists not to do.

**ES — Contribuir una capability.** Una capability es una decisión transversal que un proyecto
necesita o no (auth, pagos, agent loop, observabilidad). No es un tipo de proyecto ni una librería.
Archivo de 80–130 líneas, sin versiones, con los encabezados de arriba. Además: **enlázala desde la
tabla "Core capabilities" de al menos una shape** — una capability sin enlace es inalcanzable — y
**sé opinionado**: una matriz con cuatro opciones y ninguna recomendación le devuelve la decisión al
usuario, que es justo lo que The Architect existe para evitar.

---

## Refreshing a runtime track

This is the most valuable recurring contribution, and it is a PATCH release. See
[VERSIONING.md](VERSIONING.md).

**Re-verify every pin in the file. Not the ones you suspect — every one.** A file where half the rows
were checked today and half are eight months old is worse than one that is uniformly stale, because
the fresh `Last verified:` date makes the old rows look trustworthy.

How to check, in order of authority:

1. **The published package itself.** `npm view <pkg> version`, `npm view <pkg> dist-tags`,
   `pip index versions <pkg>`, `proxy.golang.org/<module>/@latest`. For a claim about an API — an
   export name, an option, a config key — read the published types or source in the tarball.
2. **The vendor's own docs or release notes**, for breaking changes and migration paths.
3. Nothing else. Not a blog post, not a search result, **not the model's memory**.

Then update the `Last verified:` line at the top to today's date, and record in the PR how you
checked.

**Why step 1 outranks step 2.** During the v2 runtime-track pass, research produced the React
Compiler Vite identifier `reactCompilerPreset`. A fact-checking pass "corrected" it to
`preconfiguredReactCompilerPlugin`. Grepping the published tarball's `dist/index.d.ts` settled it:
`reactCompilerPreset` appears three times, the other name appears zero times in any
`@vitejs/plugin-react*` package. **A claim that survives a web-search fact-check can still be wrong.**
When a package's own published types can answer the question, that is the only source that counts.

Other things that pass:

- Distinguish LTS from Current, and check which is which today. `latest` on a registry is routinely
  ahead of what the default stack actually supports — defaulting a track to a Current-but-not-LTS
  runtime is a real bug, not a nice-to-have.
- A caret on a `0.x` package still moves the minor. Pin `0.x` dependencies exactly.
- Never pin to an RC or a pre-release as the default.
- **If you cannot verify a pin, say so in the file** — "verify before use" beats a confident guess. A
  wrong pin poisons an entire build.

**Anything with a version number in it belongs in this directory.** If you find a version leaking
into a shape, a capability, a template, or a command, that is a bug — fix it by replacing the number
with a link to the track.

**ES — Refrescar un runtime track.** Es la contribución recurrente más valiosa y es un release PATCH.
**Reverifica todas las versiones del archivo, no solo las que sospechas** — mitad frescas y mitad de
hace ocho meses es peor que todas viejas, porque la fecha nueva hace que las viejas parezcan
confiables. Orden de autoridad: (1) el paquete publicado — `npm view`, dist-tags, y para una API, los
tipos dentro del tarball; (2) la documentación del vendor; (3) nada más — ni un blog, ni un resultado
de búsqueda, **ni la memoria del modelo**. Actualiza la línea `Last verified:` a la fecha de hoy.

Durante el pase de v2, la investigación dio `reactCompilerPreset`, una verificación "lo corrigió" a
`preconfiguredReactCompilerPlugin`, y el tarball publicado zanjó el asunto: el segundo nombre no
existe en ningún paquete. **Una afirmación que sobrevive a un fact-check puede seguir estando mal.**

---

## Adding a skill to the registry

`knowledge/skills-registry.md` is the authoritative list. A skill enters it only with all four:

1. **A verified install command.** Copied from the upstream repo's README or run yourself. Never
   reconstructed from what a command "usually" looks like. A blueprint that names a skill the builder
   cannot install breaks the self-containment promise.
2. **The real invocation form.** A leading `/` means it genuinely is a slash command. No slash means
   it **auto-activates** and must be named in prose. Writing a slash form for an auto-activating
   skill is a **silent no-op** — nothing runs, no error appears, and the workflow quietly skips the
   step. This is why `/deep-research`, `/seo-audit`, `/shadcn-ui`, `/chrome-bridge-automation`,
   `/web-reader`, `/humanizer`, and `/pdf-design` were all removed in v1.0.0: some did not exist at
   all, the rest were slash forms of skills that auto-activate.
3. **A license and star count read from the live GitHub API**, not estimated, with the date. The bar:
   first-party or well-established, licensed, pushed recently, and something we actually use. Repos
   maintained by tododeia.com are listed regardless of star count because we control them.
4. **A stated fallback.** No skill is ever a hard dependency. If it is absent, The Architect falls
   back to the knowledge base or built-in `WebSearch`/`WebFetch`, says so in one line, and keeps
   going.

Self-promotion is fine if the skill clears the bar. Say in the PR that it is yours.

**ES — Agregar una skill al registro.** `knowledge/skills-registry.md` es la lista autoritativa. Una
skill entra solo con las cuatro cosas: (1) un comando de instalación **verificado**, copiado del
README upstream o ejecutado por ti, nunca reconstruido; (2) la **forma de invocación real** — un `/`
al inicio significa que sí es slash command, sin `/` significa que se auto-activa, y escribir la
forma con slash de una skill que se auto-activa es un **no-op silencioso**: no pasa nada, no hay
error, y el flujo se salta el paso (por eso se eliminaron `/deep-research`, `/seo-audit`,
`/shadcn-ui`, `/chrome-bridge-automation`, `/web-reader`, `/humanizer` y `/pdf-design` en v1.0.0);
(3) licencia y estrellas leídas de la API de GitHub en vivo, con fecha; (4) un **fallback declarado**
— ninguna skill es dependencia dura. La autopromoción está bien si la skill cumple el criterio;
dilo en el PR.

---

## Writing style

Match the repo. It is opinionated and dense.

- **Recommend one thing and explain why.** Never five options and "which do you prefer?"
- **Tables and bullets over prose.** No walls of text.
- **Concrete over vague.** "max 300 lines per component" beats "keep files short".
- **No filler.** Cut "it's important to note", "in today's fast-paced world", "when it comes to".
- **Knowledge files are written in English.** The Architect detects the user's language at runtime
  and responds in it. Only `README.md`, this file, and `knowledge/localization.md` carry non-English
  text — and in the last one it is data (a vague-word blacklist, a terminology table), not prose.
  A PR that translates a knowledge file will be asked to revert it; a PR that *extends*
  `knowledge/localization.md` with a term or a failure mode you actually hit is welcome.
- **No emoji in knowledge files.**
- Every knowledge file carries a `Last verified:` line and ends with a `## See also` linking at least
  two real paths that exist.

**ES — Estilo.** Igual que el repo: opinionado y denso. Recomienda una cosa y explica por qué; tablas
y bullets en vez de prosa; concreto en vez de vago; sin relleno. Los archivos de `knowledge/` se
escriben **en inglés** — The Architect detecta el idioma del usuario en tiempo de ejecución; solo el
README y este archivo son bilingües. Sin emoji. Cada archivo lleva `Last verified:` y termina con un
`## See also` que enlaza al menos dos rutas reales.

**ZH —— 风格与语言。** 和本仓库一致：观点明确、信息密集。推荐一个方案并说明理由，不要罗列五个选项让人挑；
用表格和列表，不要大段散文；具体胜过含糊（"每个组件最多 300 行"胜过"文件别太长"）；不要废话；知识库文件
不用 emoji；每个文件带 `Last verified:` 行，并以 `## See also` 结尾、至少链接两条真实存在的路径。

**`knowledge/` 下的文件一律用英文写。** 它们是喂给模型的上下文，不是给人读的文档——The Architect 在运行时
检测用户语言并用该语言回应。只有 `README.md`、本文件和 `knowledge/localization.md` 含非英文内容，而后者里的
中文是**数据**（模糊词黑名单、术语对照表），不是叙述。把知识库翻译成中文的 PR 会被要求还原；给
`knowledge/localization.md` 补充术语或你真实踩到的失败模式的 PR，非常欢迎。

---

## Acceptance criteria, if your PR adds a build step

Anywhere you write a build step — in a shape, in a capability, in a template — it needs a completion
condition a script could evaluate today, on this machine, without leaving it.

| Not acceptable | Acceptable |
|---|---|
| "billing works" | `pnpm test src/api/webhooks` passes, and a `subscriptions` row appears after `stripe trigger checkout.session.completed` |
| "the store accepts the submission" | the packaging command produces a store-ready artifact and every required manifest field is non-empty |
| "no clipped or overlapping text" | a snapshot test reports no text-node truncation at min and max type scale |
| "a reviewer confirms it looks right" | a test asserts real command output byte-matches the documented example |

Prefer EARS form: **WHEN** `<trigger>` **THE SYSTEM SHALL** `<observable response>`.

Anything that genuinely needs an external party — store approval, a real device, a human sign-off —
goes in a post-build launch checklist, not the build order. It is still written down; it just is not
a gate, because an autonomous builder waiting on a review queue either stalls forever or
self-certifies, and both are worse than having no criterion.

Also banned: a criterion the blueprint itself already satisfies before any code is written. That
gates nothing.

**ES — Criterios de aceptación.** Todo paso de construcción necesita una condición que un script
pueda evaluar hoy, en esta máquina, sin salir de ella. Prefiere la forma EARS: **WHEN** `<disparador>`
**THE SYSTEM SHALL** `<respuesta observable>`. Lo que de verdad requiere a un tercero (aprobación de
store, un dispositivo real, una firma humana) va a un checklist post-build, no al build order: un
constructor autónomo esperando una cola de revisión se cuelga para siempre o se autocertifica, y
ambas son peores que no tener criterio.

---

## Before you open the PR

- [ ] No version number outside `knowledge/runtime-tracks/`.
- [ ] Every path you reference exists. Open each one.
- [ ] No skill name or install command that you did not verify.
- [ ] No slash prefix on an auto-activating skill.
- [ ] Every build step has a machine-checkable "Done when".
- [ ] `Last verified:` line present and current.
- [ ] `## See also` links at least two real paths.
- [ ] New shape? Reciprocity done in both directions, and registered in the classifier tables.
- [ ] You ran a real interview end to end against your change and it behaved.
- [ ] No generated blueprint in the diff.

Then say in the PR description which SemVer category it falls into per
[VERSIONING.md](VERSIONING.md) — MAJOR, MINOR, or PATCH — and why. If you are unsure, say that
instead of guessing; getting this wrong is how a rename ships as a patch and breaks people.

**Testing your change.** There is no test suite; the test is a real session. Run `/architect` (or
`claude` in a clone) end to end with a brief that should hit your change, and confirm the interview
reaches it and the generated blueprint reflects it. Then run `/architect-audit` on the result.

**ES — Antes de abrir el PR:** sin versiones fuera de `runtime-tracks/`; cada ruta que referencias
existe (ábrelas); ningún nombre de skill ni comando de instalación sin verificar; ningún `/` en una
skill que se auto-activa; todo paso con "Done when" verificable; `Last verified:` presente; `See
also` con dos rutas reales; si es una shape nueva, reciprocidad en ambas direcciones y registrada en
los clasificadores; corriste una entrevista real de punta a punta; y **ningún blueprint generado en
el diff**. Di en la descripción a qué categoría SemVer corresponde según
[VERSIONING.md](VERSIONING.md) y por qué. No hay suite de tests: la prueba es una sesión real —
corre `/architect` completo y luego `/architect-audit` sobre el resultado.

---

## Reporting a security issue

Do not open a public issue. See [SECURITY.md](SECURITY.md).

**ES:** No abras un issue público. Ver [SECURITY.md](SECURITY.md).

---

## License

By contributing you agree your contribution is licensed under the MIT License, the same as the rest
of the repo. See [LICENSE](LICENSE).

**ES:** Al contribuir aceptas que tu contribución se licencia bajo MIT, igual que el resto del repo.
