You are making two changes to the Anthem codebase to support a new "lean channel-only mode" for lightweight agents like Manager that only need channel communication (no issue tracker, no orchestrator, no workspaces).

Read these files first:
- CLAUDE.md (project architecture)
- internal/orchestrator/orchestrator.go (HandleUserMessage at line 1132, Run at line 125, tick at line 320, Opts struct at line 71)
- internal/agent/agent.go (AgentRunner interface)
- internal/channel/channel.go (IncomingMessage, OutgoingMessage types)
- cmd/anthem/main.go (createTracker at line 403, orchestrator wiring at lines 118-297)
- C:/Users/rafa/Projects/Manager/docs/plan.md (Manager build plan -- read this for full context on what we're building and why)

## CHANGE 1: Add handleLeanMessage to orchestrator.go

Add a new method handleLeanMessage on *Orchestrator that provides a fast, lightweight path for answering channel messages via `claude -p` (print mode) instead of the full orchestrator pipeline.

### Where it gets called (two places in HandleUserMessage):

1. At the TOP of HandleUserMessage (line 1132), after the ack broadcast but BEFORE the tracker.ListActive call: detect messages containing `[system:status]` and route them through handleLeanMessage even when orchAgent is present. This gives every agent a fast /status response.

2. At the existing orchAgent nil check (line 1159): instead of returning "Orchestrator agent is not enabled", call handleLeanMessage. This is the path Manager uses for ALL messages.

So the flow becomes:

```go
func (o *Orchestrator) HandleUserMessage(ctx context.Context, msg channel.IncomingMessage) {
    // ... existing ack broadcast ...

    // Fast path for lightweight status queries
    if strings.Contains(msg.Text, "[system:status]") {
        o.handleLeanMessage(ctx, msg)
        return
    }

    // ... existing ListActive call ...

    // Lean mode fallback when orchestrator is disabled
    if o.orchAgent == nil {
        o.handleLeanMessage(ctx, msg)
        return
    }

    // ... rest of existing orchestrator path unchanged ...
}
```

### handleLeanMessage implementation:

The method should:

1. Build a prompt string from the user message text. If the message has file attachments, include them using the existing buildUserMessageContext helper.

2. Prepend the project context (o.projectCtx.ProjectSummary -- this is the CLAUDE.md content) if available, so the agent knows what project it's working on.

3. Invoke the claude CLI in print mode. Use exec.Command to run: `claude -p "<prompt>" --output-format=text`
   - Set the working directory to the workspace root (or cwd)
   - Pipe stdout and stream each chunk back as StreamDelta frames via o.channelMgr.Broadcast
   - The command to build is: `exec.Command(o.cfg.Agent.Command, "-p", prompt, "--output-format", "text")`

4. After the command completes, send a StreamDone frame.

5. Send the full accumulated response as a res frame (for chat history).

6. Record an audit event.

Important constraints:

- Do NOT use the AgentRunner.Run interface -- that's for the full executor pipeline with sessions. Use exec.Command directly for the simple -p mode.
- DO stream output in real-time as it arrives from stdout (use bufio.Scanner or similar).
- DO handle errors gracefully -- if claude -p fails, send an error message as a res frame.
- Keep it simple -- ~80-120 lines. No action parsing, no contract validation, no session management.

## CHANGE 2: Make tracker optional in main.go

In cmd/anthem/main.go, the createTracker function (line 403) returns an error for unknown tracker kinds. When tracker.kind is empty string (no tracker configured), it hits the default case and fails.

Changes needed:

1. In createTracker: add a case for empty string that returns nil, nil (no tracker).

2. In the run command (around line 118): after createTracker, if trk is nil, skip the polling loop. The orchestrator's Run method calls tick which calls o.tracker.ListActive -- this will panic with a nil tracker.

   The cleanest approach: in Orchestrator.Run (orchestrator.go line 125), if o.tracker is nil, skip the ticker/polling loop entirely but still block on ctx.Done for graceful shutdown. The channel listener (StartChannelListener) is started separately and will still work.

   In orchestrator.go Run method, wrap the ticker and tick logic:

   ```go
   if o.tracker != nil {
       interval := time.Duration(o.cfg.Polling.IntervalMS) * time.Millisecond
       ticker := time.NewTicker(interval)
       defer ticker.Stop()
       o.tick(ctx)
       // ... existing select loop with ticker ...
   } else {
       o.logger.Info("no tracker configured, running in channel-only mode")
       <-ctx.Done()
   }
   ```

3. In main.go: allow nil tracker to be passed to orchestrator.New -- the Opts.Tracker field is already a pointer-like interface so nil is valid. Make sure workspace.NewManager and other components that might reference the tracker handle nil gracefully (check if they do).

## Testing

Add tests for both changes:

1. Test handleLeanMessage: mock exec.Command or use a test helper. Verify it sends StreamDelta frames, a StreamDone frame, and a final res frame through the channel manager. Test the error case too.

2. Test that HandleUserMessage routes [system:status] messages to handleLeanMessage even when orchAgent is present.

3. Test that Run blocks on ctx.Done when tracker is nil (channel-only mode).

4. Test that createTracker returns nil, nil for empty tracker kind.

Follow the project's existing patterns:

- Table-driven tests
- Interface-based mocks (MockRunner, MockTracker in the codebase)
- No mocking frameworks
- Wrap errors with context: fmt.Errorf("context: %w", err)
- Use log/slog for logging
- No unnecessary comments

Run `go build ./cmd/anthem` and `go test ./... -count=1` after making changes to verify everything compiles and passes.
