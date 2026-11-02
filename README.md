# hf-publish

An [hf CLI extension](https://huggingface.co/docs/huggingface_hub/en/guides/cli) that publishes a project to
Hugging Face as its `publish.toml` describes: it runs any build step, stages the listed files into a temporary
folder laid out like the repo, and uploads it with `hf upload` (or syncs a folder to a bucket with
`hf buckets sync`).

```bash
hf extensions install prteek/hf-publish
hf publish -m "Fix the card" dataset space    # publish these targets
hf publish                                     # publish the config's default targets
hf publish dataset-card                        # only the dataset target's README.md
hf publish -n space                            # dry run: build and stage, print the commands and files
hf publish --list                              # the project's targets
```

`publish.toml` is found in the current folder or the nearest folder above it, and paths in it are relative to that
folder.

## publish.toml

```toml
default = ["dataset", "space"]          # optional: targets published when none are given (default: all)

[targets.dataset]
repo = "user/my-dataset"
type = "dataset"                        # model, dataset or space
build = "bash dataset/build.sh"         # optional: run from the project folder before the files are copied,
                                        # with $STAGE set to the staging folder (the repo's root)
files = { "." = ["dataset/README.md", "data/*.csv"] }

[targets.space]
repo = "user/my-space"
type = "space"
# repo folder -> files copied into it; "." is the repo's root; globs are allowed (no spaces in paths)
files = { "." = ["space/app.py", "space/README.md", "space/requirements.txt"], "mypkg" = ["mypkg/__init__.py", "mypkg/core.py"] }

[targets.blog]
type = "bucket"
bucket = "user/assets/my-project"       # synced to hf://buckets/user/assets/my-project
dir = "blog/assets"
```

Every repo target `<name>` also gets `<name>-card`, which uploads only the `README.md` that the target puts at its
root. Use it to update a card without rebuilding the data, for example from CI when the data isn't in git.

## Environment

- `HF_CLI`: the hf command to upload with. Default: `hf` if it is on PATH (so `uv run hf publish` uploads with
  the project's hf), else `uvx --from huggingface_hub hf`.
- `HF_PUBLISH_PYTHON`: a Python 3.11+ used to read `publish.toml` (it needs `tomllib`). Default: `python3` or
  `python`, else `uv run`.
- Authentication is hf's own: `hf auth login` locally, `HF_TOKEN` in CI.

## In GitHub Actions

```yaml
    env:
      HF_TOKEN: ${{ secrets.HF_TOKEN }}
      HF_HUB_DISABLE_UPDATE_CHECK: "1"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install huggingface_hub
      - run: hf extensions install prteek/hf-publish
      - run: hf publish -m "Deploy ${GITHUB_SHA::7} from GitHub" space
```

It is a single bash script (bash 3.2 and up, so macOS's `/bin/bash` works). Windows isn't supported, because hf
runs shell-script extensions only on macOS and Linux.
