# DATA Foundation Documentation

This repository contains the official documentation for **DATA Foundation**, a purpose-built layer 1 blockchain designed specifically for intellectual property.

📚 [Official Documentation](https://docs.datafdn.org)

## Proposing Changes

If you see any issues with the docs or would like to add/modify them, please open a Pull Request and tag @jacob-tucker :)

## Development

Install the [Mintlify CLI](https://www.npmjs.com/package/mintlify) to preview the documentation changes locally. To install, use the following command

```
npm i -g mintlify
```

Run the following command at the root of the repository (where docs.json is)

```
mintlify dev
```

## Navigation Updating Script

Once you created a new page, you can use this script to add it to the navigation structure in the `docs.json` file.

### Usage

Run it from the root folder.

```bash
python add_it.py <path/to/file.mdx>
```

### Examples

```bash
# Addin the `multisig` page to the `network/more` section
python add_it.py network/more/multisig.mdx
```

Supports .md and .mdx files. Run from directory containing docs.json.

### Troubleshooting

- Mintlify dev isn't running - Run `mintlify install` it'll re-install dependencies.
- Page loads as a 404 - Make sure you are running in a folder with `docs.json`
- `Add It` script not working? Is Python 3 Installed? Is the path correct? Still not working? Please report an [issue](https://github.com/thedatafoundation/docs/issues)
