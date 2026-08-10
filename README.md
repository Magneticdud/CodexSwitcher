# codex-model — switch rapido dei modelli Codex

Serve a evitare la lista di ~700 modelli di OpenRouter quando vuoi solo passare
tra pochi modelli fissi (tier "opus" / "sonnet" / "haiku"), e a tornare
all'endpoint OpenAI nativo con un comando.

## Gli alias

| alias   | provider     | modello                        | reasoning | tier   |
|---------|--------------|--------------------------------|-----------|--------|
| `terra` | `openrouter` | `openai/gpt-5.6-terra`         | high      | opus   |
| `deep`  | `openrouter` | `@preset/deepseek4pro`         | high      | sonnet |
| `flash` | `openrouter` | `@preset/deepseek4flash-cache` | medium    | haiku  |
| `oai`   | `openai`     | `gpt-5.6-terra`                | high      | —      |

## Modo 1 — al volo, senza toccare la config

Usa i profili nativi di Codex (`-p` carica `~/.codex/<alias>.config.toml`
sopra la config base):

```sh
codex -p terra
codex -p deep
codex -p flash
codex -p oai
```

La config di default resta quella che è: utile per una singola sessione.

## Modo 2 — cambiare il default persistente

```sh
codex-model            # menu interattivo numerato
codex-model terra      # imposta l'alias come default
codex-model current    # mostra il modello attivo
codex-model list       # elenco degli alias, ● = attivo
codex-model sync       # rigenera i profili dopo aver modificato gli alias
codex-model --help     # riepilogo
```

`codex-model <alias>` riscrive in `~/.codex/config.toml` solo le righe delle
chiavi dichiarate nell'alias, lasciando intatti commenti, tabelle e ordine.
Prima di scrivere salva una copia in `~/.codex/config.toml.bak`.

## Tornare all'endpoint OpenAI

```sh
codex-model oai      # oppure: codex -p oai
```

Cambia `model_provider` da `openrouter` a `openai` (il provider integrato di
Codex) e toglie il prefisso dal nome del modello: su OpenRouter è
`openai/gpt-5.6-terra`, sull'endpoint nativo è `gpt-5.6-terra`.

L'autenticazione è già a posto: `~/.codex/auth.json` contiene il login ChatGPT
(`auth_mode: chatgpt`, con refresh token), quindi non serve rifare `codex login`.
Verifica con `codex doctor` — sezione `auth`.

Se un id modello viene rifiutato, lancia `codex -p oai` e usa `/model` dentro la
TUI: con provider `openai` la lista è corta, non 700 voci.

Per tornare su OpenRouter basta un altro alias (`codex-model deep`): serve la
variabile `OPENROUTER_API_KEY`, già esportata in `~/.zshrc`.

## Attenzione alle chiavi "appiccicate"

Una chiave **non** dichiarata in un alias mantiene il valore lasciato dall'alias
precedente. È il motivo per cui `model_provider` è ripetuto in tutti e quattro
gli alias: senza, dopo `codex-model oai` un `codex-model deep` resterebbe su
provider `openai` con un id OpenRouter, e fallirebbe.

`codex-model` avvisa da solo quando succede:

```
! 'model_provider' non e' dichiarato in [deep]: resta openai
```

Se vedi quel warning, aggiungi la chiave mancante all'alias in questione.

## Aggiungere o modificare un modello

Unico file da toccare: **`~/.codex/model_aliases.toml`**

```toml
[nuovo]
model_provider = "openrouter"
model = "google/gemini-3.5-flash"
model_reasoning_effort = "medium"
```

Poi:

```sh
codex-model sync      # genera ~/.codex/nuovo.config.toml → codex -p nuovo
codex-model nuovo     # oppure impostalo come default
```

Il nome della tabella **è** il nome dell'alias: per chiamarli `opus`/`sonnet`/`haiku`
basta rinominare le tre tabelle e rilanciare `codex-model sync`
(i vecchi file `~/.codex/<vecchio-nome>.config.toml` vanno cancellati a mano).

Qualsiasi chiave valida di `config.toml` messa dentro una tabella viene applicata,
non solo `model` — es. `model_provider`, `approval_policy`.

Per vedere gli id esatti dei modelli OpenRouter:

```sh
curl -s https://openrouter.ai/api/v1/models | jq -r '.data[].id' | sort
```

## File coinvolti

| percorso                          | cosa |
|-----------------------------------|------|
| `~/.local/bin/codex-model`        | lo script (Python 3.11+, nessuna dipendenza) |
| `~/.codex/model_aliases.toml`     | definizione degli alias — l'unico da editare |
| `~/.codex/<alias>.config.toml`    | profili generati da `sync`, per `codex -p` |
| `~/.codex/config.toml`            | config Codex, modificata da `codex-model <alias>` |
| `~/.codex/config.toml.bak`        | backup automatico dell'ultimo switch |
| `~/.codex/auth.json`              | login ChatGPT, usato dall'alias `oai` |

## Se qualcosa va storto

```sh
cp ~/.codex/config.toml.bak ~/.codex/config.toml   # ripristina il backup
codex doctor                                        # config, auth, provider
```
