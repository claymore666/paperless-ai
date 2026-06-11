# Paperless-AI Development

## Repo Layout

- **Upstream**: `clusterzx/paperless-ai` (origin)
- **Fork**: `claymore666/paperless-ai` (remote: fork)
- **Local checkout**: `/home/claude/claude.ai/paperless-ai/` (pve1)
- **Bug tracker**: `BUGS.md` (gitignored, local only)

## Environments

| Environment | Host | Paperless-ngx | Paperless-AI | Credentials |
|-------------|------|---------------|--------------|-------------|
| **Production** | VM 200 (pve2, 192.168.0.130) | http://paperless.local | http://paperless-ai.local | (see Paperless DB) |
| **QA** | gpu1 (192.168.0.141) | http://192.168.0.141:18000 | http://192.168.0.141:13000 | admin/admin |

### QA Environment

**Location**: `gpu1:/opt/paperless-qa/`

```
docker-compose.yml          # 4 containers: web, db, redis, ai
seed/documents/             # 50 prod PDFs (baseline source)
snapshots/baseline/         # pg dump + media + appdata + ai-data
data/                       # live runtime data
scripts/
  snapshot.sh               # save/restore/list snapshots
  reset.sh                  # wipe-to-zero reset (legacy)
```

**Snapshot operations** (run via Ansible since claude user needs sudo docker):
```bash
# Save current state
ssh claude@192.168.0.34 'cd /opt/ansible && sudo -u ansible ansible gpu1 -m shell -a "/opt/paperless-qa/scripts/snapshot.sh save <name>" -b'

# Restore to saved state (~11 seconds)
ssh claude@192.168.0.34 'cd /opt/ansible && sudo -u ansible ansible gpu1 -m shell -a "/opt/paperless-qa/scripts/snapshot.sh restore baseline" -b'

# List snapshots
ssh claude@192.168.0.34 'cd /opt/ansible && sudo -u ansible ansible gpu1 -m shell -a "/opt/paperless-qa/scripts/snapshot.sh list" -b'
```

**Deploy patched code to QA**:
```bash
scp server.js claude@192.168.0.141:/tmp/
ssh claude@192.168.0.141 'sudo docker cp /tmp/server.js paperless-qa-ai:/app/server.js && sudo docker restart paperless-qa-ai'
```

**QA API token**: `457b2039df9c358796a541cff95c26ef3c89d22f`

## Branch Strategy

| Branch | Purpose | Tracks |
|--------|---------|--------|
| `main` (fork) | **Single source of truth and deployment branch.** Upstream v3.0.9 + all our fixes + cherry-picked community PRs | tagged releases, e.g. `v3.0.9-fixes` |
| `fix/*` | Short-lived fix branches | branched from fork main, deleted after merge |

**Upstream is officially unmaintained** (README notice since Apr 2026; our PRs #920–#924 were auto-closed by the stale-bot). The old `prod` branch and all merged `fix/*` branches were merged into fork `main` and deleted (June 2026). Releases are tagged on fork main: [`v3.0.9-fixes`](https://github.com/claymore666/paperless-ai/releases/tag/v3.0.9-fixes).

Remaining unmerged branches on the fork are **stale clusterzx work kept for reference only** — do not merge: `deepseek-r1` (superseded by v3.0.9 reasoning support), `ocr-server-feature` (abandoned WIP), `dev-rag` (old RAG state), `fix-custom-service` (version-bump only).

## Workflow: New Fix

1. Branch from fork main: `git checkout main && git checkout -b fix/<name>`
2. Make changes, test on QA
3. Commit (no Claude attribution):
   ```bash
   git add <files>
   git commit -m "fix: description

   Fixes #NNN"
   ```
4. Push to fork: `git push fork fix/<name>`
5. Merge into fork main: `git checkout main && git merge fix/<name> && git push fork main`
6. Delete the fix branch: `git push fork --delete fix/<name>`
7. Tag a release when a deployable set of fixes has accumulated (e.g. `v3.0.9-fixes2`), update the README fork-status section
8. (Optional) PR upstream — note the stale-bot auto-closes PRs after inactivity, so don't expect merges

**Important**: Always show PR/comment text to user for approval before posting.

## Workflow: Sync from upstream (if it ever revives)

```bash
git fetch origin
git checkout main && git merge origin/main
git push fork main
```

## Git Config

```
user.name = claymore666
user.email = christian.kamien@gmail.com
```

`gh` authenticated as `claymore666` on pve1. Fork remote: `fork` → `claymore666/paperless-ai`.

## Merged Fixes (all in fork main since June 2026)

| Fix | Issue | Upstream PR (closed unmerged) |
|-----|-------|-------------------------------|
| Enforce `RESTRICT_TO_EXISTING_DOCUMENT_TYPES` | #834 | #920 |
| Reconcile stale documents from AI database | #471 | #921 |
| Conservative token estimation for `num_ctx` | #913, #745 | #922 |
| Normalize API URL (double `/api/` paths) | #880 | #923 |
| `config.responseTokens` in customService | #802 | #924 |
| Repair JSON parsing in manual service | — | — |
| Cherry-picked community PRs | — | #893, #900, #902–#907, #915 |

## Key Source Files

| File | Purpose |
|------|---------|
| `server.js` | Main app — `scanInitial()`, `scanDocuments()`, `processDocument()`, `buildUpdateData()` |
| `services/paperlessService.js` | Paperless-ngx API client — `getOrCreateDocumentType()`, `processTags()`, `getAllDocuments()` |
| `services/ollamaService.js` | Ollama/LLM integration — `analyzeDocument()`, `_processOllamaResponse()` |
| `services/restrictionPromptService.js` | Prompt restriction placeholder replacement |
| `models/document.js` | SQLite operations — `isDocumentProcessed()`, `deleteDocumentsIdList()`, `getProcessedDocuments()` |
| `routes/setup.js` | Web routes — dashboard (line 2578), setup wizard, auth |
| `config/config.js` | Environment config loading |
| `main.py` | RAG service (Python) — `documents.json`, chromadb |
