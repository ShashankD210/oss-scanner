# Wenjun portfolio website

This repository is a static Vite-based portfolio deployed to GitHub Pages. The
build uses Node.js 24 and the committed npm lockfile. There is no application
backend or authenticated server in this checkout.

## Inputs and boundaries

Review URL fragments/query parameters, DOM operations, external links and media,
content rendering, asset-path validation, and build tooling. Focus on script/DOM
injection, unsafe URL schemes, prototype mutation from externally controlled
objects, and paths escaping the intended public/build root. Do not assume
repository-authored portfolio text is attacker-editable; explain a realistic
untrusted input path for each finding. Build dependencies are in scope when
reachable through this project's use of them.

## Offline checks

The checkout is `/src`; production output is `/src/dist`. Run:

```sh
npm run check:content
npm test
npm run check:images
npm run build
```

Serve the existing output with `python3 -m http.server 8000 --directory dist`.
`npm run check` also performs an online registry audit, so run the individual
offline commands above during scanning. Dependencies are installed during the
Docker build. Production hosting configuration and external sites are not in
this environment; do not probe the live website or deploy changes.

## Severity and reporting

Prioritize browser execution from an actual remote input, secret leakage, or
arbitrary file access through build input. Include origin, user interaction,
browser prerequisites, a local reproducer, and a minimal patch. Lack of a
security header alone is not proof of an exploitable application vulnerability.
Send findings privately.
