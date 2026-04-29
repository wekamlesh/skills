# Deployment Reference

## Cloudflare Pages (Default)

For fully static sites (default): no adapter needed. Cloudflare Pages serves the `dist/` folder directly.

Build settings in Cloudflare dashboard:
- Build command: `npm run build`
- Output directory: `dist`

For SSR/hybrid output:
```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  output: 'server', // or 'hybrid'
  adapter: cloudflare(),
});
```

Install adapter:
```bash
npx astro add cloudflare
```

## GitHub Pages

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://<username>.github.io',
  base: '/<repo-name>', // omit for user/org sites
});
```

GitHub Actions workflow (`.github/workflows/deploy.yml`):
```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

## VPS (Node.js SSR)

Install adapter:
```bash
npx astro add node
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),
});
```

Run the server:
```bash
node ./dist/server/entry.mjs
```

Manage with PM2:
```bash
npm install -g pm2
pm2 start dist/server/entry.mjs --name my-astro-site
pm2 save
pm2 startup
```

Pair with Nginx as a reverse proxy:
```nginx
server {
  listen 80;
  server_name yourdomain.com;

  location / {
    proxy_pass http://localhost:4321;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
```

Use Certbot to add HTTPS:
```bash
certbot --nginx -d yourdomain.com
```
