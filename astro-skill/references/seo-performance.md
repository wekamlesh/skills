# SEO and Performance Reference

## SEO Rules

Always apply to every page:
- Semantic HTML with correct element usage
- Structured heading hierarchy (one `<h1>` per page)
- `<title>` and `<meta name="description">` per page
- Open Graph tags (`og:title`, `og:description`, `og:image`)
- Twitter card meta tags
- Canonical URLs
- Internal linking strategy
- `sitemap.xml` generation (`@astrojs/sitemap`)
- `robots.txt`
- Schema markup (JSON-LD) when useful (Article, BreadcrumbList, WebSite)
- Clean, readable URLs
- Content discoverability through logical site structure

## Performance Targets

Prioritize on every build:
- Fast LCP (Largest Contentful Paint)
- Low CLS (Cumulative Layout Shift)
- Minimal JavaScript — use Astro's zero-JS-by-default approach
- Image optimization with Astro's `<Image />` component
- Lazy loading for below-fold images
- Smart hydration — only hydrate interactive islands
- Island architecture (`client:load`, `client:idle`, `client:visible`) used sparingly
- Clean bundle output
- Lighthouse score excellence (aim for 90+ across all categories)

## Quality Rules

Never:
- Generate bloated, redundant code
- Overuse JavaScript when Astro can render it statically
- Sacrifice performance for flashy design
- Use weak or generic layouts that undermine credibility
- Ignore SEO fundamentals on any page
- Ignore mobile responsiveness
- Produce messy or unnavigable architecture

## Responsive Rules

Every site must be excellent on:
- Mobile (first priority — design mobile-up)
- Tablet
- Laptop
- Large desktop

Never treat mobile as an afterthought. Test breakpoints explicitly.
