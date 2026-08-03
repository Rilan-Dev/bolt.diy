# Bolt DIY Production Deployment Design

**Date:** 2026-08-04
**Status:** Approved for implementation planning

## Objective

Deploy the upstream `stable` version of Bolt DIY as a public production application at `https://bolt.ostechnologies.in`, using Cloudflare Pages and Cloudflare credentials already stored in NexaHub. Visitors will supply their own AI-provider credentials through Bolt's existing settings. No shared AI-provider key will be deployed.

## Scope and constraints

- Keep Bolt DIY application code unchanged from `stackblitz-labs/bolt.diy` `stable`.
- Make configuration and deployment changes only where required.
- Do not print, commit, copy into the repository, or persist Cloudflare credentials in shell history.
- Use the configured Cloudflare zone `ostechnologies.in` and hostname `bolt.ostechnologies.in`.
- The application is intentionally public; Cloudflare Access authentication is out of scope.
- Use direct Cloudflare Pages upload rather than Git-connected builds or an always-on Docker host.

## Repository ownership and provenance

GitHub treats `stackblitz-labs/bolt.diy` as part of the `stackblitz/bolt.new` fork network. The `Rilan-Dev` account already owns `Rilan-Dev/bolt.new`, so GitHub cannot create a second true fork in that network. The safe approved alternative is a standalone public repository created from the official Bolt DIY GitHub template:

- `origin`: `https://github.com/Rilan-Dev/bolt.diy.git`
- `upstream`: `https://github.com/stackblitz-labs/bolt.diy.git`
- deployment branch: `stable`

The local clone is `/docker/bolt.diy`. Its `stable` source tree must match `upstream/stable` before deployment. Upstream synchronization remains explicit through the `upstream` remote.

## Architecture

Cloudflare Pages will host the existing Remix/Vite client bundle and Pages Functions output on Cloudflare's edge. The repository's existing `wrangler.toml` and build scripts define `build/client` as the Pages output. A direct upload will create an immutable deployment under the dedicated Pages project `bolt-diy-rilan`, after which the custom domain will be associated through the Cloudflare Pages API.

The public application will not receive a shared OpenAI, Anthropic, or other model-provider secret. Each visitor configures a provider using Bolt's existing settings flow, keeping usage and billing tied to that visitor's credentials.

## Credential handling

NexaHub's `/docker/NexaHub/apps/server/.env` contains a configured Cloudflare global API key, account email, and zone ID. It does not contain a Cloudflare API token.

For this one-time deployment:

1. Read the required values into process-local environment variables without echoing them.
2. Map NexaHub's `CLOUDFLARE_API_EMAIL` value to Wrangler's expected `CLOUDFLARE_EMAIL` variable.
3. Pass `CLOUDFLARE_API_KEY` and `CLOUDFLARE_EMAIL` only to authentication and deployment processes.
4. Do not create a Bolt `.env` containing Cloudflare credentials.

Cloudflare recommends scoped API tokens over global API keys. Token migration is a follow-up security improvement, not a prerequisite for this authorized deployment.

## Build and verification

Use the package-manager version declared by the repository and the committed lockfile. Before deployment:

1. Confirm the working tree has no application-code changes and verify that the application source, excluding this deployment specification, matches `upstream/stable`.
2. Install dependencies with the frozen lockfile.
3. Run type checking, linting, automated tests, and the production build.
4. Treat failures as blockers and diagnose them before deploying.
5. Inspect the generated Pages artifact and confirm no `.env` file or Cloudflare credential is included.

After deployment:

1. Verify the Pages deployment URL over HTTPS.
2. Associate `bolt.ostechnologies.in` with the Pages project before relying on DNS alone.
3. Wait for the custom domain and managed certificate to become active.
4. Verify HTTPS, the application shell, static assets, and a representative application route.
5. Confirm the site does not expose a shared provider credential.

## Production operations

- Cloudflare provides managed TLS, CDN delivery, and baseline DDoS protection.
- Pages deployment history provides an immutable rollback target.
- Record the Pages project name and final deployment URL in the handoff.
- Do not enable Cloudflare Access because public availability was explicitly selected.
- Avoid unrelated DNS, zone, repository, or application changes.

## Error handling and rollback

- If Cloudflare authentication fails, stop without changing DNS and report the exact non-secret error.
- If `bolt-diy-rilan` already belongs to an unrelated application, stop rather than overwriting it.
- If build or test verification fails, do not deploy an unverified artifact.
- If domain activation fails, retain the working `pages.dev` deployment, inspect the Pages-domain status and DNS conflict, and do not delete unrelated records.
- If the custom-domain deployment is unhealthy, detach only the newly added domain association or roll back to the prior Pages deployment; preserve all unrelated Cloudflare resources.

## Success criteria

- `Rilan-Dev/bolt.diy` exists and the local deployment source, excluding this deployment specification, matches official `stable` application code.
- Required checks and the production build pass, or any upstream failure is explicitly diagnosed and resolved without unapproved code changes.
- A Cloudflare Pages production deployment is active.
- `https://bolt.ostechnologies.in` resolves with a valid certificate and serves Bolt DIY.
- No Cloudflare or shared AI-provider secret is committed, embedded in build artifacts, or disclosed in output.
