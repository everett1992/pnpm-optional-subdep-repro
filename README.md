# pnpm fails on an unresolvable dependency inside an optional subtree

Each project here has one optional dependency, and that package has a regular
dependency on `@repro/missing`, which the registry doesn't have (404):

```
freebsd-only/  repro -optional-> @repro/freebsd-only (os: freebsd) -> @repro/missing (404)
any-os/        repro -optional-> @repro/any-os                     -> @repro/missing (404)
```

| project | npm 11.19.1 | pnpm 10.34.6 | pnpm 11.28.3 | pnpm 12.8.1 |
|---|---|---|---|---|
| `freebsd-only` | OK | OK | OK | **`ERR_PNPM_FETCH_404`** |
| `any-os` | OK | **`ERR_PNPM_FETCH_404`** | **`ERR_PNPM_FETCH_404`** | **`ERR_PNPM_FETCH_404`** |

There are two problems:

1. **Regression in pnpm 12 (`freebsd-only`).** `@repro/freebsd-only` is
   skipped on this platform, so `@repro/missing` is never needed. pnpm 10 and
   11 make every dependency of a package that isn't installable on this
   platform optional, so the 404 is skipped. pnpm 12 doesn't, and fails.
2. **pnpm differs from npm (`any-os`).** npm lets anything under an optional
   edge fail: it records `@repro/any-os` in `package-lock.json` as
   `"optional": true` and installs nothing. In pnpm, only the optional edge
   itself may fail. A regular dependency of an installable optional package
   must resolve. That's also why the pnpm 10 and 11 result for `freebsd-only`
   depends on the platform: on FreeBSD the package is installable and the
   install fails there. You can see this without FreeBSD by adding
   `supportedArchitectures: { os: [freebsd] }` to a `pnpm-workspace.yaml` in
   `freebsd-only/`.

## Reproduce

Run on any OS except FreeBSD.

The repro uses a local [pnpr](https://github.com/pnpm/pnpm/tree/main/pnpr)
registry so that it's self-contained. Nothing about the failure depends on
pnpr: it happens with any registry that doesn't serve the dependency, such as
a mirror that filters packages. A registry that returns a packument with no
versions fails the same way, with `ERR_PNPM_NO_VERSIONS` instead of
`ERR_PNPM_FETCH_404`. `registry/storage/` already contains the two packages, so
nothing needs to be published. Each project's `.npmrc` routes only the `@repro`
scope to it.

Start the registry in one terminal:

```sh
npx @pnpm/pnpr@0.1.0-alpha.14 -c registry/pnpr.yaml
```

Install in another terminal:

```sh
for dir in freebsd-only any-os; do
  for pm in npm@11.19.1 pnpm@10.34.6 pnpm@11.28.3 pnpm@12.8.1; do
    (cd "$dir" && rm -rf node_modules pnpm-lock.yaml package-lock.json &&
      npx -y "$pm" install) && echo "$dir $pm: OK" || echo "$dir $pm: FAILED"
  done
done
```

pnpm 12.8.1 output:

```
Error: ERR_PNPM_FETCH_404

  × installing dependencies
  ╰─▶ Failed to resolve dependency tree: GET http://127.0.0.1:7677/
      @repro%2Fmissing: Not Found - 404
```

For `freebsd-only`, pnpm 10 and 11 lock the package without its dependency:

```yaml
snapshots:

  '@repro/freebsd-only@1.0.0':
    optional: true
```

## Related issues

- [#13041](https://github.com/pnpm/pnpm/issues/13041): pnpm 12 aborted on an
  unresolvable *direct* optional dependency. That was fixed by deduplicating
  edges, and it didn't change how dependencies of a skipped package are
  handled.
- [#13326](https://github.com/pnpm/pnpm/issues/13326): the subtree of a
  platform-incompatible optional package is still resolved in both pnpm 11 and
  pnpm 12.
