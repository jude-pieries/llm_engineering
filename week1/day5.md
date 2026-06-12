# Day 5 — Company Brochure Generator

## What it does

Takes a company name and URL, crawls the site, uses an LLM to pick the most relevant pages, fetches their content, then generates a polished markdown brochure using a second LLM call. The final version streams the output back token-by-token for a typewriter effect.

---

## How the pieces fit together

### 1. Setup & dependencies

```python
from scraper import fetch_website_links, fetch_website_contents
from openai import OpenAI

openai = OpenAI()  # client instance — must be run before any function that uses it
MODEL = 'gpt-5-nano'
```

`scraper` is a local module providing two helpers:
- `fetch_website_links(url)` — returns all `<a href>` links found on the page
- `fetch_website_contents(url)` — returns the cleaned text content of a page

`openai` is assigned the `OpenAI()` **client instance** (not the library itself). This is why cells must be run in order — if this cell is skipped, every function that calls `openai.chat.completions.create(...)` will raise `NameError: name 'openai' is not defined`.

---

### 2. Link selection — `select_relevant_links(url)`

**Goal:** Given a URL, figure out which links on that page are worth reading for a brochure (About, Careers, Blog, etc.) and discard noise (Terms, Privacy, social icons).

**How it works:**

1. `get_links_user_prompt(url)` calls `fetch_website_links(url)` and formats the raw link list into a prompt.
2. `link_system_prompt` instructs the model to respond with a JSON object listing only relevant links with their type and full URL. This is **one-shot prompting** — an example response is embedded in the system prompt so the model knows the exact format expected.
3. The OpenAI call uses `response_format={"type": "json_object"}` to force valid JSON output.
4. The result is parsed with `json.loads()` and returned as a Python dict.

```python
# Abbreviated flow:
links = fetch_website_links(url)          # raw links from the page
prompt = format_as_user_prompt(links)     # build the user message
response = openai.chat.completions.create(
    model=MODEL,
    messages=[system_prompt, user_prompt],
    response_format={"type": "json_object"}
)
return json.loads(response.choices[0].message.content)
# → {"links": [{"type": "about page", "url": "https://..."}, ...]}
```

---

### 3. Content aggregation — `fetch_page_and_all_relevant_links(url)`

**Goal:** Assemble all the text the brochure generator will need.

1. Fetches the landing page text via `fetch_website_contents(url)`.
2. Calls `select_relevant_links(url)` to get the curated link list.
3. Iterates over each link, fetching its content and appending it under a labelled heading.

The result is one large string:

```
## Landing Page:
<homepage text>

## Relevant Links:

### Link: about page
<about page text>

### Link: careers page
<careers page text>
...
```

---

### 4. Brochure prompt assembly — `get_brochure_user_prompt(company_name, url)`

Calls `fetch_page_and_all_relevant_links(url)` and prepends a brief instruction naming the company. The combined string is **truncated to 5,000 characters** to stay within token limits before being sent to the model.

---

### 5. Brochure generation — `create_brochure(company_name, url)`

A straightforward single LLM call using `gpt-4.1-mini`. The `brochure_system_prompt` tells the model to write a short markdown brochure covering culture, customers, and careers. The result is rendered inline via `display(Markdown(result))`.

A commented-out humorous variant of the system prompt is also provided to demonstrate how easily tone can be changed by swapping one prompt string.

---

### 6. Streaming version — `stream_brochure(company_name, url)`

Identical to `create_brochure` but passes `stream=True` to the API. Instead of waiting for the full response, it:

1. Creates a live `display_handle` with an empty Markdown cell.
2. Iterates over streamed chunks, appending each `delta.content` token to a running `response` string.
3. Calls `update_display(Markdown(response), ...)` on every chunk — producing the typewriter animation.

```python
for chunk in stream:
    response += chunk.choices[0].delta.content or ''
    update_display(Markdown(response), display_id=display_handle.display_id)
```

---

## End-to-end flow

```
stream_brochure("HuggingFace", "https://huggingface.co")
        │
        ├─ get_brochure_user_prompt(...)
        │       ├─ fetch_page_and_all_relevant_links(url)
        │       │       ├─ fetch_website_contents(url)          ← landing page text
        │       │       └─ select_relevant_links(url)
        │       │               ├─ fetch_website_links(url)     ← raw links
        │       │               └─ LLM call (gpt-5-nano)        ← picks relevant ones
        │       │               └─ fetch_website_contents(each) ← page text per link
        │       └─ truncate to 5,000 chars
        │
        └─ LLM streaming call (gpt-4.1-mini)
                └─ update_display() per token → typewriter output
```

---

## Key concepts introduced

| Concept | Where |
|---|---|
| One-shot prompting | `link_system_prompt` includes a JSON example |
| Structured / JSON output | `response_format={"type": "json_object"}` |
| Multi-step LLM chaining | Link selection → content fetch → brochure generation |
| Streaming responses | `stream=True` + iterating chunks with `update_display` |
| Prompt tone control | Swapping the system prompt for a humorous variant |
| Token limit management | `user_prompt[:5_000]` truncation |
