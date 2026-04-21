# [Backstage](https://backstage.io)

This is your newly scaffolded Backstage App, Good Luck!

To start the app, run:

```sh
yarn install
yarn start
```

Local GitHub SSO (development) setup
-----------------------------------

1. Create a GitHub OAuth App (Settings → Developer settings → OAuth Apps) with a callback URL pointing to `http://localhost:7007/api/auth/github/handler/frame`.
2. Add the `AUTH_GITHUB_CLIENT_ID` and `AUTH_GITHUB_CLIENT_SECRET` to your environment.

Run the backend with the required env vars:

```sh
cd packages/backend
APP_BASE_URL=http://localhost:7007 \
AUTH_GITHUB_CLIENT_ID=your_client_id \
AUTH_GITHUB_CLIENT_SECRET=your_client_secret \
../../node_modules/.bin/backstage-cli package start
```

To deploy to Kubernetes the deployment expects a Kubernetes secret named `backstage-auth` with keys `github_client_id` and `github_client_secret`. The GitHub Actions workflow now creates this secret from repository secrets `AUTH_GITHUB_CLIENT_ID` and `AUTH_GITHUB_CLIENT_SECRET` before applying the ArgoCD application.

If you want to enable Guest sign-in during local testing, ensure `auth.providers.guest` and `signIn.providers` include `guest` in `app-config.yaml` (they are enabled by default in this repo).
