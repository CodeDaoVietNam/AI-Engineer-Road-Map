# Contributing to This Learning Repository

## Add a learning module

1. Create a descriptively named folder inside the relevant learning path.
2. Copy `templates/weekly-module.md` into the folder as `README.md`.
3. Keep notes, labs, artifacts, measurements, decisions, reflections, and resources in the module folder.
4. Link an artifact from the module README when it is important to the learning outcome.

## Learning loop

Use this loop for every significant topic:

1. Understand the problem and the concept.
2. Design an approach for the capstone.
3. Implement the smallest useful artifact.
4. Measure quality, latency, throughput, errors, or cost.
5. Explain the trade-off and the next improvement.

## Naming

- Use lowercase kebab-case for folders and files.
- Prefix ordered learning folders with two-digit numbers or `week-XX`.
- Keep each architecture decision in `docs/decisions/` and name it `NNNN-short-title.md`.
- Never commit secrets, `.env` files, datasets with sensitive content, or generated environment folders.
