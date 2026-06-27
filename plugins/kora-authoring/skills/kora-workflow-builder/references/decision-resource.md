# Decision resource

Use `kora schema get decision --json` for exact field truth. Use this file
for the quick mental model.

## What a decision does

A Decision is a named business-rule bundle used by `decision` flow nodes.

## Hit policies

- `first` — first matching rule wins
- `unique` — expect exactly one matching rule
- `collect` — gather all matching outputs

## Usage reminders

- decision flow nodes reference a Decision by name
- keep rule conditions explicit and readable
- use annotations when they help future review
- validate after editing because workflows can break if rule outputs drift from
  downstream expectations
