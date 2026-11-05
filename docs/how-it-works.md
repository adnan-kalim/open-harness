# How open-harness works: a beginner's guide

This guide explains what a coding agent harness is, how this one is built, and what the project is trying to prove. You don't need any background in AI agents. Words in **bold** are explained in the glossary at the end.

## 1. What is a harness?

An AI **model** on its own can only do one thing: read text and write text back. It can't open files, run commands or check its own work.

A **harness** is the program around the model that lets it do those things. A useful picture:

- The **model** is a brain that can only talk.
- The **harness** gives it hands (tools to read files and run commands), eyes (it shows the model what happened), a notebook (it keeps track of the conversation) and rules (it decides what the model is allowed to do).

Claude Code, Codex CLI, Aider and OpenHands are all harnesses. With the same model inside, a good harness can solve far more tasks than a bad one. This project is about proving that.

## 2. The basic idea: the loop

Every harness is built around a simple loop:

```
1. Send the task to the model.
2. The model replies. It either asks to use a tool ("read this file") or says it's done.
3. If it asked for a tool, the harness runs it and sends the result back to the model.
4. Go back to step 2.
```

For example:

> **You:** Fix the failing date test.
> **Model:** I'll read the test file. → *asks for* `read("tests/date.test.ts")`
> **Harness:** runs it and sends back the file's contents
> **Model:** The bug is in `formatDate`. → *asks for* `edit(...)`
> **Harness:** makes the edit and reports "done"
> **Model:** Let me run the tests. → *asks for* `bash("npm test")`
> **Harness:** runs it and sends back "all tests pass"
> **Model:** Fixed. The month was off by one. → *no tool request, so the loop ends*

The model never touches your computer directly. It only asks, and the harness does the work.

## 3. The parts of a harness

| Part | What it does |
|---|---|
| **The loop** | Keeps asking the model and running its tools until the task is done. |
| **Model connection** | Sends messages to the AI service and gets replies back. |
| **Tools** | The actions the model can take: run commands, read files, edit files, search. |
| **Context management** | Decides what goes into the model's limited memory (the **context window**): instructions, conversation history, file contents. |
| **Permissions** | Rules about what the model may do on its own and what needs your approval. |
| **Interface** | How you use it: a chat in the terminal, or a single command for scripts. |
| **Logs** | A record of everything that happened in each run. |

## 4. What this project is trying to prove

Lots of projects show that an agent can edit code. This one asks a harder question: **how much does each part of the harness actually help?**

To answer it, the project is built like a science experiment:

- **The loop never changes.** It's the fixed part.
- **Each improvement is a feature** that can be switched on or off.
- **A test runner measures** how many tasks the agent solves under different settings.

The settings are called **profiles**:

| Profile | What's on | Why we run it |
|---|---|---|
| `bare` | No features, just the loop and a bash tool | The starting point (the **baseline**) |
| `full` | Every feature | The best the harness can do |
| `full-minus-X` | Every feature except X | Shows what X adds on its own |

**Why "full minus one"?** If you switch off one feature and the score drops, that feature was helping. This kind of test is called an **ablation**. It also shows things a simpler test misses: two features might each look useful on their own but do the same job, so having both adds nothing.

### The features being tested

1. **Edit tools:** proper read and edit tools, instead of making the model edit files through bash commands.
2. **Error feedback:** after each edit, check the code for errors and show them to the model straight away.
3. **Check before finishing:** if the model tries to stop without running the tests, tell it to run them.
4. **Project map:** give the model a list of files and functions at the start.
5. **Trim long output:** keep the start and end of very long command output, and say what was cut.
6. **Summarize old history:** when the conversation gets too long, summarize the older parts.
7. **Stuck detection:** notice when the model repeats the same failing step, and push it to try something else.

Each one is a guess that it will help. The tests decide whether it really does.

## 5. The folder structure

```
open-harness/
  src/     the harness itself
  evals/   the test setup that measures how good the harness is
  docs/    guides like this one
```

### Inside `src/`

| Folder | What it does | Example |
|---|---|---|
| `providers/` | Talks to the AI services. Each service uses its own message format, so this folder translates them all into one shared format. The rest of the code never needs to know which service it's using. | `openai-compat.ts` covers Groq, Cerebras, OpenRouter and Mistral, which all use the same format |
| `loop/` | The heart of the agent: ask the model, run its tools, repeat. | `agent-loop.ts` |
| `tools/` | The actions the model can take. Each tool has a name, a description the model reads, and the code that does the work. | `bash.ts`, `read.ts`, `edit.ts` |
| `env/` | Where the tools actually run: a temporary folder on your computer, or inside a **Docker** container. | `local-env.ts`, `docker-env.ts` |
| `features/` | The improvements being tested, one file each. Each plugs into the loop at a set point. | `error-feedback.ts` |
| `profiles/` | Named lists of which features are on. | `bare`, `full` |
| `trace/` | Saves a full log (a **trace**) of every run. | `trace-writer.ts` |
| `cli/` | How you start the harness: a single command or a chat. | `oh -p "fix the bug"` |

### Inside `evals/`

| Folder | What it does |
|---|---|
| `tasks/` | Small test projects, each with something to fix or build, plus **hidden tests** the agent never sees. |
| `runner/` | Runs every task with every profile and model. It slows down to stay within free usage limits and can resume if stopped. |
| `report/` | Turns the results into tables and charts. |

## 6. One run, step by step

The example task is "fix the failing date test", run with the `full` profile.

```
 START: cli/ sets everything up
   │
   ▼
 1. Build the starting instructions ........ loop/ + features/
   │
   ▼
 2. Ask the model .......................... providers/
   │
   ├── the model asks for tools ──► 3. Run the tools ............ tools/ + env/ + features/
   │                                  │
   │                                  ▼
   │                               4. Save to history and logs .. loop/ + trace/ + features/
   │                                  │
   │                                  └──► back to step 2
   │
   └── the model says it's done ──► 5. Allow it to finish? ...... features/
                                      │
                                      ├── not yet ──► back to step 2
                                      └── yes ──► END
```

**Start (`cli/`).** You type `oh -p "fix the failing date test" --profile full`. The CLI looks up which features `full` turns on, creates a temporary folder for the agent to work in, connects to the AI service, opens a log file, and starts the loop.

**Step 1: build the starting instructions (`loop/` + `features/`).** These are called the **system prompt**. The loop writes the basics: what the agent's job is, which folder it's in, and which tools it has. Then the switched-on features add to it. For example, *project map* adds a list of the project's files.

**Step 2: ask the model (`providers/`).** The conversation is sent to the AI service. `providers/` translates it into that service's format on the way out, and translates the reply back on the way in.

**Step 3: run the tools (`tools/` + `env/` + `features/`).** If the model asked for a tool, the loop finds it in `tools/` and runs it through `env/`. Then features can adjust the result before the model sees it:
- *Trim long output* shortens a 5,000-line test log.
- *Error feedback* adds any code errors found after an edit.
- *Stuck detection* adds a warning if the model keeps repeating the same failing step.

**Step 4: save to history and logs (`loop/` + `trace/` + `features/`).** The request and result are added to the conversation and written to the log. If the conversation is getting too long, *summarize old history* shortens it. If the agent has used up its step limit, the run stops here.

**Step 5: allow it to finish? (`features/`).** When the model says it's done, features get a chance to object. *Check before finishing* looks at the history, and if the tests were never run, it sends the model back to work: "Run the tests before finishing."

**End.** The log is closed and the final answer is shown. In a test run, the runner then runs the hidden tests to decide pass or fail.

## 7. The loop in code

A simplified version of `loop/agent-loop.ts`:

```ts
async function runAgent(task, profile, provider, env, trace) {
  const features = profile.features;
  const system = features.buildSystemPrompt(basePrompt(env, tools));
  const history = [userMessage(task)];

  for (let step = 0; step < MAX_STEPS; step++) {
    // Step 2: ask the model
    const reply = await provider.complete(system, history, tools);
    history.push(reply);
    trace.log(reply);

    // Step 5: the model says it's done
    if (reply.toolCalls.length === 0) {
      const objection = features.beforeFinish(history);
      if (objection) {
        history.push(userMessage(objection));
        continue;
      }
      return reply.text;
    }

    // Step 3: run each tool the model asked for
    for (const call of reply.toolCalls) {
      let result = await findTool(call.name).run(call.input, env);
      result = features.afterToolResult(call, result, history);
      history.push(toolResult(call, result));
      trace.log(call, result);
    }

    // Step 4: keep the conversation from getting too long
    await features.checkLength(history, provider);
  }
  return "Stopped: step limit reached";
}
```

Notice the lines that start with `features.`. That's where features plug in. With the `bare` profile, those lines do nothing, and what's left is the plain loop. That keeps the comparison fair: the loop is exactly the same in every test, and only the features change.

## 8. How the testing works

**Two sets of tasks:**
1. **Our own small tasks** (about 25): bug fixes, new features and changes across several files, in small TypeScript and Python projects. They run quickly on a normal laptop and are used while building.
2. **SWE-bench Verified Mini:** 50 real bugs from real GitHub projects. It's a well-known **benchmark**, so results can be compared with other people's. Each task runs inside its own Docker container.

**Free models only.** The project uses the free tiers of hosted AI services instead of running models on the laptop, which would be far too slow. Every test runs on two models, one stronger and one weaker. That shows whether a feature helps all models or only some.

**Grading.** Each task runs in a fresh copy of the project. When the agent finishes, the hidden tests run. If they pass, the task counts as solved.

**Being honest about the numbers.** With only 25–50 tasks, luck plays a part. A score of 60% might really be anywhere from about 45% to 75%. So every score is reported with an **error range**, and a feature only counts as helpful if the difference is bigger than that range.

**A fair starting point.** The `bare` score is compared with **mini-SWE-agent**, a well-known minimal agent. If they score about the same, we know the baseline wasn't made weak on purpose to make the features look better.

## 9. How this compares with well-known harnesses

**What's the same:**
- **The loop.** Every major harness has one, and the loops in Claude Code and Codex CLI are fairly simple. Most of the hard work is in the tools and in managing context.
- **The tools.** Ours are close to Claude Code's (Bash, Read, Edit, Write, Glob, Grep). Codex CLI uses a shell plus a patch tool. Aider doesn't use tool requests at all: the model writes its edits as text in a set format, and Aider applies them.
- **Many AI services.** OpenCode, Aider, Cline and Goose support many providers too.
- **Running inside Docker.** OpenHands and SWE-agent do this.
- **Saving a log of every run.** SWE-agent calls these trajectories.
- **Some of the features.** The project map and error feedback ideas come from Aider. Summarizing old history is in Claude Code and OpenHands.

**What's different:**
- **Measuring each feature.** Products like Claude Code build their features in and don't publish scores for each one. The closest match is **SWE-agent**, a research project that switched tools on and off and measured the effect. So this project is closer to a research harness than a product. It builds on that earlier work by testing newer features on several models, including free ones.
- **Things we're skipping for now:** helper agents that work on part of a task, connecting outside tools (MCP), saving and resuming sessions, editor integration, and a full permission system. They're useful in a product but don't help answer the main question.

## 10. Glossary

| Word | Meaning |
|---|---|
| **Ablation** | Switching off one part to see how much it was helping. |
| **API** | The way programs talk to an online service, for example sending a message to an AI model. |
| **Baseline** | The starting score that everything else is compared with. Here, the `bare` profile. |
| **Benchmark** | A standard set of tasks that many people use, so results can be compared. |
| **Context window** | How much text a model can take in at once. Everything it knows about the current task has to fit inside it. |
| **Docker / container** | A sealed, throwaway environment for running code, so it can't affect the rest of the computer. |
| **Eval** | Short for evaluation: a test that measures how well the agent performs. |
| **Feature** | One improvement to the harness that can be switched on or off. |
| **Harness** | The program around a model that lets it use tools and work on tasks. |
| **Hidden tests** | Tests used for grading that the agent never sees, so it can't cheat by reading them. |
| **Model** | The AI itself. It reads text and writes text back. |
| **Profile** | A named set of features that are switched on. |
| **Provider** | A company or service that hosts AI models, such as Google or Groq. |
| **System prompt** | The starting instructions given to the model before the task. |
| **Token** | A small piece of text (about ¾ of a word) that models read and write. Usage limits are counted in tokens. |
| **Tool / tool request** | An action the harness lets the model take. The model asks to use one, and the harness runs it. |
| **Trace** | A full log of one run: every message, tool request and result. |
| **WSL** | Windows Subsystem for Linux: a way to run Linux (here, Ubuntu) inside Windows. Docker runs inside it for this project. |
