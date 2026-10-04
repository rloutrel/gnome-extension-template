# Template synchronization

This template is consumed by derived projects (e.g.
`gnome-extension-ai-status-monitor`). Because `setup-project.sh` renames the
extension directory, UUID and schema, consumers cannot `git merge` template
updates. Synchronization is done at file level, driven by tags and a
per-consumer manifest.

## Tag convention

- Sync-worthy template states are tagged `template/vMAJOR.MINOR.PATCH`
  (e.g. `template/v1.0.0`).
- The `template/` prefix is required: `release.yml` triggers on `v*` tags
  and must never fire for template tags.
- Bump the tag when a change is relevant to consumers: pipeline/CI fixes,
  GJS practice updates, logger/indicator/prefs refactors, test tooling.
  Do not tag cosmetic-only commits.
- Every tag has an entry in `TEMPLATE_CHANGELOG.md` so consumers can see
  what a tag carries without diffing.

## Consumer manifest

Each derived project commits a `.template-sync.json` at its root:

```json
{
  "source": "rloutrel/gnome-extension-template",
  "ref": "template/v1.0.0",
  "tracked": [
    "extension.js",
    "indicator.js",
    "logger.js",
    "prefs.js",
    "stylesheet.css",
    "test/validateMetadata.js",
    ".github/workflows/test.yml",
    ".github/workflows/release.yml",
    ".github/workflows/codeql.yml",
    "eslint.config.js",
    "sonar-project.properties",
    "po/POTFILES",
    "AGENTS.md"
  ],
  "ignored": [
    "sampleModule.js",
    "test/sampleModule.test.js",
    "metadata.json",
    "README.md",
    "setup-project.sh"
  ]
}
```

- `ref`: the last template tag the consumer synced against. This makes the
  sync diff incremental (compare `ref` → latest tag) instead of a full re-diff.
- `tracked`: template-owned files. Changes here are candidates for
  forward-sync (template → consumer) and the trigger for backport
  suggestions (consumer → template).
- `ignored`: project-owned files never synced (identity, docs, sample code).

Consumers map tracked paths to their own layout (extension dir is renamed by
`setup-project.sh`, e.g. `extension.js` →
`ai-status-monitor@rloutrel.github.com/extension.js`).

## Sync procedures

### Template → consumer

1. Read the consumer's `.template-sync.json` (`ref`, `tracked`).
2. Fetch the template at `ref` and at the latest `template/v*` tag.
3. Diff the `tracked` files between the two tags.
4. Apply relevant hunks to the consumer's mapped paths, adapting the
   UUID/schema/paths as needed.
5. Update `ref` in the manifest and commit.

### Consumer → template (backport)

When a consumer change touches a `tracked` file and the fix is generic
(not project-specific), suggest a backport PR to this repository. Once
merged, the fix ships under the next `template/v*` tag so other consumers
pick it up on their next forward-sync.
