# GeoConvert

GeoConvert is a browser-based GeoJSON coordinate reference system (CRS)
converter. Paste or edit GeoJSON, select the source and target CRS, and
convert the coordinates directly in the browser. The result can be reviewed
on a map and copied for use in another application.

## Live application

The production application is available at
[geojson.bikkel.pro](https://geojson.bikkel.pro).

## Features

- Convert GeoJSON `FeatureCollection`, `Feature`, and geometry objects.
- Detect a CRS from the input when it is included in the GeoJSON.
- Select from common CRS definitions and all WGS 84 UTM zones.
- Preview GeoJSON on an interactive MapLibre map.
- Copy converted GeoJSON to the clipboard.
- English and Dutch interface translations.
- Light and dark color modes.

## Technology

- [Nuxt](https://nuxt.com/) 4
- [Vue](https://vuejs.org/)
- [Nuxt UI](https://ui.nuxt.com/)
- [MapLibre GL JS](https://maplibre.org/)
- [terrio-js](https://www.npmjs.com/package/terrio-js) for coordinate
  transformations
- [pnpm](https://pnpm.io/) for package management

## Local development

Install dependencies:

```bash
pnpm install
```

Start the development server:

```bash
pnpm dev
```

The application is then available at
[`http://localhost:3000`](http://localhost:3000).

## Production build

Build the Nuxt production bundle:

```bash
pnpm build
```

Preview the production build locally:

```bash
pnpm preview
```

Useful checks:

```bash
pnpm lint
pnpm typecheck
```

## Docker deployment

The repository includes a multi-stage [Dockerfile](./Dockerfile) and a
production [Docker Compose configuration](./docker-compose.yml).

Build and start the application:

```bash
docker compose up --build -d
```

The container listens on port `3000` and is published on port `3000` on the
host. On a server, put a reverse proxy such as Nginx, Caddy, or Traefik in
front of it to terminate HTTPS and route
`geojson.bikkel.pro` to `http://127.0.0.1:3000`.

View the container status and logs:

```bash
docker compose ps
docker compose logs -f
```

Stop the application:

```bash
docker compose down
```
