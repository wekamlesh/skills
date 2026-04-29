# Architecture Reference

## Standard Project Layout

```text
src/
  pages/          - File-based routing (index.astro, blog/[slug].astro, etc.)
  layouts/        - Page wrapper components (BaseLayout.astro, BlogLayout.astro)
  components/     - Reusable UI components (Header, Footer, Card, etc.)
  content/        - Content collections (MDX/Markdown blog posts, docs pages)
  styles/         - Global styles, Tailwind base customizations
  utils/          - Helper functions (reading time, date formatting, etc.)
public/           - Static assets served as-is (images, fonts, favicon)
astro.config.mjs
package.json
tailwind.config.mjs
```

## Blog System

Build complete blog systems including:
- Markdown / MDX content flow with content collections
- Categories and tags
- Author pages
- Related posts
- Reading time estimate
- Search (pagefind or custom)
- Pagination
- RSS feed (`@astrojs/rss`)
- SEO meta templates per post
- Featured images with `<Image />` optimization
- Syntax highlighting (Shiki default)

## Docs System

Build docs systems including:
- Sidebar navigation with active state
- Search (pagefind or Algolia)
- Version-ready folder structure
- MDX support with custom components
- Code blocks with copy-to-clipboard buttons
- Good readability and generous spacing
- API docs patterns

## CMS Integrations

When requested, support headless CMS integrations with clean data flow:
- **Contentful** — `contentful` SDK, typed entries
- **Sanity** — `@sanity/client`, GROQ queries
- **Strapi** — REST or GraphQL API
- **Ghost** — Content API
- **Hygraph** — GraphQL
- **Custom APIs** — `fetch` in `.astro` frontmatter or API routes

## Monetization Systems

When useful, support:
- Newsletter signup (ConvertKit, Mailchimp, Resend)
- Lead capture forms
- Contact funnels
- Waitlists
- Member-only areas
- Product funnels and landing pages
- Community onboarding flows

## Design Direction

Default styles:
- Minimal clean
- Modern SaaS
- Elegant corporate
- Startup bold

Visual principles:
- Premium spacing and strong typography
- Clean hierarchy with clear focal points
- Mobile-first responsiveness
- Accessible UI (WCAG AA minimum)
- Fast-loading with trust-building layouts
- Strong CTA positioning where relevant

Motion: use sparingly and purposefully. Animations should improve perceived quality, not reduce speed or distract.
