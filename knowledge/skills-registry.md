# Skills Registry

Maps available Claude Code skills to blueprint sections. Use this during Phase 4 (Generate) to recommend the right skills for the build phase.

## Skills for Blueprint Recommendations

### UI & Diseño
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/ui-ux-pro-max` | Design System | Any project with a UI | 67 estilos, 161 paletas, 57 font pairings, 99 UX guidelines |
| `/frontend-design` | Frontend Architecture, Build Order | Any project with a UI | Interfaces production-grade distintivas |
| `/design` | Design System | When full design pass needed | Diseño visual completo de componentes |
| `/design-system` | Tech Stack, Build Order | When building component library | Tokens, primitives, design system base |
| `/ui-styling` | Frontend Architecture | Styling decisions & CSS | Estilos visuales avanzados |
| `/web-design-guidelines` | Frontend Architecture | Any web project | Guías de implementación web |
| `/banner-design` | Marketing / Content | Marketing sites, landing pages | Banners y creatividades de marketing |
| `/brand` | Brand Identity | New product with brand needs | Sistema de identidad de marca |
| `/slides` | Pitch / Docs | Presentations, decks, docs | Slides y presentaciones |

### Componentes & Stack
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/shadcn-ui` | Tech Stack, Build Order | When shadcn/ui chosen | Buscar e implementar componentes shadcn |
| `/building-components` | Build Order | React component-heavy projects | Componentes React production-ready |
| `/vercel-react-best-practices` | Performance, Build Order | Next.js / React projects | Performance y best practices React |

### SaaS / Pagos
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/payment-advisor` | Tech Stack (payments) | Any SaaS with monetization | Estrategia de integración de pagos |
| `/integration-builder` | Build Order | When connecting external services | Construcción de integraciones y APIs |
| `/webhook-verifier` | Build Order | When webhooks are needed | Verificación y testing de webhooks |
| `/project-analyzer` | Build Order | SaaS architecture review | Análisis y validación del proyecto pago |
| `/doctor` | Testing | Payment system health check | Diagnóstico del sistema de pagos |

### Research & Automatización
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/deep-research` | Tech Stack (during design) | Comparing tech, unfamiliar tools | Investigación web multi-ángulo |
| `/web-reader` | Build Order | Web scraping / content extraction | Extracción de contenido de páginas |
| `/chrome-bridge-automation` | Build Order | Browser workflow automation | Automatización con visión de pantalla |
| `/playwright-cli` | Testing Strategy, Build Order | E2E testing | Automatización y testing de browser |

### Contenido & SEO
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/humanizalo` | Build Order (content) | Any project with copy or content | Elimina patrones AI, texto natural |
| `/humanizer` | Build Order (content) | Alt. humanizador de textos | Reescritura con patrones humanos |
| `/seo-audit` | Deployment Strategy, Build Order | Marketing sites, SaaS, public sites | Auditoría SEO técnica completa |

### Despliegue
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/all-deploy` | Deployment | Any deployable project | Auto-detecta stack → Vercel/Railway/Docker |
| `/vercel-deploy` | Deployment | Next.js / Vercel projects | Despliegue específico para Vercel |

### Calidad de Código
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/karpathy-guidelines` | Tech Stack, Build Order | ALL projects | Principios Karpathy: simplicity, surgical changes |

### Aprendizaje
| Skill | Blueprint Section | When to Recommend | What It Does |
|-------|------------------|-------------------|-------------|
| `/aprende` | Always | ALL projects | Captura conocimiento de la sesión |
| `/learn` | Always | ALL projects | Inglés: session knowledge capture |

---

## Skills The Architect Uses During Design

| Skill | Phase | Purpose |
|-------|-------|---------|
| `/deep-research` | Phase 2 | Compare technologies, research patterns |
| `/ui-ux-pro-max` | Phase 3 | Design the visual system (colors, fonts, style) |
| `/find-skills` | Phase 2 | Discover skills to recommend in the blueprint |
| `/playwright-cli` | Phase 3 | Screenshot reference sites the user mentions |
| `/chrome-bridge-automation` | Phase 3 | Alternative for reference site analysis |

---

## Templates de Blueprint por Tipo de Proyecto

### SaaS
```
| Skill | When to Use | Why |
|-------|-------------|-----|
| /karpathy-guidelines | Siempre | Código quirúrgico y limpio |
| /ui-ux-pro-max | Step 1 (design system) | Sistema de diseño profesional |
| /frontend-design | Steps 3-7 (UI) | Interfaces production-grade |
| /building-components | Steps 3-7 | Componentes React robustos |
| /shadcn-ui | Step 2 (scaffolding) | Setup componentes |
| /payment-advisor | Step 8 (payments) | Estrategia de pagos correcta |
| /integration-builder | Step 9 (integrations) | APIs externas |
| /webhook-verifier | Step 10 (webhooks) | Stripe/webhooks |
| /humanizalo | Step 11 (copy) | Copy natural sin AI |
| /seo-audit | Step 12 (pre-launch) | SEO antes de lanzar |
| /cyber-neo | Step 13 (security) | Audit OWASP completo |
| /all-deploy | Step 14 (deploy) | Deploy a producción |
```

### Web / Landing
```
| Skill | When to Use | Why |
|-------|-------------|-----|
| /ui-ux-pro-max | Step 1 | Sistema de diseño |
| /frontend-design | Steps 2-5 | UI distintiva |
| /shadcn-ui | Step 2 | Componentes |
| /web-design-guidelines | Steps 2-5 | Best practices web |
| /humanizalo | Step 6 (copy) | Textos naturales |
| /seo-audit | Step 7 | SEO |
| /cyber-neo | Step 8 | Seguridad |
| /all-deploy | Step 9 | Deploy |
```

### App Móvil
```
| Skill | When to Use | Why |
|-------|-------------|-----|
| /ui-ux-pro-max | Step 1 | Mobile design system |
| /building-components | Steps 3-7 | Componentes nativos |
| /playwright-cli | Step 8 | Testing E2E |
| /cyber-neo | Step 9 | Seguridad |
```

---

## How to Include in Blueprint

In Section 14 (Skills to Use During Build), list each relevant skill with:
1. **Skill name** — the slash command
2. **When to use** — which build step (reference the build order)
3. **Why** — what it helps accomplish
