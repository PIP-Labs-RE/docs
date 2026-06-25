# Repository guidelines

## Never edit Korean (`ko/`) files

Do **not** create, edit, or otherwise modify any files under the `ko/` directory. All Korean content is generated and kept in sync automatically by Mintlify's translation pipeline — any manual changes will be overwritten and just create noise.

When a change needs to apply to Korean too, edit only the English source file. The translation will be regenerated automatically. This applies to every `ko/**` file (`.mdx`, `.md`, etc.).

The one exception is `docs.json`: its `navigation.languages` array (including the `ko` entry) is repo-managed configuration, not auto-translated content, so it may be edited when navigation structure changes.
