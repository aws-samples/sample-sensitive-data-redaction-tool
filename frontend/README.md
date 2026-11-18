# React + TypeScript + Vite

This template provides a minimal setup to get React working in Vite with HMR and some Oxlint rules.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Oxc](https://oxc.rs)
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/)

## React Compiler

The React Compiler is not enabled on this template because of its impact on dev & build performances. To add it, see [this documentation](https://react.dev/learn/react-compiler/installation).

## Expanding the Oxlint configuration

If you are developing a production application, we recommend enabling type-aware lint rules by installing `oxlint-tsgolint` and editing `.oxlintrc.json`:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["react", "typescript", "oxc"],
  "options": {
    "typeAware": true
  },
  "rules": {
    "react/rules-of-hooks": "error",
    "react/only-export-components": ["warn", { "allowConstantExport": true }]
  }
}
```

See the [Oxlint rules documentation](https://oxc.rs/docs/guide/usage/linter/rules) for the full list of rules and categories.

## Deployment and backend API configuration

The frontend and backend are **two independently deployable components**. The frontend build output is a set of static assets (`dist/`) that can be hosted on any static hosting solution (S3 with CloudFront is the default assumption). The only coupling between them is the backend API base URL, `VITE_API_BASE_URL`, injected at build time, plus the Cognito settings below. The frontend source does **not** import from `backend/` and shares no build step, so backend implementation changes never require a frontend rebuild, and vice versa.

### Environment variables

| Variable | Description |
|---|---|
| `VITE_API_BASE_URL` | Base URL of the deployed backend API (API Gateway). **Must be `https://`**; a non-HTTPS value is rejected when the API client is constructed (see `toHttpsBaseUrl` in `src/api/client.ts`). |
| `VITE_COGNITO_REGION` | Region of the Cognito user pool used for sign-in. Required; the app refuses to start without it. |
| `VITE_COGNITO_CLIENT_ID` | User pool client ID (the stack output `UserPoolClientId`). Required; the app refuses to start without it. |

Per-environment files (Vite convention; only `VITE_`-prefixed variables reach the bundle):

- `.env.example`: committed, documents the required variables with placeholders and **no real URLs**.
- `.env.development`: defaults for development mode (`vite dev`), pointing at a local or port-forwarded backend.
- `.env.production`: production-mode config. It **does not hardcode a real backend URL**; the variable is left empty and must be supplied by the deployment environment at build time.
- `.env.local` / `.env.*.local`: machine-level overrides, ignored by `.gitignore`; put real URLs here. In development mode `.env.development` takes precedence over `.env.local`, so use `.env.development.local` to override it.

### Build the frontend against a deployed backend

```bash
# Production build: inject the deployed backend's HTTPS base URL and the Cognito settings at build time
VITE_API_BASE_URL=https://your-api-id.execute-api.your-region.amazonaws.com \
VITE_COGNITO_REGION=us-west-2 VITE_COGNITO_CLIENT_ID=your-user-pool-client-id npm run build

# Output is in frontend/dist/; deploy it to static hosting (S3 with CloudFront, etc.)
```

A `VITE_API_BASE_URL` provided by the shell or CI takes precedence over the value in `.env.production`.

### Missing variable in production (no fallback, fail fast)

In a production build (`import.meta.env.PROD`), if `VITE_API_BASE_URL` is missing or empty, `resolveApiBaseUrl` (`src/api/apiConfig.ts`) **throws** instead of falling back to `https://localhost:4000` or any HTTP address. This keeps a misconfigured production deployment from silently pointing at the wrong backend. The convenient `localhost` fallback only applies in development mode.
