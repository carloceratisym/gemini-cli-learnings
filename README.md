# Gemini CLI Learnings

Inspired by [Addy Osmani's CLI tips](https://github.com/addyosmani/gemini-cli-tips), this is a collection of learnings by me and [Fabio Sirna](https://github.com/fabio-symphony) through our journey of Gemini CLI.

## Table of Contents

- [Ephemeral Agent Sandboxing](#ephemeral-agent-sandboxing)
- ["Secure YOLO" with Dynamic Configuration](#secure-yolo-with-dynamic-configuration)
- [Headless Configuration with `GEMINI_SETTINGS_FILE`](#headless-configuration-with-gemini_settings_file)
- [Build Resilient Agents with Retries](#build-resilient-agents-with-retries)
- [Heal Imperfect LLM Output with a JSON Repair Function](#heal-imperfect-llm-output-with-a-json-repair-function)
- [Stream Large, Dynamic Prompts via `stdin`](#stream-large-dynamic-prompts-via-stdin)


## Ephemeral Agent Sandboxing

**Quick use-case:** When automating Gemini CLI for tasks, especially research or generation, run each agent in a temporary, isolated directory. This provides a powerful security and stability sandbox, preventing the agent from accidentally modifying your project files or inheriting unwanted context.

For example, every time a research task is dispatched to Gemini CLI, a temporary directory can be created using Python's `tempfile.TemporaryDirectory()`. The `gemini` subprocess is then executed with its working directory set to this ephemeral folder.

```python
import subprocess
import tempfile

with tempfile.TemporaryDirectory() as temp_dir:
    # The 'cwd' argument is key
    result = subprocess.run(
        ["gemini", "-p", "Your prompt here..."],
        cwd=temp_dir,
        capture_output=True,
        text=True
    )
```

**Why it's a pro-tip:**

1.  **Security:** The agent has no access to your source code, git repository, or other project files. It can't accidentally delete, modify, or read sensitive information. Its world is limited to the empty sandbox.
2.  **State Management:** It guarantees a clean slate for every run. There's no risk of the agent being influenced by leftover files, a `GEMINI.md` from your project, or a previous failed run. This makes your automation stateless and predictable.
3.  **Concurrency:** You can safely run multiple Gemini CLI agents in parallel, as each operates in its own completely isolated sandbox.

This pattern is essential for building robust, multi-tenant, or simply safe applications on top of Gemini CLI.

## "Secure YOLO" with Dynamic Configuration

**Quick use-case:** Gain the speed of `--yolo` mode without the risk. For automated tasks where you need the agent to run without interactive approval, you can dynamically generate a restrictive `settings.json` file that explicitly forbids dangerous actions.

YOLO mode (`--yolo`) is fantastic for automation, but it's risky if the agent has access to tools like `run_shell_command` or `write_file`. The solution is to create a `settings.json` *on-the-fly* inside your ephemeral sandbox that whitelists only the tools you need.

In an application, for instance, before running the `gemini` command, a `settings.json` can be created in the temporary directory with the following content:

```json
{
  "tools": {
    "excludeTools": [
      "run_shell_command",
      "run_interactive_shell",
      "write_file",
      "replace_in_file"
    ],
    "allowedTools": [
      "google_web_search"
    ]
  },
  "checkpointing": false
}
```

**How it works:**

1.  **`excludeTools`:** Explicitly block all tools that could interact with the filesystem or execute code.
2.  **`allowedTools`:** Only permit the `google_web_search` tool, as the agent's only job is to do research.
3.  **`--yolo`:** Now, when run with `--yolo`, the agent will automatically use the search tool without prompting, but it is physically incapable of doing anything harmful.

This "Secure YOLO" pattern gives you the best of both worlds: fully automated, non-interactive execution with strong security guarantees.

## Headless Configuration with `GEMINI_SETTINGS_FILE`

**Quick use-case:** Apply a specific, temporary configuration to a Gemini CLI agent without modifying your project's or home directory's `settings.json`. This is the perfect companion to the Ephemeral Agent Sandboxing and Secure YOLO tips.

Once you've dynamically created a secure `settings.json` inside a temporary sandbox, you need to tell the `gemini` process to use it. The cleanest way to do this is with the `GEMINI_SETTINGS_FILE` environment variable.

```python
import os
import subprocess
from your_app import generate_secure_settings # See previous tip

with tempfile.TemporaryDirectory() as temp_dir:
    # 1. Create the secure settings file in the sandbox
    settings_path = generate_secure_settings(temp_dir)

    # 2. Set the environment variable for the subprocess
    env = os.environ.copy()
    env["GEMINI_SETTINGS_FILE"] = settings_path

    # 3. Run the agent
    result = subprocess.run(
        ["gemini", "--yolo", "-p", "Research topic..."],
        cwd=temp_dir,
        env=env, # Pass the custom environment
        capture_output=True,
        text=True
    )
```

This tells the Gemini CLI process to load its configuration from the specified path, overriding any global or project-level settings. It's a clean, contained way to manage agent configuration for headless, automated runs.

## Build Resilient Agents with Retries

**Quick use-case:** Prevent your automated scripts from failing due to temporary network glitches or API hiccups. When wrapping Gemini CLI in a script, implement a simple retry loop.

Automated tools that rely on network requests will inevitably face transient errors. A script that fails on the first sign of trouble is not robust. A `GeminiCLIConnector`, for example, could include a retry mechanism for this very reason.

The logic is simple: wrap your `subprocess.run` call in a loop. If it fails with a specific, retry-able error (like a network `AbortError` or `GaxiosError`), wait for a second or two and try again.

```python
import time
import logging

def execute_with_retries(prompt, max_retries=2):
    for attempt in range(max_retries + 1):
        try:
            # Your subprocess call to Gemini CLI
            return run_gemini_subprocess(prompt) 
        except Exception as e:
            logging.warning(f"Attempt {attempt + 1} failed: {e}")
            if "Transient Network Error" in str(e) and attempt < max_retries:
                time.sleep(2) # Wait before the next attempt
            else:
                raise e # Max retries reached or a non-retriable error
```

This small addition makes your automation significantly more reliable in real-world conditions.

## Heal Imperfect LLM Output with a JSON Repair Function

**Quick use-case:** When you ask the LLM to generate JSON, it can sometimes produce a slightly malformed or truncated string, which would crash a standard `json.loads()` parser. To make your application robust, create a "healer" function to repair the broken JSON.

An application might need a valid JSON object as its final output. It can be found that occasionally the model's output would be cut off, resulting in a `JSONDecodeError`. A `_repair_json` function is a practical solution.

**The Strategy:**

1.  **Try Parsing:** First, attempt a normal `json.loads()`. If it works, great.
2.  **Balance Brackets:** If it fails, check for mismatched brackets (`[]`) or braces (`{}`). Add the missing closing characters to the end of the string and try parsing again. This fixes unterminated objects or lists.
3.  **Aggressive Trimming:** If it still fails, the JSON is likely truncated mid-key or mid-value. Start removing characters from the end of the string one by one, attempting to parse at each step. If you find a valid JSON structure (a `dict` or `list`) before you've deleted everything, you've successfully recovered a partial, but usable, object.

```python
import json

def repair_json(broken_json_str):
    # (Implementation can be adapted from various online examples)
    # ... returns a dict or None
```

This heuristic approach salvages the output in many common failure cases, preventing your application from crashing and allowing it to proceed with at least some of the generated data. It's a must-have for production-grade LLM applications that rely on structured output.

## Stream Large, Dynamic Prompts via `stdin`

**Quick use-case:** When building agentic wrappers or backend services, you often need to construct large, complex prompts dynamically. Passing these via command-line flags (`-p`) can be clumsy or hit character limits. A more robust pattern is to spawn the `gemini` process and stream the prompt directly to its `stdin`.

For example, you might construct detailed prompts by combining style guides, user requests, and conversation history. These prompts can be very large. They can be fed to Gemini CLI programmatically using the `stdin` stream for maximum flexibility and reliability.

```javascript
import { spawn } from 'child_process';
import os from 'os';

function executeGeminiCli(prompt) {
    return new Promise((resolve, reject) => {
        // 1. Spawn gemini with no prompt arguments
        const geminiProcess = spawn('gemini', [], { 
            cwd: os.tmpdir(), // Run in a temporary directory
            shell: false
        });

        let stdout = '';
        let stderr = '';
        
        geminiProcess.stdout.on('data', (data) => { stdout += data.toString(); });
        geminiProcess.stderr.on('data', (data) => { stderr += data.toString(); });

        geminiProcess.on('close', (code) => {
            if (code === 0) {
                resolve(stdout.trim());
            } else {
                reject(new Error(`Gemini CLI failed with code ${code}. Stderr: ${stderr.trim()}`));
            }
        });

        geminiProcess.on('error', (err) => reject(new Error(`Failed to run gemini process. Is 'gemini' in your PATH? Error: ${err.message}`)));

        // 2. Write the entire dynamic prompt to the process's stdin
        geminiProcess.stdin.write(prompt);
        
        // 3. Close the stream to signal that the prompt is complete
        geminiProcess.stdin.end();
    });
}
```

**Why it's a pro-tip:**

1.  **No Prompt Size Limit:** Command-line arguments have platform-specific length limits. Writing to `stdin` bypasses these entirely, allowing you to send prompts that are megabytes in size if needed.
2.  **Avoids Temporary Files:** This pattern is cleaner and more secure than writing your prompt to a temporary file and passing the path to the CLI. It keeps all data in memory.
3.  **Clean Code Structure:** It promotes a clean separation of concerns. Your application code can focus on building the perfect prompt string using complex logic, and then hand it off to a simple, reliable execution function.
4.  **Enables Complex Agents:** This is the ideal pattern for sophisticated agents that augment prompts with extensive context, such as from a RAG (Retrieval-Augmented Generation) system, before calling the LLM.

When your prompts become more than just a simple question, the `stdin` streaming pattern is the professional-grade solution for invoking Gemini CLI programmatically.