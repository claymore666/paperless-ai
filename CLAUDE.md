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
| `main` | Upstream mirror | `origin/main` (clusterzx/paperless-ai) |
| `prod` | All our fixes merged, used for production deployment | main + all fix branches |
| `fix/*` | Individual fix branches for upstream PRs | branched from main |

**Upstream appears abandoned** (last commit Nov 2025, last PR merge Jul 2025, 20+ open PRs unreviewed). We keep PRs open upstream but deploy from our `prod` branch.

## Workflow: New Fix

1. Branch from main: `git checkout main && git checkout -b fix/<name>`
2. Make changes, test on QA
3. Commit (no Claude attribution):
   ```bash
   git add <files>
   git commit -m "fix: description

   Fixes #NNN"
   ```
4. Push to fork: `git push fork fix/<name>`
5. Create PR upstream: `gh pr create --repo clusterzx/paperless-ai --head claymore666:fix/<name> --base main`
6. Merge into prod: `git checkout prod && git merge fix/<name> && git push fork prod`

**Important**: Always show PR/comment text to user for approval before posting.

## Workflow: Update prod from upstream

```bash
git checkout main && git pull origin main
git checkout prod && git merge main
git push fork prod
```

## Git Config

```
user.name = claymore666
user.email = christian.kamien@gmail.com
```

`gh` authenticated as `claymore666` on pve1. Fork remote: `fork` → `claymore666/paperless-ai`.

## Active Branches

| Branch | Issue | Status |
|--------|-------|--------|
| `prod` | — | All fixes merged, deploy from here |
| `fix/restrict-document-types` | #834 | PR #920 submitted |
| `fix/reconcile-stale-documents` | #471 | PR #921 submitted |
| `fix/num-ctx-calculation` | #913, #745 | PR #922 submitted |
| `fix/double-api-path` | #880 | PR #923 submitted |
| `fix/custom-service-max-tokens` | #802 | PR #924 submitted |

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
