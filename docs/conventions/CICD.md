<!-- AUTO-SYNCED from agents KB: concepts/CICD.md @ b285312.
     Do NOT edit here — edit the source in ~/projects/agents and re-run scripts/sync-conventions.sh. -->

# CI/CD

CI runs on **GitHub Actions**. The pipeline builds a container image, pushes it to
**ghcr.io**, and then hands off to GitOps by bumping the image tag in the deployment
repo — ArgoCD does the actual rollout (see [DEPLOYMENT.md](DEPLOYMENT.md)).

## Typical workflows per product repo

- `ci.yml` — tests + lint on push/PR (backend: Maven + JUnit; frontend: Biome/ESLint
  + Vitest). Gate merges on this.
- `docker.yml` — build & push the image, then update the deployment repo.
- Optional: `test.yml`, `issue-redirect.yml`, etc.

## The build-and-push → tag-bump loop

`docker.yml` is the important one. Shape (from `cosy-domain-provider-backend`):

1. **Build & push** on push to `main` (and on `v*.*.*` tags):
   - Log in to `ghcr.io` with `GITHUB_TOKEN`, set up Buildx.
   - `docker/metadata-action` computes tags:
     - branch push → `main-<short-sha>` **and** `staging-latest`
     - semver tag → `<version>` **and** `prod-latest`
   - `docker/build-push-action` with `cache-from/to: type=gha`.
2. **Update staging** (only on branch push):
   - Clone the `*-deployment` repo with write access — preferably a **deploy key**
     (see below), legacy repos use a `DEPLOY_REPO_TOKEN` PAT.
   - `cd overlays/staging && kustomize edit set image <img>:main-<short-sha>`
   - Commit + push. ArgoCD auto-syncs staging.
3. **Update prod** (only on tag push): same idea against `overlays/prod` with the
   semver/`prod-latest` tag.

So: **push to `main` ships staging; cutting a `vX.Y.Z` tag ships prod.** Projects
without a staging/prod split (e.g. Strata) use a single `main` overlay and bump that.

## Image tag conventions

| Trigger              | Tags produced                        | Deploys to |
|----------------------|--------------------------------------|------------|
| push to `main`       | `main-<sha>`, `staging-latest`       | staging    |
| tag `vX.Y.Z`         | `X.Y.Z`, `prod-latest`               | prod       |

The deployment overlay pins the immutable `main-<sha>` / `X.Y.Z` tag (not the
`*-latest` alias) so every environment is reproducible and rollbacks are a git revert.

## Write access to the deployment repo — use a deploy key

The tag-bump job needs push rights on the `*-deployment` repo. Prefer a **repo-scoped
SSH deploy key** over a PAT: a deploy key grants write to exactly *one* repo, doesn't
act as a user, doesn't expire, and revoking it can't break anything else. (Wharf is
the reference implementation; older repos like cosy-domain-provider still use a
`DEPLOY_REPO_TOKEN` fine-grained PAT — migrate opportunistically.)

Setup, once per deployment repo:

```bash
ssh-keygen -t ed25519 -N "" -C "<project>-ci-tag-bump" -f deploy_key
gh repo deploy-key add deploy_key.pub -R <owner>/<project>-deployment \
  --allow-write --title "<project> CI tag bump"
# same secret name in every product repo that bumps this deployment repo:
gh secret set DEPLOY_REPO_SSH_KEY -R <owner>/<project>-backend  < deploy_key
gh secret set DEPLOY_REPO_SSH_KEY -R <owner>/<project>-frontend < deploy_key
rm -P deploy_key deploy_key.pub   # private key lives ONLY in Actions secrets
```

In `docker.yml`, the bump job checks the deployment repo out with the key —
`actions/checkout` configures the remote so the later `git push` goes over SSH
automatically:

```yaml
- name: Check out <project>-deployment
  uses: actions/checkout@v4
  with:
    repository: <owner>/<project>-deployment
    ssh-key: ${{ secrets.DEPLOY_REPO_SSH_KEY }}
```

One key is shared by all product repos of the *same* project (same secret name), but
never reuse a key across projects — one deploy key per deployment repo, so a leak or
revocation stays contained.

**DO:**
- Keep secrets (`GITHUB_TOKEN`, `DEPLOY_REPO_SSH_KEY` / legacy `DEPLOY_REPO_TOKEN`)
  in GitHub Actions secrets; never in the workflow file.
- Use the GHA build cache (`type=gha`) to keep image builds fast.
- Let CI own the image tag in the deployment repo — don't race it with manual edits.
- Commit the deployment change as `github-actions[bot]` with a `ci: update <c> image
  to <tag>` message.

**DON'T:**
- Deploy by building images locally and pushing tags by hand.
- Point overlays at moving tags (`staging-latest`/`prod-latest`) for the pinned
  image — use the sha/semver tag so state is reproducible.
- Put deployment (kubectl) steps in the product-repo CI — CI only bumps Git.

## Legacy exception

**Medals** is deployed via **Watchtower** (auto-pull of a moving image tag), not
ArgoCD/Kustomize. It predates the GitOps standard — treat it as the odd one out and
don't copy its model to new projects.
