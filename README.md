# "Cannot resolve source" warnings when using unioned API configurations with dependencies

## Description

When using a unioned API configuration that depends on other APIs (via `dependencies.yml`), the `fern generate` command produces numerous "Cannot resolve source X from file Y" warnings during SDK generation. While these warnings don't cause generation to fail, they clutter the output significantly.

## Reproduction

This repository contains a minimal reproducible example.

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
[unioned]: Writing preview to .../fern/apis/unioned/.preview
[unioned]: Download ../api-a Started.
[unioned]: Download ../api-a Parsing...
[unioned]: Download ../api-b Started.
[unioned]: Download ../api-b Parsing...
[unioned]: Download ../api-a Modifying source filepath ...
[unioned]: Download ../api-b Modifying source filepath ...
[unioned]: Download ../api-a Loaded...
[unioned]: Download ../api-a Finished.
[unioned]: Download ../api-b Loaded...
[unioned]: Download ../api-b Finished.
[unioned]: ✓ All checks passed
[unioned]: fernapi/fern-typescript-node-sdk Started.
[unioned]: fernapi/fern-typescript-node-sdk Cannot resolve source openapi.json from file api-a/__package__.yml
[unioned]: fernapi/fern-typescript-node-sdk Cannot resolve source openapi.json from file api-b/__package__.yml
[unioned]: fernapi/fern-typescript-node-sdk Downloaded to .../fern/apis/unioned/.preview/fern-typescript-node-sdk
[unioned]: fernapi/fern-typescript-node-sdk Finished.
```

## Root Cause Analysis

I've traced the issue to `packages/cli/cli-source-resolver/src/SourceResolverImpl.ts` in the fern source code:

1. During dependency loading, `OpenAPILoader.loadDocuments()` correctly computes source paths that include the `relativePathToDependency` (e.g., `../api-a/openapi.json`)

2. However, `SourceResolverImpl.resolveOpenAPISource()` (line 68-83) attempts to resolve these paths by joining them with `this.workspace.absoluteFilePath`, which points to the unioned API directory

3. This results in trying to find files at incorrect locations:
   - Looking for: `/path/to/unioned/openapi.json`
   - Actual location: `/path/to/api-a/openapi.json`

4. The `relativePathToDependency` information that was added to the `OpenAPISpec` in `getAllOpenAPISpecs()` is not available in `SourceResolverImpl`, which only receives the raw `SourceSchema` containing just the filename.

## Expected Behavior

No warnings should be emitted when source files exist at the correct dependency-relative paths.

## Actual Behavior

Warnings are emitted for every type/endpoint that references a source file from a dependency, even though:
1. The files exist
2. The OpenAPI specs are successfully loaded
3. SDK generation completes successfully

## Suggested Fix

The `SourceResolverImpl` needs access to dependency path information when resolving sources. Possible approaches:

1. Pass the `relativePathToDependency` through to `SourceResolverImpl.resolveSource()`
2. Store dependency-aware workspace paths in the resolver
3. Include the full relative path (with dependency prefix) in the `SourceSchema` itself during IR generation

## Impact

- Clutters generator output with false warnings (in complex APIs, this can produce hundreds of warnings)
- Makes it difficult to spot actual warnings/errors
- May confuse users into thinking something is wrong with their configuration

## Workaround

These warnings can be safely ignored - they don't affect the generated SDK functionality. The source information is only used for features like source mapping in generated code.

## Testing

To reproduce the issue:

```bash
cd fern
fern generate --api unioned --group ts-sdk --preview
```

You should see the "Cannot resolve source" warnings despite successful generation.
