---
name: weather-formatter
description: Formats raw weather payloads into human-friendly summaries.
---

# Weather Formatter

This skill takes a JSON weather payload (as returned by a typical
weather provider) and produces a one-paragraph natural-language summary.

## Inputs

- `payload` — a JSON object with `temperature`, `humidity`, `conditions`,
  and `wind_speed` keys.

## Output

A 1–3 sentence English summary. Example: "It's 72°F and sunny in Austin,
with light winds and 45% humidity."

## Usage

1. Receive the payload.
2. Convert units if the user prefers metric.
3. Return the summary string.

That's it — no network access, no file writes, no agent-level
permissions required.
