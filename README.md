# custom-frappe

Builds a custom [Frappe](https://frappe.io) Docker image with a fixed set of apps already installed, and publishes it to the GitHub Container Registry (GHCR).

The build uses the official [`frappe_docker`](https://github.com/frappe/frappe_docker) layered `Containerfile`. This repo only provides the settings i.e which Frappe version to use, which apps to bundle, and what to call the image.

## Included apps

| App | Repository |
| --- | --- |
| ERPNext | [frappe/erpnext](https://github.com/frappe/erpnext) |
| HRMS | [frappe/hrms](https://github.com/frappe/hrms) |
| Insights | [frappe/insights](https://github.com/frappe/insights) |
| Print Designer | [frappe/print_designer](https://github.com/frappe/print_designer) |
| POS Awesome (v15) | [defendicon/POS-Awesome-V15](https://github.com/defendicon/POS-Awesome-V15) |
| Builder | [frappe/builder](https://github.com/frappe/builder) |

The branch or tag used for each app is set in [`.env`](.env).

## Repository layout

| File | Purpose |
| --- | --- |
| [`.env`](.env) | Build settings: the `frappe_docker` version, the Frappe version, each app's branch, and the image name and tag. |
| [`apps.json.template`](apps.json.template) | The list of apps to install. `${...}` placeholders are filled in from `.env` at build time. |
| [`.github/workflows/build-image.yml`](.github/workflows/build-image.yml) | GitHub Actions workflow that builds and pushes the image. |

## How a build works

1. The workflow runs when a push to `main` changes `.env`, `apps.json.template` or the workflow file. You can also start it manually from the **Actions** tab (**Build Custom Frappe Image** → **Run workflow**).
2. It reads the settings from `.env` and checks out `frappe_docker` at `FRAPPE_DOCKER_REF`.
3. It fills in `apps.json.template` with `envsubst` to make `apps.json`, and prints the result in the build log.
4. It builds `frappe_docker/images/layered/Containerfile` for `linux/amd64`, passing `apps.json` as a build secret.
5. It pushes the image to GHCR with two tags:
   - `ghcr.io/<owner>/<IMAGE_NAME>:<IMAGE_TAG>`
   - `ghcr.io/<owner>/<IMAGE_NAME>:sha-<commit>`, which points to one exact build

Docker layers are cached between runs with the GitHub Actions cache. A hash of `apps.json` is passed as `CACHE_BUST`, so changing any app or branch forces the apps to be fetched again rather than reused from the cache.

## Configuration

All settings are in `.env`:

| Variable | Description |
| --- | --- |
| `FRAPPE_DOCKER_REF` | Branch, tag or commit of `frappe_docker` whose `Containerfile` is used. |
| `FRAPPE_PATH` | Git URL of the Frappe framework repository. |
| `FRAPPE_BRANCH` | Frappe branch or tag, for example `version-15`. |
| `ERPNEXT_BRANCH` | ERPNext branch or tag. |
| `HRMS_BRANCH` | HRMS branch or tag. |
| `INSIGHTS_BRANCH` | Insights branch or tag. |
| `PRINT_DESIGNER_BRANCH` | Print Designer branch or tag. |
| `POS_AWESOME_V15_BRANCH` | POS Awesome branch or tag. |
| `BUILDER_BRANCH` | Builder branch or tag. |
| `IMAGE_NAME` | Name of the image on GHCR. |
| `IMAGE_TAG` | Tag given to each new build, for example `latest` or `v15`. |

Keep the app branches compatible with `FRAPPE_BRANCH`. For example, use `version-15` apps with Frappe `version-15`.

> [!NOTE]
> `.env` is committed to the repository and read by the workflow. Use it only for build settings. Never put passwords, tokens or other secrets in it.

## Common tasks

### Update a version

Change the relevant `*_BRANCH` value (or `FRAPPE_DOCKER_REF`) in `.env`, then commit and push to `main`. A new build starts on its own.

### Add an app

1. Add an entry to `apps.json.template`:

   ```json
   { "url": "https://github.com/<org>/<app>", "branch": "${MY_APP_BRANCH}" }
   ```

2. Add the matching variable to `.env`:

   ```sh
   MY_APP_BRANCH=version-15
   ```

3. Commit and push to `main`.

To remove an app, delete its entry from `apps.json.template` and its variable from `.env`.

### Rebuild without changes

Run the workflow manually from the **Actions** tab.

## Using the image

Pull it:

```sh
docker pull ghcr.io/scariest/<IMAGE_NAME>:<IMAGE_TAG>
```

To run it, follow the [`frappe_docker` documentation](https://github.com/frappe/frappe_docker/tree/main/docs). Set these variables in its `.env` (or in `pwd.yml`) so that it uses this image instead of the stock one:

```sh
CUSTOM_IMAGE=ghcr.io/scariest/<IMAGE_NAME>
CUSTOM_TAG=<IMAGE_TAG>
PULL_POLICY=always
```

After creating a site, install the apps you need on it, for example:

```sh
bench --site <site-name> install-app erpnext hrms insights print_designer posawesome builder
```

Check each app's own documentation for its exact module name before installing.

> [!TIP]
> For production, pin the `sha-<commit>` tag instead of a moving tag like `latest`. That way a new build won't change a running deployment without warning.

### Private images

New GHCR packages are private by default. To pull a private image, either log in first with a token that has `read:packages`:

```sh
echo <token> | docker login ghcr.io -u <github-username> --password-stdin
```

or make the package public under **Package settings** on GitHub.
