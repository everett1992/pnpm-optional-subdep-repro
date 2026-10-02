# pnpm 12 fails on an unresolvable dependency of a skipped optional package

The project has one optional dependency, `@repro/freebsd-only`, which only
installs on FreeBSD (`"os": ["freebsd"]`). It has a regular dependency on
`@repro/missing`, which the registry doesn't have (404).

```
repro -optional-> @repro/freebsd-only (os: freebsd) -> @repro/missing (404)
```

`@repro/freebsd-only` is skipped on this platform, so `@repro/missing` is never
needed. npm, pnpm 10 and pnpm 11 install successfully. pnpm 12 fails.

| package manager | result |
|---|---|
| npm 11.19.1 | OK |
| pnpm 10.34.6 | OK |
| pnpm 11.28.3 | OK |
| pnpm 12.8.1 | **`ERR_PNPM_FETCH_404`** |

The `os` field is what makes this a regression. `@repro/any-os` is the same
package without it. With that package, pnpm 10, 11 and 12 all fail, and npm
succeeds.

## Reproduce

Run on any OS except FreeBSD. `registry/` holds a [pnpr](https://github.com/pnpm/pnpm/tree/main/pnpr)
config and its storage directory with the two packages already in it.
`.npmrc` routes the `@repro` scope to it.

Start the registry in one terminal:

```sh
npx @pnpm/pnpr@0.1.0-alpha.14 -c registry/pnpr.yaml
```

Install in another terminal:

```sh
for pm in npm@11.19.1 pnpm@10.34.6 pnpm@11.28.3 pnpm@12.8.1; do
  rm -rf node_modules pnpm-lock.yaml package-lock.json
  npx -y "$pm" install && echo "$pm: OK" || echo "$pm: FAILED"
done
```

pnpm 12.8.1 output:

```
Error: ERR_PNPM_FETCH_404

  × installing dependencies
  ╰─▶ Failed to resolve dependency tree: GET http://127.0.0.1:7677/
      @repro%2Fmissing: Not Found - 404
```

pnpm 10 and 11 lock `@repro/freebsd-only` without its dependency:

```yaml
snapshots:

  '@repro/freebsd-only@1.0.0':
    optional: true
```

To try the variant without the `os` field, change the dependency in
`package.json` to `"@repro/any-os": "1.0.0"`.

## Related issues

- [#13041](https://github.com/pnpm/pnpm/issues/13041): pnpm 12 aborted on an
  unresolvable *direct* optional dependency. That was fixed by deduplicating
  edges, and it didn't change how dependencies of a skipped package are
  handled.
- [#13326](https://github.com/pnpm/pnpm/issues/13326): the subtree of a
  platform-incompatible optional package is still resolved in both pnpm 11 and
  pnpm 12.
