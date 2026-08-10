# codex-model — quick Codex model switching

Avoid scrolling through OpenRouter's list of roughly 700 models when you only
need a few fixed models (the "opus" / "sonnet" / "haiku" tiers), and switch
back to the native OpenAI endpoint with one command.

## Aliases (this is an example, you need to write your own description file, see the chapter ["Adding or changing a model"](Adding or changing a model))

| alias   | provider     | model                          | reasoning | tier   |
|---------|--------------|--------------------------------|-----------|--------|
| `terra` | `openrouter` | `openai/gpt-5.6-terra`         | high      | opus   |
| `deep`  | `openrouter` | `@preset/deepseek4pro`         | high      | sonnet |
| `flash` | `openrouter` | `@preset/deepseek4flash-cache` | medium    | haiku  |
| `oai`   | `openai`     | `gpt-5.6-terra`                | high      | —      |

## Mode 1 — temporary selection

Use Codex's native profiles (`-p` loads `~/.codex/<alias>.config.toml` on top
of the base configuration):

```sh
codex -p terra
codex -p deep
codex -p flash
codex -p oai
```

The default configuration remains unchanged, which is useful for a single
session.

## Mode 2 — persistent default

```sh
codex-model            # interactive numbered menu
codex-model terra      # set the alias as the default
codex-model current    # show the active model
codex-model list       # list aliases, ● = active
codex-model sync       # regenerate profiles after changing aliases
codex-model --help     # show the summary
```

`codex-model <alias>` rewrites only the lines for keys declared by the alias in
`~/.codex/config.toml`, preserving comments, tables, and ordering. Before
writing, it saves a copy to `~/.codex/config.toml.bak`.

`model_catalog_json` is managed automatically: it is added only when the alias
uses a model declared in `catalog_sources`, and removed when switching to an
OpenAI alias or a model without a custom catalog. This prevents an OpenRouter
catalog from remaining active accidentally after `codex-model oai`.

## Switching back to OpenAI

```sh
codex-model oai      # or: codex -p oai
```

This changes `model_provider` from `openrouter` to `openai` (Codex's built-in
provider) and removes the provider prefix from the model name: on OpenRouter it
is `openai/gpt-5.6-terra`, while the native endpoint uses `gpt-5.6-terra`.

Authentication is already configured: `~/.codex/auth.json` contains the
ChatGPT login (`auth_mode: chatgpt`, with a refresh token), so you do not need
to run `codex login` again. Check it with `codex doctor` in the `auth` section.

If a model ID is rejected, run `codex -p oai` and use `/model` inside the TUI.
The native `openai` provider has a short model list rather than OpenRouter's
700-plus entries.

To switch back to OpenRouter, use another alias (`codex-model deep`). This
requires `OPENROUTER_API_KEY`, already exported in `~/.zshrc`.

## Beware of sticky keys

A key **not** declared in an alias retains the value left by the previous alias.
This is why `model_provider` is repeated in all aliases: without it,
after `codex-model oai`, running `codex-model deep` would leave the provider set
to `openai` while using an OpenRouter model ID, causing the request to fail.

`codex-model` warns when this happens:

```
! 'model_provider' is not declared in [deep]: keeping openai
```

If you see this warning, add the missing key to the relevant alias.

## Adding or changing a model

The only file you need to edit is **`~/.codex/model_aliases.toml`**:

```toml
[new]
model_provider = "openrouter"
model = "google/gemini-3.5-flash"
model_reasoning_effort = "medium"
```

Then run:

```sh
codex-model sync      # generate ~/.codex/new.config.toml → codex -p new
codex-model new       # or set it as the default
```

The table name **is** the alias name. To call the tiers `opus` / `sonnet` /
`haiku`, rename the three tables and run `codex-model sync` again. Old files
such as `~/.codex/<old-name>.config.toml` must be removed manually.

Any valid `config.toml` key placed inside an alias table is applied, not just
`model` — for example, `model_provider` and `approval_policy`.

## Custom metadata catalog for OpenRouter presets

An OpenRouter preset (`@preset/...`) may not appear in
`~/.codex/models_cache.json`. If Codex displays the warning `Model metadata ...
not found`, define the preset's source in the same alias configuration:

```toml
[catalog_sources."@preset/deepseek4flash-cache"]
source = "deepseek/deepseek-v4-flash"
display_name = "DeepSeek V4 Flash (cache preset)"
```

Then regenerate the catalog and select the preset:

```sh
codex-model catalog     # write ~/.codex/openrouter_catalog.json
codex-model flash       # enable model_catalog_json in config.toml
```

`codex-model catalog` only generates the JSON metadata file; selecting the
preset alias is what updates `config.toml`.

You do not need to add `model_catalog_json` to aliases manually: the script
adds it to aliases whose model slug is present in `catalog_sources`. When you
select `codex-model terra` or `codex-model oai`, the line is removed instead.

The command reads the original entry from the cache and writes the complete
Codex catalog document in the form `{"models": [...]}`. Do not write the entry
object directly with `jq`: that produces a document without the root `models`
field and causes the `missing field models` error.

The equivalent generation using only `jq` is:

```sh
jq --arg slug '@preset/deepseek4flash-cache' \
   --arg name 'DeepSeek V4 Flash (cache preset)' \
   '(.models | map(select(.slug == "deepseek/deepseek-v4-flash"))) as $found
    | if ($found | length) != 1 then
        error("source model not found or duplicated")
      else
        {models: [($found[0] | .slug = $slug | .display_name = $name)]}
      end' \
   ~/.codex/models_cache.json > ~/.codex/openrouter_catalog.json
```

Add more `catalog_sources` tables to support multiple presets;
`codex-model catalog` includes them all. After Codex updates its model cache,
run the command again to refresh the metadata.

To see the exact OpenRouter model IDs:

```sh
curl -s https://openrouter.ai/api/v1/models | jq -r '.data[].id' | sort
```

## Files

| path                                | purpose |
|-------------------------------------|---------|
| `~/.local/bin/codex-model`          | the script (Python 3.11+, no dependencies) |
| `~/.codex/model_aliases.toml`       | alias definitions — the only file to edit |
| `~/.codex/<alias>.config.toml`      | profiles generated by `sync`, for `codex -p` |
| `~/.codex/openrouter_catalog.json`  | custom metadata generated by `catalog` |
| `~/.codex/config.toml`              | Codex configuration modified by `codex-model <alias>` |
| `~/.codex/config.toml.bak`          | automatic backup of the last switch |
| `~/.codex/auth.json`                | ChatGPT login used by the `oai` alias |

## Troubleshooting

```sh
cp ~/.codex/config.toml.bak ~/.codex/config.toml   # restore the backup
codex doctor                                        # check config, auth, provider
```
