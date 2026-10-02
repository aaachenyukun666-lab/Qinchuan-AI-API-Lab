# Demo

This directory is a place for runnable, self-contained demos. The core app is
the single-page `index.html` at the project root, so there is no separate demo
build to install:

```bash
npm run serve          # then open http://127.0.0.1:4173/
```

To exercise the engine against a fake relay without spending real quota, run the
integration tests — they spin up an in-process OpenAI-compatible mock server:

```bash
npm test
```

For a real, small, one-request smoke check against your own key:

```bash
ARM_BASE_URL=https://api.deepseek.com/v1 \
ARM_API_KEY=sk-... \
ARM_MODEL=deepseek-chat \
npm run test:live
```

No additional demo assets are required for v0.1.
