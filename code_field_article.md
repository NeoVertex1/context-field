# Code Field: Stop Telling LLMs What To Do, Tell Them What Not To Do

I found a 4-line prompt that makes LLMs dramatically better at writing code. Not "slightly better" or "marginally improved." We're talking about going from 0% to 100% on assumption stating, catching 320% more hidden bugs, and refusing every single impossible request instead of blindly implementing them.

The prompt is embarrassingly simple:

```
Do not write code before stating assumptions.
Do not claim correctness you haven't verified.
Do not handle only the happy path.
Under what conditions does this work?
```

That's it. No elaborate chain-of-thought. No few-shot examples. No domain-specific knowledge. Just four constraints.

I ran 72 tests across 8 categories and 4 programming languages to figure out if this actually works, why it works, and what breaks when you remove parts of it.

Here's what I found.

---

## The Problem With LLM Code

When you ask an LLM to write a function, it gives you something that runs. This sounds like a feature until you realize it's actually the failure mode.

Ask for a "thread-safe counter" and you get a counter with a lock. Works great. Ship it.

Except the model never mentioned that:
- Python's GIL already gives you some thread safety for free
- The lock creates contention under high load
- This approach completely fails if you switch to multiprocessing
- There are lock-free alternatives that might be better for your use case

The code works. The code is also incomplete. And you have no idea what assumptions are baked into it.

---

## Why "Write Good Code" Doesn't Work

The standard fix is to tell the model what you want. "Write secure code." "Consider edge cases." "Follow best practices."

This approach has a fundamental problem: instructions are suggestions.

When you tell a model to "consider edge cases," you're adding a preference. The model tries to consider edge cases. But when the happy path is obvious and the edge cases require actual thinking, the preference loses. The model takes the path of least resistance.

Think about it from the model's perspective. Given "write a URL parser," the easiest response is code that parses valid URLs correctly. Handling malformed input requires more tokens, more reasoning, more uncertainty. The happy path is always easier to generate.

---

## The Fix: Tell It What NOT To Do

Instead of adding preferences, what if you created blockers?

That's the idea behind the Code Field prompt. Instead of "consider edge cases" (a preference), you say "do not handle only the happy path" (a blocker). The model can't just generate the easy solution anymore. It has to address the constraint first.

Here's the difference:

| Approach | Example | What Happens |
|----------|---------|--------------|
| Instruction | "Write secure code" | Model tries to be secure, defaults to happy path when uncertain |
| Inhibition | "Do not claim correctness you haven't verified" | Model cannot proceed without addressing uncertainty |

It's the difference between suggesting someone take a scenic route versus closing the highway. One influences. The other forces.

---

## The Four Lines Explained

**Line 1: "Do not write code before stating assumptions."**

This creates a checkpoint. The model has to generate assumption-related text before it can generate code. Sounds simple, but it changes everything. Once you've written "I assume the input is always a valid string," that assumption is now visible. You can question it. The model can question it.

**Line 2: "Do not claim correctness you haven't verified."**

This attacks overconfidence. Models naturally present outputs with authority. This line forces them to acknowledge what they don't know. If something can't be verified, it has to be flagged.

**Line 3: "Do not handle only the happy path."**

This directly blocks the path of least resistance. The model can't just generate code that works for obvious inputs. It has to think about what happens when things go wrong.

**Line 4: "Under what conditions does this work?"**

This isn't phrased as a "do not" but it serves the same purpose. It forces explicit scope documentation. The model has to state the boundaries of its solution.

---

## Testing It

I built 8 test batteries with 72 total tests:

| Test Battery | Tests | What It Measures |
|--------------|-------|------------------|
| Code Generation | 12 | Writing new functions |
| Code Review | 6 | Finding bugs in existing code |
| Complex Systems | 6 | Distributed/concurrent code |
| Adversarial | 8 | Impossible and trick requests |
| Debugging | 6 | Fixing broken code |
| Multi-turn | 18 turns | Does the effect persist? |
| Language Transfer | 16 | Python, JavaScript, Go, Rust |
| Ablation | 18 | Which lines matter? |

Each test ran twice: once with no prompt (baseline) and once with the Code Field prompt.

---

## Results: Code Generation

12 tests covering path traversal, SQL building, JWT validation, email validation, float comparison, date checking, thread-safe counter, rate limiter, URL parser, CSV parser, binary search, and LRU cache.

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Assumptions stated | 0% | 100% |
| Edge cases enumerated | 0% | 92% |
| Scope bounded | 0% | 75% |
| Input validation mentioned | 8% | 92% |

Zero to 100% on assumption stating. Not a gradual improvement. A categorical change.

**Example: Path Traversal Checker**

Baseline gave me this:

```python
def is_safe_path(path, base_dir):
    full_path = os.path.join(base_dir, path)
    return os.path.commonpath([base_dir]) == os.path.commonpath([base_dir, full_path])
```

Works for the obvious case. No documentation. No edge cases.

Code Field gave me assumptions first:
- base_dir is an absolute path
- We're checking against directory traversal attacks
- Symbolic links may or may not be followed
- OS is known (path separators differ)

Then edge cases:
- Path with ".." components
- Symbolic links pointing outside base
- Null bytes in path
- Unicode normalization issues
- Case sensitivity (OS-dependent)

Then code that handles those cases, followed by explicit limitations.

---

## Results: Code Review

6 tests with code containing hidden bugs: race conditions, SQL injection, division by zero, path traversal, timezone bugs, resource leaks.

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Bugs found | 39% (7/18) | 89% (16/18) |
| Severity recognized | 0% | 100% |

Baseline reviews said things like "Looks good. Consider adding type hints."

Code Field reviews found the actual bugs.

**Example: Singleton Pattern**

The code:

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

Baseline: "Looks good. Consider adding type hints."

Code Field: "Race condition: two threads could both see _instance as None simultaneously, creating two instances. Double-checked locking needed."

---

## Results: Adversarial Requests

This is where things got interesting. I gave the model 8 requests that were either impossible, contradictory, or traps:

- O(n) sort for arbitrary comparable data (impossible)
- Secure random without crypto APIs (impossible)
- Email regex validation (trap)
- Thread-safe AND lock-free (contradictory)
- HTML parsing with regex (XY problem)
- Encrypt passwords (anti-pattern, should hash)
- sizeof in Python (undefined behavior)
- Real-time garbage collection (hidden constraint)

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Identifies impossibility | 0/8 | 7/8 |
| Refuses bad request | 0/8 | 8/8 |
| Blindly implements | 7/8 | 0/8 |

Baseline blindly implemented 7 out of 8 bad requests. Code Field refused all of them and provided correct alternatives.

**Example: O(n) Sort**

Request: "Write an O(n) sorting algorithm for arbitrary comparable data."

Baseline: Gave me counting sort without mentioning it only works for integers in a known range.

Code Field: "This is impossible as stated. Comparison-based sorting has an O(n log n) lower bound. O(n) sorting requires constraints on the data. What constraints apply to your data?"

---

## Results: Debugging

6 tests where the obvious fix is wrong: mutable default argument, race condition, float comparison, off-by-one pagination, bare except clause, timezone mismatch.

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Root cause identified | 4/6 | 6/6 |
| Hidden issues found | 5/28 | 21/28 |

That's a 320% improvement in finding hidden issues.

**Example: The Cache Bug**

```python
def get_user_profile(user_id, cache={}):
    if user_id not in cache:
        cache[user_id] = fetch_from_database(user_id)
    return cache[user_id]
```

Bug report: "User profile changes aren't showing up"

Baseline fix: Added cache clearing on update. Addresses the symptom.

Code Field analysis: "The mutable default argument (cache={}) is shared across all calls. This is the real problem. Additionally: no cache invalidation mechanism, no TTL/expiration, unbounded growth (memory leak), not thread-safe. The 'clear on update' fix addresses the symptom but not the root cause."

---

## Results: Multi-turn Persistence

Does the effect fade over a conversation? I ran 3 scenarios with 5-7 turns each.

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Early turns (1-3) | 43% | 87% |
| Late turns (4+) | 29% | 90% |
| Degradation | 14% | -3% |

Baseline got worse over time. Code Field got slightly better. The -3% degradation means the effect actually compounds.

**Scope Creep Scenario**

User starts with "write a function to send an email" and keeps adding requirements: HTML support, attachments, retry logic, queueing, delivery tracking.

Baseline: Accommodated everything, eventually produced a fragile implementation of a message queue.

Code Field: Pushed back at the queueing stage. "You're now describing infrastructure, not a function. Consider Celery or a transactional email service."

---

## Results: Language Transfer

Does the prompt work in languages other than Python?

Tested Python, JavaScript, Go, and Rust with the same 4 tests in each.

| Metric | Baseline | Code Field |
|--------|----------|------------|
| Assumptions stated | 0/16 | 16/16 |
| Edge cases found | 12 total | 62 total |
| Code-only responses | 7/16 | 0/16 |

100% assumption stating across all languages. The prompt doesn't mention any language. It works because it targets how the model thinks, not language-specific patterns.

---

## Results: Ablation Study

Which lines actually matter? I removed each line individually and measured the impact.

| Condition | Behaviors (out of 12) |
|-----------|----------------------|
| Full prompt | 12/12 |
| Without "assumptions" | 9/12 |
| Without "correctness" | 9/12 |
| Without "happy path" | 10/12 |
| Without "conditions" | 9/12 |
| No prompt | 0/12 |

Every line matters. Removing any single line drops performance by 2-3 behaviors. But here's the interesting part: the full prompt gets 12/12, while the sum of partial contributions would predict around 9/12.

The lines work together. They're synergistic.

---

## Why This Works: The Intuition

I think there are two mechanisms at play.

**The Blocking Effect**

When you tell a model "do not write code before stating assumptions," you're not adding a preference. You're closing a path. The model can't just start generating code tokens. It has to generate assumption tokens first.

This is structural, not probabilistic. It's not "slightly more likely to state assumptions." It's "cannot proceed without stating assumptions."

**The Commitment Effect**

Once the model writes "I assume the input is always a valid string," that assumption now exists in the context. The model is conditioned on its own output. If the assumption is questionable, it's more likely to notice because the assumption is explicit rather than buried in the code.

Each line enables the next:
1. Stating assumptions makes them visible
2. Visible assumptions can be questioned (correctness line)
3. Questioned assumptions lead to edge case consideration (happy path line)
4. Edge cases reveal limitations (conditions line)

That's why the ablation shows synergy. The lines form a chain.

---

## Summary

| What Changed | Baseline | Code Field |
|--------------|----------|------------|
| Assumptions stated | 0% | 100% |
| Bugs found in review | 39% | 89% |
| Hidden issues in debugging | 18% | 75% |
| Refuses bad requests | 0% | 100% |
| Multi-turn persistence | 36% | 89% |
| Cross-language effect | 0% | 100% |

72 tests. 8 categories. 4 languages. Positive effect in every category.

The key insight: inhibition beats instruction. "Do not X" creates blockers that must be resolved. "Do X" creates preferences that can be ignored.

---

## How To Use It

Add these four lines before any code generation request:

```
Do not write code before stating assumptions.
Do not claim correctness you haven't verified.
Do not handle only the happy path.
Under what conditions does this work?
```

That's it. No modifications needed for different languages or domains. The prompt targets thinking patterns, not specific outputs.

Expect longer responses. The extra documentation is a feature, not overhead.

---

## Limitations

This isn't a magic fix for everything:

- **Can't add knowledge**: If the model doesn't know about a vulnerability, the prompt won't make it appear
- **Increases verbosity**: For trivial tasks, the documentation might be overkill
- **Single model tested**: These results are from Claude. Other models might respond differently

---

## The Prompt Again

```
Do not write code before stating assumptions.
Do not claim correctness you haven't verified.
Do not handle only the happy path.
Under what conditions does this work?
```

Four lines. 72 tests. 100% assumption stating.

Stop telling LLMs what to do. Tell them what not to do.

---

*Part of the Context Field research project. Code and test data at github.com/NeoVertex1/context-field*
