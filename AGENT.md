# Agent architecture

Documentation agent for Task 2 with tool-calling capabilities.

## Flow

1. Read question from command line arguments.
2. Load configuration from environment variables (`LLM_API_KEY`, `LLM_API_BASE`, `LLM_MODEL`).
3. Initialize the agentic loop with a system prompt and user question.
4. Send OpenAI-compatible chat completion request with tool schemas.
5. If the LLM returns tool calls:
   - Execute each tool (`read_file` or `list_files`)
   - Append tool results as `tool` role messages
   - Loop back to step 4
6. If the LLM returns a text message (no tool calls):
   - Extract the answer and source from the response
   - Output JSON with `answer`, `source`, and `tool_calls` fields
7. Maximum 10 tool calls per question to prevent infinite loops.

## Tools

Two tools are registered as function-calling schemas:

### `read_file`

Reads a file from the project repository.

- **Parameters:** `path` (string) — relative path from project root (e.g., `wiki/git-workflow.md`)
- **Returns:** File contents as a string, or an error message if the file doesn't exist or is not a file
- **Security:** Validates that the resolved path stays within the project root (no `../` traversal)

### `list_files`

Lists files and directories at a given path.

- **Parameters:** `path` (string) — relative directory path from project root (e.g., `wiki`)
- **Returns:** Newline-separated listing of entries, or an error message if the path doesn't exist or is not a directory
- **Security:** Validates that the resolved path stays within the project root (no `../` traversal)

## Agentic loop

The loop alternates between LLM inference and tool execution:

```
Question ──▶ LLM ──▶ tool call? ──yes──▶ execute tool ──▶ back to LLM
                         │
                         no
                         │
                         ▼
                    JSON output
```

1. **Send to LLM:** User question + system prompt + tool definitions
2. **Check response:**
   - If `tool_calls` present → execute tools, append results, continue loop
   - If text message only → extract answer and source, output JSON, exit
3. **Limit:** Maximum 10 iterations to prevent infinite loops

## System prompt strategy

The system prompt instructs the LLM to:

- Use `list_files` to discover wiki files in the `wiki/` directory
- Use `read_file` to inspect specific wiki files and find answers
- Include a `Source: path#section-anchor` line in the response when an answer is found
- Keep answers concise and focused on the documentation

Example system prompt:

```
You are a documentation agent for this repository.
Use list_files to discover wiki files and read_file to inspect them.
Prefer wiki/ for documentation questions.
When you answer, include a line in the format 'Source: path#section-anchor'
if you found the answer in documentation.
Keep answers concise.
```

## Output format

```json
{
  "answer": "Edit the conflicting file, choose which changes to keep, then stage and commit.",
  "source": "wiki/git-workflow.md#resolving-merge-conflicts",
  "tool_calls": [
    {
      "tool": "list_files",
      "args": {"path": "wiki"},
      "result": "git-workflow.md\n..."
    },
    {
      "tool": "read_file",
      "args": {"path": "wiki/git-workflow.md"},
      "result": "..."
    }
  ]
}
```

- `answer` (string, required) — the final answer to the user's question
- `source` (string, required) — the wiki section reference (e.g., `wiki/git-workflow.md#resolving-merge-conflicts`)
- `tool_calls` (array, required) — all tool calls made during the agentic loop, each with `tool`, `args`, and `result`

## Security

Path resolution uses `safe_resolve()` to ensure tools cannot access files outside the project directory:

- Resolves the candidate path relative to `PROJECT_ROOT`
- Validates that the resolved path is either `PROJECT_ROOT` itself or a descendant
- Returns an error if the path escapes the project root (e.g., `../../etc/passwd`)
