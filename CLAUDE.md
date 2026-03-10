# CLAUDE.md — Building_with_Claude

This file provides guidance for AI assistants (e.g., Claude Code) working in this repository.

## Project Overview

An educational repository demonstrating the Anthropic Python SDK (`anthropic`) through progressively advanced examples. Each notebook or script is a self-contained, runnable demonstration of a specific Claude API capability.

**Tech stack:** Python, Jupyter Notebooks, Anthropic SDK, python-dotenv

---

## Repository Structure

```
Building_with_Claude/
├── Simple_Claude_Demo.py               # Minimal API usage example (good starting point)
├── Claude_With_Tools.ipynb             # Tool use / function calling with a tool loop
├── Claude_With_memory_demo.ipynb       # Multi-turn conversation with memory
├── Claude_Memory_&_SystemPrompt.ipynb  # System prompts for persona/behavior control
├── Claude_Streaming_data.ipynb         # Streaming API responses in real time
├── Claude_Eveluation_Framework.ipynb   # Custom eval framework with model-based grading
├── Claude_Structure_extraction.ipynb   # Structured output via stop sequences + prefill
├── Claude_images.ipynb                 # Vision / image analysis (fire risk assessment)
├── thinking_feature.ipynb              # Extended thinking (budget_tokens)
├── web_search.ipynb                    # Web search tool integration (in progress)
├── images/                             # Sample images for vision demos (prop1-7.png)
├── README.md
└── LICENSE                             # MIT
```

---

## Environment Setup

1. **Python environment** — No `requirements.txt` exists. Install dependencies manually:
   ```bash
   pip install anthropic python-dotenv
   ```
2. **API key** — Create a `.env` file in the root with:
   ```
   ANTHROPIC_API_KEY=your_key_here
   ```
   All notebooks and scripts call `load_dotenv()` at startup; the `Anthropic()` client reads `ANTHROPIC_API_KEY` automatically.

3. **Jupyter** — Run notebooks interactively:
   ```bash
   jupyter notebook
   # or
   jupyter lab
   ```

4. **Python script** — Run directly:
   ```bash
   python Simple_Claude_Demo.py
   ```

---

## Model Conventions

| Use case | Default model |
|---|---|
| General chat / demos | `claude-sonnet-4-0` |
| Cost-sensitive / eval grading | `claude-haiku-4-5-20251001` |
| Vision, web search, advanced | `claude-sonnet-4-5` |
| Extended thinking | `claude-3-7-sonnet-20250219` |

When adding new demos, default to the latest Sonnet model unless a specific feature requires otherwise. Use the model ID string directly (no helper constants).

---

## Code Patterns

Every notebook follows the same scaffold. Reproduce this pattern in new work:

```python
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()
model = "claude-sonnet-4-6"  # update to latest as needed

# --- Message helpers ---
def add_user_message(messages, text):
    messages.append({"role": "user", "content": text})

def add_assistant_message(messages, text):
    messages.append({"role": "assistant", "content": text})

def chat(messages, **params):
    message = client.messages.create(
        model=model,
        max_tokens=1000,
        messages=messages,
        **params
    )
    return message.content[0].text

# --- Conversation loop ---
messages = []
while True:
    user_input = input("> ")
    if user_input == "exit":
        break
    add_user_message(messages, user_input)
    response = chat(messages)
    add_assistant_message(messages, response)
    print(response)
```

### Key patterns to know

**Tool use loop** (`Claude_With_Tools.ipynb`):
- Define tools as dicts with `name`, `description`, `input_schema`
- Pass `tools=tools` to `client.messages.create()`
- Loop: if `stop_reason == "tool_use"`, extract `ToolUseBlock`, call the function, append a `tool_result` message, and call the API again until `stop_reason == "end_turn"`

**Streaming** (`Claude_Streaming_data.ipynb`):
```python
with client.messages.stream(model=model, max_tokens=1000, messages=messages) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

**Structured extraction** (`Claude_Structure_extraction.ipynb`):
- Use `stop_sequences=["```"]` to stop at a closing code fence
- Pre-fill the assistant turn with `"```python\n"` to force code-block output

**Extended thinking** (`thinking_feature.ipynb`):
```python
client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=messages,
)
```
Responses contain `ThinkingBlock` (or `RedactedThinkingBlock`) followed by a `TextBlock`.

**Vision** (`Claude_images.ipynb`):
- Read image → base64-encode → embed as `{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}`

---

## Development Conventions

- **Style:** `snake_case` for all functions and variables. No type hints required. Keep functions short and focused.
- **No formal tests:** This is a demo/education repo; there is no test suite. The `Claude_Eveluation_Framework.ipynb` notebook is its own self-contained evaluation tool.
- **No build step:** Nothing to compile or bundle.
- **No CI/CD:** No `.github/workflows` or equivalent.
- **`.gitignore` is absent** — avoid committing `.env`, `__pycache__/`, `.ipynb_checkpoints/`, or `.DS_Store`.
- **Each notebook is independent** — don't create shared utility modules unless a pattern is used in 3+ notebooks.
- **Images** for vision demos go in the `images/` directory.

---

## Adding a New Demo

1. Copy the scaffold above into a new notebook or `.py` file.
2. Name it descriptively: `Claude_<Feature>.ipynb`.
3. Add a markdown cell at the top explaining what the demo shows.
4. Use `load_dotenv()` and `Anthropic()` — never hardcode API keys.
5. Keep it self-contained; avoid importing from other notebooks.
6. Update this CLAUDE.md if a new pattern or dependency is introduced.

---

## Common Pitfalls

- `client.messages.create()` returns a `Message` object; the text is at `message.content[0].text` (only valid when `stop_reason == "end_turn"` and the first block is `TextBlock`). Use a helper like `text_from_message()` if the response may contain tool-use blocks.
- `max_tokens` is required for all API calls.
- Conversation history must alternate `user` / `assistant` roles. Never append two consecutive messages with the same role.
- Extended thinking requires `max_tokens >= budget_tokens + expected_output_tokens`.
- The `web_search.ipynb` demo is incomplete — the tool schema is defined but the execution loop is a TODO.
