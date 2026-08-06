# Qxern v6 public independent audit notebooks

This package contains public, English-language, output-stripped Colab notebooks for the independent Qxern v6 follow-up discussed in the Hugging Face Forum thread.

## Files

- `Qxern_v6_independent_audit_public.ipynb` — canonical public notebook. Reproduces:
  - causal sidecar/latent replacement;
  - coarse sample-specific semantic controls;
  - external identifier accessibility;
  - candidate-free identifier receiver response;
  - pairwise fine-grained behavior sensitivity.
- `Qxern_v6_independent_audit_public_executed_T4.ipynb` — sanitized executed snapshot of the core notebook. It retains concise text tables and completion records while omitting embedded plots, widgets, download JavaScript, and repetitive progress logs.
- `Qxern_v6_cross_receiver_bridge_appendix_public.ipynb` — optional negative compatibility appendix for the lightweight 0.8B→2B bridge.

## Excluded from the public canonical set

- The earlier closed-set identifier notebook, superseded by candidate-free scoring.
- Exploratory history/state-stress notebooks whose fresh-state gates did not pass.
- Internal handoff notes, forum-drafting material, debugging-only paths, and intermediate result archives.

## Suggested repository layout

```text
independent_audit/
├── README.md
├── Qxern_v6_independent_audit_public.ipynb
├── Qxern_v6_independent_audit_public_executed_T4.ipynb
└── Qxern_v6_cross_receiver_bridge_appendix_public.ipynb
```

## Public references

- Qxern repository: https://github.com/bezvka/Qxern
- Released deployment bundle: https://huggingface.co/aximi/qxern-v6-deploy-int4
- Forum audit post: https://discuss.huggingface.co/t/qxern-v6-two-llms-that-talk-in-latent-space-32-tokens-zero-code-text-a-symbolic-sidecar-built-in-one-week/178411/6

## Reproducibility notes

- Use a fresh Colab GPU runtime.
- Run each notebook from top to bottom once.
- Model, adapter, and dataset revisions are pinned in the notebook specifications.
- Each clean notebook automatically packages raw outputs, summary tables, metadata, logs, and checksums into a result ZIP.
- The executed Core snapshot was run on a Colab Tesla T4 on 2026-08-04. Its public result bundle SHA-256 is `2fefa1e006f4644edf598aea26c72a0dfdf1420450f060c461686c220ddc88f9`.
- The core notebook combines three result archives into one final public bundle.

These notebooks are an independent follow-up, not an official upstream artifact unless adopted by the Qxern maintainer.
