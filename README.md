# "Cannot resolve source" warnings when using unioned API configurations with dependencies

When generating Hume SDKs, I get a bunch of spammy

[unioned]: `Cannot resolve source <open-api-spec> from file .../__package__.yml` warnings.

This repo minimally reproduces this. This appears to happen when you have a "unioned" API definition with a dependencies.yml that points to other APIs.

In this repo we have


**Directory structure:**
```
fern/
├── fern.config.json
└── apis/
    ├── api-a/
    │   ├── generators.yml
    │   └── openapi.json
    ├── api-b/
    │   ├── generators.yml
    │   └── openapi.json
    └── unioned/
        ├── generators.yml
        ├── dependencies.yml
        └── definition/
            ├── api.yml
            ├── api-a/__package__.yml
            └── api-b/__package__.yml
```

**Key files:**

`apis/unioned/dependencies.yml`:
```yaml
dependencies:
  api-a: ../api-a
  api-b: ../api-b
```

`apis/unioned/definition/api-a/__package__.yml`:
```yaml
export: api-a
```

`apis/unioned/definition/api-b/__package__.yml`:
```yaml
export: api-b
```

**Command:**
```bash
cd fern && fern generate --api unioned --group ts-sdk --preview
```

**Output:**
```
...
[unioned]: fernapi/fern-typescript-node-sdk Cannot resolve source openapi.json from file api-a/__package__.yml
[unioned]: fernapi/fern-typescript-node-sdk Cannot resolve source openapi.json from file api-b/__package__.yml
...
```

## Cause (according to Claude)


Claude says the cause is
in `packages/cli/cli-source-resolver/src/SourceResolverImpl.ts` in the fern source code:

1. During dependency loading, `OpenAPILoader.loadDocuments()` correctly computes source paths that include the `relativePathToDependency` (e.g., `../api-a/openapi.json`)

2. However, `SourceResolverImpl.resolveOpenAPISource()` (line 68-83) attempts to resolve these paths by joining them with `this.workspace.absoluteFilePath`, which points to the unioned API directory

3. This results in trying to find files at incorrect locations:
   - Looking for: `/path/to/unioned/openapi.json`
   - Actual location: `/path/to/api-a/openapi.json`

4. The `relativePathToDependency` information that was added to the `OpenAPISpec` in `getAllOpenAPISpecs()` is not available in `SourceResolverImpl`, which only receives the raw `SourceSchema` containing just the filename.
