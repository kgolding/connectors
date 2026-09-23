# D-Sub Connector Pinout

An interactive drawing of D-Sub connectors with numbered pins, for working out which pin is which when wiring or checking a connector.

**Try it:** https://kgolding.github.io/connectors/

## Features

- **Connector types:** DE-9, DE-15 HD (VGA), DA-15 and DB-25, drawn to standard proportions.
- **Male or female:** pins or sockets.
- **Back or front view:** the wiring (solder cup) side or the mating face. The drawing mirrors correctly for each combination of gender and view.
- **Rotation:** turn the connector in 90° steps with the corner buttons or the ← / → arrow keys. Pin numbers always stay upright and readable.
- **Pin highlighting:** list the pins in use (for example `1, 3, 5-7`) and they are highlighted, with the unused pins faded.

Pin 1 is outlined in red so you can always see which way round the connector is.

## Developing

Built with [SvelteKit](https://svelte.dev/docs/kit) and Svelte 5. Install dependencies and start the development server:

```sh
npm install
npm run dev
```

Other scripts:

```sh
npm run check    # type-check
npm run build    # static build into build/
npm run preview  # serve the production build
```

## Deployment

The site is a fully static build (`@sveltejs/adapter-static`) hosted on GitHub Pages. The [build and deploy workflow](.github/workflows/deploy.yml) checks and builds every push and pull request, and deploys pushes to `main`. It sets `BASE_PATH` to the repository name so the app works under `/connectors/`.
