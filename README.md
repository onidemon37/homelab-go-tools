# homelab-go-tools

Personal toolbox for kubectl/vault/infra one-off scripts, replacing ad-hoc
bash scripts with small Go commands.

## Layout

```text
homelab-go-tools/
├── go.mod
├── Makefile
├── Dockerfile
├── .dockerignore
├── .github/workflows/
│   ├── build.yml    # CI: go vet + build all tools
│   ├── docker.yml   # CI: build + push each tool's image to GHCR
│   └── lint.yml     # CI: golangci-lint + markdownlint
├── internal/
│   ├── secrets/     # resolve secrets from env vars, fallback to hidden prompt
│   └── kexec/       # run commands / copy files against k8s pods via kubectl
└── cmd/
    ├── vault-unseal/    # unseal Vault pods interactively
    └── vault-snapshot/  # take + download a Vault Raft snapshot
```

Each tool lives in its own `cmd/<toolname>/main.go` and shares the
`internal/` packages. Add a new tool by creating a new folder under `cmd/`.

## Building

```bash
make build     # builds every tool under cmd/ into ./bin/
make vet       # go vet ./...
make fmt       # gofmt everything
make tidy      # go mod tidy
```

Or run a single tool directly without building a binary:

```bash
go run ./cmd/vault-unseal
```

## vault-unseal

Unseals a list of Vault pods, submitting the configured threshold of
unseal keys to each.

### Providing keys

Keys are resolved in this order:

1. **Combined list** — one env var, comma-separated:

```bash
export VAULT_UNSEAL_KEYS="key1,key2,key3"
```

2. **Individual indexed vars**:

```bash
export VAULT_UNSEAL_KEY_1="key1"
export VAULT_UNSEAL_KEY_2="key2"
export VAULT_UNSEAL_KEY_3="key3"
```

3. **Interactive hidden prompt** — anything not covered by env vars above
   gets prompted for at runtime (input hidden, like `read -s`).

You can mix and match — e.g. set key 1 and 2 via env, and get prompted just
for key 3.

### Other overrides

```bash
export VAULT_UNSEAL_PODS="vault-1,vault-2"      # default: vault-1,vault-2
export VAULT_UNSEAL_NAMESPACE="vault"            # default: vault
export VAULT_UNSEAL_THRESHOLD="3"                # default: 3
```

### Running

```bash
go run ./cmd/vault-unseal
```

## vault-snapshot

Takes a Vault Raft snapshot inside a pod (`vault operator raft snapshot save`)
and copies it to a local `./snapshots/` directory with a timestamped filename.

```bash
export VAULT_TOKEN="s.xxxxx"                # required, needs raft snapshot permission
export VAULT_SNAPSHOT_POD="vault-1"          # default: vault-1
export VAULT_SNAPSHOT_NAMESPACE="vault"      # default: vault
export VAULT_SNAPSHOT_OUTDIR="./snapshots"   # default: ./snapshots

go run ./cmd/vault-snapshot
```

The token is piped over stdin and read inside the pod's shell rather than
put on the command line, so it doesn't show up in `ps` output inside the pod.

## Adding a new tool

1. `mkdir cmd/my-tool && touch cmd/my-tool/main.go`
2. Import shared packages as needed:

```go
import (
    "github.com/edy/homelab-go-tools/internal/secrets"
    "github.com/edy/homelab-go-tools/internal/kexec"
)
```

3. `go run ./cmd/my-tool` — no other wiring needed; `make build` and the
   `cmd/*` wildcard in the Makefile will pick it up automatically.

`secrets.ResolveN` and `secrets.StringsOrEnv` work for any tool needing
secrets or config lists from env vars with a prompt fallback — not just
Vault keys. `kexec.ShellIn` and `kexec.CopyFromPod` cover most
exec-into-pod / copy-out-of-pod needs.

## Publishing to GitHub

```bash
cd homelab-go-tools
git init
git add .
git commit -m "Initial commit: vault-unseal, vault-snapshot"
git branch -M main
git remote add origin git@github.com:<your-username>/homelab-go-tools.git
git push -u origin main
```

CI (`.github/workflows/build.yml`) runs `go vet` and builds every tool on
every push/PR to `main`.

**Before pushing**, double check `.gitignore` covers anything sensitive —
it already excludes `.env*` and `*.snap` (Vault snapshot output can contain
encrypted data but there's no reason to keep it in git), but never commit
real unseal keys or Vault tokens in a script or test fixture.
