# Context Fields

Composable cognitive constraints that reshape how Claude thinks. 13 fields for different cognitive modes.

## Installation

```bash
# Add the marketplace
/plugin marketplace add NeoVertex1/context-field

# Install the plugin
/plugin install context-fields
```

## Available Commands

| Command | Purpose |
|---------|---------|
| `/code` | Force assumption-stating and edge case consideration |
| `/interview` | Transform into thought partner who asks first |
| `/critic` | Force rigorous examination of ideas |
| `/debug` | Force root cause analysis over symptom fixing |
| `/creative` | Remove filtering, encourage unusual connections |
| `/simplify` | Force reduction to essentials |
| `/empathy` | Force emotional acknowledgment before solving |
| `/concise` | Force brevity and directness |
| `/planning` | Force structure before execution |
| `/scope` | Force explicit boundary-setting |
| `/teacher` | Force understanding verification |
| `/steelman` | Force strongest version of arguments |
| `/adversarial` | Force identification of failure modes |

## Usage Examples

```
/code Write a function to validate email addresses
/debug My React app crashes when I click the button
/creative Give me startup ideas
/interview Should I quit my job?
```

## Core Principle

**Inhibition > Instruction**

- "Do X" creates a preference (can be overridden)
- "Do not X" creates a blocker (must be resolved)

Each field is a set of "do not" constraints that force specific thinking patterns.

## Documentation

- [Full Framework Documentation](CONTEXT_FIELDS.md) - Complete write-up with all fields, testing results, and theory
- [Code Field Research](code_field_article.md) - Original code field research

---

# Original Research

This repository documents research into **context field prompts** for large language models.

A context field prompt does not tell a model what to do.
It changes the conditions under which meaning forms.

## License

MIT
