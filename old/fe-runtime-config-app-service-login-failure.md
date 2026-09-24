# Finding: QA login redirects to localhost:3000 after `emerie` deploy

**Date:** 2026-08-14
**Repo:** `altus-ai-assistant-fe`, branch `emerie`
**Environment:** QA (Azure App Service, `assistant.qa.altusnova.ai`)
**Status:** diagnosed, not fixed

## Symptom

The `emerie` branch deployed to QA fails at login: MSAL sends the browser to `http://localhost:3000` as the redirect URI instead of `https://assistant.qa.altusnova.ai`.

## Root cause

Commit `766fd8a` ("Load MSAL and API URL from Helm runtime-config.js") moved MSAL/API config from build-time Vite env vars to a runtime file, so AKS can change it via GitOps without an image rebuild.

`index.html` loads `/runtime-config.js` as a classic script before the module bundle, and `src/libs/shared/runtimeConfig.ts` spreads the resulting global **after** the Vite values:

```ts
return {
  ...fromVite,
  ...(typeof window !== 'undefined' ? window.__RUNTIME_CONFIG__ : undefined),
};
```

The runtime file therefore always wins, and every key is present in it — nothing from `import.meta.env` survives.

The file shipped in `dist/` is the placeholder `public/runtime-config.js`, whose own comment reads *"Overwritten in AKS by Helm ConfigMap"*:

```js
msalClientId:    '6316a0a6-d518-4fb5-942b-b42bd682ee6a',
msalRedirectUri: 'http://localhost:3000',
msalScopes:      'api://501b03ab-30ec-46da-8e94-bf983fb23dd9/user-access',
apiBaseUri:      'http://localhost:8080',
```

In AKS, `charts/altus-ai-assistant-fe/templates/deployment.yaml` mounts the `-runtime` ConfigMap over `/usr/share/nginx/html/runtime-config.js`, so the placeholder is never seen. **App Service performs no such mount**, so QA runs on the placeholder and the `npm run build:qa` values from `.env.qa` are silently discarded.

## Blast radius beyond the login redirect

Currently masked by the login failure, but equally wrong on QA:

| Value | Placeholder (in use) | `.env.qa` (intended) |
|---|---|---|
| `msalRedirectUri` | `http://localhost:3000` | `https://assistant.qa.altusnova.ai` |
| `apiBaseUri` | `http://localhost:8080` | `https://assistant.api.qa.altusnova.ai` |
| `msalClientId` | `6316a0a6-…` (emerie app reg) | `d341df90-…` |
| `msalScopes` | `api://501b03ab-…/user-access` | `api://4279cecc-…/user-access` |

`apiBaseUri` feeds both `axiosInstance` and the SignalR hub URL, so chat streaming would fail too.

## Options

1. **Quick unblock, no rebuild** — hand-edit `runtime-config.js` in the App Service `wwwroot` (Kudu/SSH) with the `.env.qa` values. `web.config` sets no cache header for it, so hard-refresh; IIS may still serve a heuristically cached copy.
2. **Pipeline step** — after `npm run build:$(Env)`, overwrite `dist/runtime-config.js` from the env values. App-Service-only change, AKS path untouched.
3. **Recommended** — generate `runtime-config.js` at build time from the mode's `.env` via a small Vite plugin (dev-server middleware + `emitFile` in `generateBundle`), and delete `public/runtime-config.js`. `.env.qa` / `.env.production` become the single source of truth for static hosting; Helm still mounts over the file in AKS, so the emerie/AKS path is unchanged. Add a `no-store` header for `/runtime-config.js` in `web.config` to match what `nginx.conf` already does for the container.

## Related

Whichever client ID QA ends up on, `https://assistant.qa.altusnova.ai/auth-redirect.html` must be registered as a SPA redirect URI on that app registration — `6316a0a6-…` likely carries only the emerie hosts. See `entra-app-registrations.md`.