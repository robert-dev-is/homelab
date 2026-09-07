# Lessons Learned

## 1. Isolation Should Come Before Autonomy

The most important design decision was creating an environment where the agent could make mistakes safely.

Autonomous tooling becomes much more useful when it can modify real systems, but the cost of a mistake increases at the same time.

The solution was not to remove useful permissions entirely. It was to place those permissions inside systems designed to be disposable.

## 2. The Harness Matters as Much as the Model

Model capability alone did not explain the results.

Hermes provided:

- terminal access
- file operations
- browser automation
- persistent sessions
- retry behavior
- tool orchestration
- context compression
- project workflows

The same model becomes much more capable when it can repeatedly observe, act, test, and correct itself.

## 3. Context Windows Are Not Durable Project Storage

A very large context window helps, but it does not make an indefinitely long software project easy.

Long sessions accumulated:

- old reasoning
- obsolete source
- failed attempts
- repeated tool output
- truncated responses
- compression summaries

Eventually the conversation itself became baggage.

## 4. The Filesystem Is Better Long-Term Memory

The strongest workflow that emerged was:

> persist project state to disk and let fresh sessions rehydrate from the current files.

This allowed an agent to continue large work without carrying the entire historical conversation.

It also created an authoritative source of truth when remembered source and current source disagreed.

## 5. Fresh Sessions Can Be a Feature

Starting a new agent session initially felt like losing continuity.

In practice, a fresh session with:

- original requirements
- current source files
- a short handoff

could be more effective than another round of context compression.

This resembles a developer handoff more than a reset.

## 6. Agent Memory Can Become Stale

During long debugging runs, the model sometimes referred to:

- symbols that no longer existed
- older variable names
- earlier versions of files

The important behavior was not that the mistake happened.

The important behavior was that the agent could search the filesystem, recognize the mismatch, and correct itself.

## 7. "It Runs" Is Not the Same as "It Works"

At several points the agent concluded that the generated game was working because:

- JavaScript was executing
- chunks were generated
- the player object existed
- rendering continued

Manual testing showed that interactive behavior could still be broken.

Functional acceptance criteria need to include real user-visible behavior, not only internal state.

## 8. Automated Tests Dramatically Improve Agent Feedback

Once the agent had a Node-based test harness and runtime telemetry, debugging became much more systematic.

The agent could reason from:

- pass/fail counts
- state changes
- timing results
- feature-specific assertions
- logs

rather than guessing from source alone.

## 9. Agent-Generated Tests Also Need Skepticism

An autonomous agent can write incorrect tests or make invalid assumptions inside a test harness.

Tests remain software.

When results looked suspicious, the agent sometimes had to debug the test itself rather than the application.

## 10. Observability Helps Autonomous Systems

Telemetry became one of the most useful additions to the voxel-game experiment.

When browser vision was unreliable, the agent instrumented the application so it could inspect:

- game-loop progress
- chunk counts
- player state
- runtime failures

This is the same reason observability matters in production infrastructure: systems are easier to troubleshoot when they explain what they are doing.

## 11. Local Inference Changes the Economics of Experimentation

A multi-day autonomous run can consume enormous token volume.

With local inference, the cost becomes primarily:

- electricity
- hardware
- time

rather than a continuously increasing API bill.

That makes it practical to explore long, inefficient, failure-prone workflows that would be uncomfortable to run through premium cloud APIs.

## 12. Human Oversight Still Matters

The agent performed substantial autonomous work, but human judgment remained important for:

- deciding when a session had become too bloated
- starting fresh handoffs
- judging whether the application actually worked
- distinguishing environment limitations from application bugs
- setting security boundaries
- deciding when to stop or redirect an experiment

The strongest workflow was not fully manual or fully autonomous.

It was a controlled collaboration between:

- human-defined boundaries
- agent execution
- automated testing
- real system feedback

## Conclusion

The project changed my view of local autonomous agents.

They are capable of much more than simple chat or one-shot code generation, but useful autonomy depends on infrastructure around the model:

- isolation
- persistence
- observability
- testing
- recoverability
- constrained access
- good handoffs

Those supporting systems are what make long-running autonomous work practical rather than merely impressive.
