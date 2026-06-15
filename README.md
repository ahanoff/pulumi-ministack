# Testing Pulumi locally with MiniStack

[![Infrastructure Tests](https://github.com/ahanoff/pulumi-ministack/actions/workflows/infrastructure-tests.yml/badge.svg)](https://github.com/ahanoff/pulumi-ministack/actions/workflows/infrastructure-tests.yml)

Sample codebase accompanying the blog post [Testing Pulumi stacks locally with MiniStack](https://ahanoff.dev/blog/testing-pulumi-locally-with-ministack).

## Prerequisites

- [Podman](https://podman.io/getting-started/installation) (or [Docker](https://docs.docker.com/get-docker/))
- [Pulumi CLI](https://www.pulumi.com/docs/install/)
- [Bun](https://bun.sh/)

## Quick start

Start MiniStack:

```bash
podman run -p 4566:4566 ministackorg/ministack

# Docker works too
# docker run -p 4566:4566 ministackorg/ministack
```

Install dependencies and run the stack:

```bash
bun install
make init           # create the local stack (once)
make local-preview   # dry run
make local-up        # deploy against MiniStack
```

Verify resources exist in MiniStack:

```bash
aws --endpoint-url=http://localhost:4566 \
  dynamodb describe-table --table-name users \
  --query 'Table.TableName' --output text

aws --endpoint-url=http://localhost:4566 \
  secretsmanager describe-secret --secret-id db-password \
  --query 'Name' --output text
```

Clean up:

```bash
make local-destroy
make reset
```

## What's in here

| File | Purpose |
| --- | --- |
| `Pulumi.yaml` | Project definition |
| `Pulumi.local.yaml` | Stack config with MiniStack endpoint overrides |
| `index.ts` | Infrastructure: native DynamoDB table + custom dynamic resource |
| `managed-secret.ts` | Custom `pulumi.dynamic.ResourceProvider` for Secrets Manager |
| `Makefile` | Convenience commands for local testing |
| `.githooks/pre-push` | Blocks push if `pulumi preview` fails against MiniStack |
| `.github/workflows/infrastructure-tests.yml` | GitHub Actions CI workflow |

## Pre-push hook

A git hook in `.githooks/pre-push` runs `pulumi preview` against MiniStack before every push. If MiniStack is not running, it skips with a warning. If preview fails (unsupported resource, config error), the push is blocked.

Enable it once after cloning:

```bash
git config core.hooksPath .githooks
```

## How endpoints reach both code paths

1. **`Pulumi.local.yaml`** (`aws:endpoints`): endpoint overrides for the native `@pulumi/aws` provider.
2. **`Pulumi.local.yaml`** (`ministack:endpoint`): read by dynamic resource providers via `pulumi.getConfig()`.

Both come from the same file. No environment variables needed for endpoint configuration.
