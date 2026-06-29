# Day 16 — `child_process`: spawn, exec, execFile, fork

> **Why this day matters:** Yesterday covered scaling Node ITSELF (cluster) and offloading JS computation (Worker Threads). Today covers a different need entirely: **running other programs from within Node** — shell commands, command-line tools like `ffmpeg` or `imagemagick`, Python scripts, or even other Node.js scripts as fully separate processes. This comes up in real backend work more than people expect (image/video processing pipelines, calling CLI tools, running isolated scripts), and interviewers use this topic to check if you understand the four methods' real differences, not just that they all "run something."

---

## 1. Background: why does Node need this at all?

Node.js is good at many things, but some tasks are genuinely better handled by **existing, specialized external programs** rather than reimplementing their logic in JavaScript — video transcoding (`ffmpeg`), image manipulation (`imagemagick`), PDF generation tools, machine learning scripts written in Python, or simply running shell utilities (`git`, `ls`, custom bash scripts). The `child_process` module is Node's way of spawning and communicating with these external processes from your application code.

**The four methods, and the one-sentence distinction for each (memorize this table — it's asked directly, often as "explain the difference between spawn and exec"):**

| Method | Returns output as | Best for | Runs through a shell? |
|---|---|---|---|
| `spawn` | A stream (chunks over time) | Long-running processes, large output, streaming data | No (by default) |
| `exec` | A single buffered string (all at once) | Short commands with small, simple output | Yes |
| `execFile` | A single buffered string (all at once) | Running a specific executable file directly, without shell features | No |
| `fork` | Special — message-passing | Spawning another **Node.js** script specifically, with built-in IPC | N/A (Node-specific) |

---

## 2. `spawn` — for streaming, long-running processes

### Background: why does it return a stream instead of just the final output?

Recall Day 5 — streams exist precisely because some operations produce data over time, and you don't want to wait for everything to finish before doing anything with it, nor hold a huge amount of output in memory at once. `spawn` is built around this same philosophy: it gives you the child process's output as a **stream**, which is exactly right for long-running processes (a video conversion that takes minutes, a process that produces gigabytes of output, a process you want to react to incrementally).

```javascript
const { spawn } = require('child_process');

// Runs 'ls -la' as a child process (Linux/Mac example -- use 'dir' on Windows)
const child = spawn('ls', ['-la']); // command + array of arguments (NOT one combined string)

// child.stdout and child.stderr are READABLE STREAMS (Day 5 concept,
// directly applicable here) -- data arrives in chunks as the process
// produces it, not all at once at the end.
child.stdout.on('data', (chunk) => {
  console.log(`stdout chunk: ${chunk.toString()}`);
});

child.stderr.on('data', (chunk) => {
  console.error(`stderr chunk: ${chunk.toString()}`);
});

// 'close' fires when the process has fully exited AND all stdio
// streams have been closed -- the safe point to know everything is done
child.on('close', (exitCode) => {
  console.log(`Child process exited with code ${exitCode}`);
});

child.on('error', (err) => {
  console.error('Failed to start child process:', err);
});
```

### A real-world example: piping video conversion progress with ffmpeg

```javascript
const { spawn } = require('child_process');

function convertVideo(inputPath, outputPath) {
  return new Promise((resolve, reject) => {
    // ffmpeg writes its PROGRESS output to stderr (not stdout) by
    // convention -- a real, slightly surprising detail about that
    // specific tool that you'd learn from actually using it.
    const ffmpeg = spawn('ffmpeg', ['-i', inputPath, outputPath]);

    ffmpeg.stderr.on('data', (chunk) => {
      console.log('ffmpeg progress:', chunk.toString()); // streamed progress updates as they happen
    });

    ffmpeg.on('close', (code) => {
      if (code === 0) resolve('Conversion complete');
      else reject(new Error(`ffmpeg exited with code ${code}`));
    });
  });
}
```

**Why `spawn` deliberately does NOT use a shell by default:** passing arguments as a separate ARRAY (`['-la']`) rather than as one combined string avoids a category of **shell injection vulnerabilities** — if you build a command by string-concatenating user input into a shell command string, a malicious input like `"; rm -rf /"` could be interpreted as an ADDITIONAL shell command. Passing arguments as a discrete array, with no shell interpretation, avoids this entirely, because the arguments are passed directly to the program, never interpreted as shell syntax.

---

## 3. `exec` — for short commands with small, simple output

```javascript
const { exec } = require('child_process');

// exec takes the ENTIRE command as ONE STRING, and runs it through a
// shell (e.g., /bin/sh on Linux/Mac) -- this means shell features like
// pipes (|), wildcards (*), and environment variable expansion ($HOME)
// actually work, unlike with spawn's default behavior.
exec('ls -la | grep ".js"', (err, stdout, stderr) => {
  if (err) {
    console.error('Error:', err);
    return;
  }
  console.log('Output:', stdout); // the ENTIRE output, buffered, all at once
});
```

**The real danger with `exec` — a genuine, important security point:**

```javascript
// NEVER DO THIS: if 'userInput' comes from a client request and is
// directly concatenated into the command string, an attacker could
// inject arbitrary shell commands -- this is COMMAND INJECTION, a
// serious, real vulnerability class (similar in spirit to SQL
// injection from Day 11, but executing OS-level commands instead of
// database queries).
exec(`some-tool --name=${userInput}`, callback);
// If userInput is: "x; rm -rf /important-data"
// The shell would interpret this as TWO separate commands!

// THE FIX: use 'spawn' or 'execFile' with arguments passed as a
// SEPARATE ARRAY instead of concatenated into a shell string -- this
// closes the injection vector entirely, since arguments are passed
// directly to the program without shell interpretation.
spawn('some-tool', [`--name=${userInput}`]);
```

**Interview tip:** if asked "what's a security concern with `child_process`," command injection via `exec` with unsanitized user input is the precise, correct answer — directly parallel to SQL injection (Day 11) and NoSQL injection (Day 13), just at the OS command level instead of the database level. This pattern-matching across different injection types is a strong way to demonstrate broad security understanding in an interview.

**Another practical limitation of `exec`:** it has a default **maximum buffer size** (`maxBuffer`, default around 1MB) for collected stdout/stderr — if a command produces MORE output than that, `exec` will throw an error and kill the process. This is exactly why `spawn` (streaming, no inherent buffer size limit since you process chunks as they arrive) is preferred for anything with large or unpredictable output volume.

---

## 4. `execFile` — like `exec`, but without the shell

```javascript
const { execFile } = require('child_process');

// Runs a SPECIFIC executable file directly, with arguments as an array
// (like spawn), but STILL buffers all output into a single callback
// (like exec) -- it's a hybrid of the two.
execFile('node', ['--version'], (err, stdout, stderr) => {
  if (err) {
    console.error('Error:', err);
    return;
  }
  console.log('Node version:', stdout.trim());
});
```

**Why choose `execFile` over `exec`:** since it doesn't go through a shell, it avoids shell injection risks (like `spawn`) while still giving you `exec`'s convenience of buffered output for short-running commands. **Why choose it over `spawn`:** when you genuinely don't need streaming and just want the simpler "give me a callback with the final result" pattern, without the shell-related risk of `exec`. Knowing this exact tradeoff (shell-safety of `spawn`/`execFile` vs. the buffered convenience of `exec`/`execFile`) is what separates a memorized answer from a real understanding.

---

## 5. `fork` — specifically for spawning other Node.js processes

### Background: what makes this different from just `spawn`-ing `node script.js`?

You COULD use `spawn('node', ['worker-script.js'])` to run another Node script as a child process — but `fork` is a **specialized convenience method built specifically for Node-to-Node communication**, automatically setting up an **IPC (Inter-Process Communication) channel** between the parent and child, letting them exchange structured JavaScript messages directly, instead of just raw stdout/stderr text.

```javascript
// parent.js
const { fork } = require('child_process');

const child = fork('./child-script.js'); // runs child-script.js as a SEPARATE Node process

// Sending a structured message to the child -- NOT just a string, an
// actual JS object, automatically serialized/deserialized for you.
child.send({ task: 'calculate', payload: { number: 42 } });

// Receiving messages back from the child
child.on('message', (msg) => {
  console.log('Received from child:', msg);
});

child.on('exit', (code) => {
  console.log(`Child process exited with code ${code}`);
});
```

```javascript
// child-script.js
process.on('message', (msg) => {
  console.log('Child received:', msg);

  if (msg.task === 'calculate') {
    const result = msg.payload.number * 2;
    process.send({ result }); // sends a message BACK to the parent
  }
});
```

### `fork` vs Worker Threads (Day 15) — a question interviewers sometimes ask to test if you confuse them

This is genuinely a common point of confusion, so be precise: both let you run separate JS execution concurrently, but `fork` creates a full **separate OS process** (heavier, fully isolated memory, like `cluster`'s workers), while Worker Threads create a **thread within the SAME process** (lighter weight, can share memory via `SharedArrayBuffer`). You'd choose `fork` when you want full isolation (a crash in the child doesn't affect the parent process at all, and you genuinely want a separate Node.js runtime instance — e.g., running a completely separate microservice-like script), and Worker Threads when you want lighter-weight concurrent computation within the same application process.

**A genuinely useful real-world `fork` pattern:** offloading a CPU-heavy task to a forked child process specifically so a crash or memory issue in that task **cannot bring down your main API server process** — since they're fully separate processes, a crash in the child is isolated, whereas an uncaught exception in a Worker Thread, if not handled carefully, has more potential to affect shared process state.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your app needs to convert uploaded videos to a different format — how would you approach this?"** → `spawn` with `ffmpeg`, streaming progress via stderr, NOT loading the whole video into memory, and likely combined with a background job queue (Day 19) so the upload request doesn't block waiting for a potentially long conversion.
- **"You're building a feature that runs a user-influenced shell command (e.g., a custom build/deploy tool) — what's your biggest security concern?"** → Command injection if using `exec` with concatenated user input; mitigate by using `spawn`/`execFile` with arguments passed as a discrete array, and ideally validating/allowlisting inputs strictly rather than passing them through unchecked.
- **"A child process you spawned seems to hang forever sometimes — what would you check?"** → Whether you're listening to BOTH `stdout` and `stderr` (a process can block if its output buffer fills up and nothing is reading it, especially relevant for `spawn`'s stream-based approach), and whether you've set a timeout/kill mechanism for processes that should have a maximum reasonable runtime.
- **"When would you fork a separate Node process instead of using a Worker Thread for a CPU-heavy task?"** → When you want full process isolation (a crash doesn't affect the main app at all) or need to run genuinely separate logic that benefits from being its own independent Node runtime instance, accepting the higher memory/overhead cost of a full process versus a thread.

---

## 7. Hands-on practice for today

```javascript
// practice-day16-spawn.js
const { spawn } = require('child_process');

const child = spawn('node', ['--version']); // spawns another node process just to check its version

child.stdout.on('data', (data) => console.log(`stdout: ${data}`));
child.stderr.on('data', (data) => console.error(`stderr: ${data}`));
child.on('close', (code) => console.log(`Process exited with code ${code}`));
```

```javascript
// practice-day16-fork-parent.js
const { fork } = require('child_process');

const child = fork('./practice-day16-fork-child.js');

child.send({ task: 'square', number: 7 });

child.on('message', (msg) => {
  console.log('Parent received:', msg);
  child.kill(); // cleanly terminate the child once we have our answer
});
```

```javascript
// practice-day16-fork-child.js
process.on('message', (msg) => {
  if (msg.task === 'square') {
    process.send({ result: msg.number * msg.number });
  }
});
```

Run the fork example (`node practice-day16-fork-parent.js`) and confirm you get `{ result: 49 }` back — this proves the IPC message-passing round trip is working correctly between two genuinely separate Node processes.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What's the key difference between `spawn` and `exec`?**
   A: `spawn` returns output as a stream, suited for long-running processes or large/unpredictable output volumes, and doesn't go through a shell by default. `exec` runs the command through a shell (enabling pipes/wildcards) and buffers the entire output into a single callback, suited for short commands with small output — but it has a default max buffer size and shell-related injection risk.

2. **Q: What's a security risk with `child_process`, and how do you mitigate it?**
   A: Command injection — if user input is concatenated directly into a shell command string passed to `exec`, malicious input can inject additional shell commands. Mitigate by using `spawn`/`execFile` with arguments passed as a separate array (bypassing shell interpretation entirely) instead of building one command string.

3. **Q: What does `fork` give you that plain `spawn` doesn't?**
   A: `fork` is specifically for running other Node.js scripts, and automatically sets up an IPC channel allowing structured JavaScript message-passing (`.send()`/`.on('message')`) between parent and child, rather than just raw stdout/stderr text streams.

4. **Q: What's the difference between `fork` and Worker Threads?**
   A: `fork` creates a full separate OS process with completely isolated memory (heavier, but a crash is fully isolated from the parent). Worker Threads create a thread within the SAME process (lighter weight, and can share memory via `SharedArrayBuffer`), suited for lighter-weight concurrent computation rather than full process isolation.

5. **Q: Why might `exec` fail on a command that works fine when you run it directly in your terminal?**
   A: Likely exceeding `exec`'s default max buffer size for collected output, or a difference in shell/environment context between your interactive terminal and the one `exec` spawns — worth checking both possibilities.

6. **Q: Why does `execFile` exist if we already have both `spawn` and `exec`?**
   A: It's a middle ground — buffered, callback-based output like `exec` (convenient for short-running commands), but without going through a shell like `spawn`, avoiding shell injection risk while keeping the simpler callback pattern instead of stream handling.

---

## Tomorrow (Day 17) Preview
We start **Redis** — what it actually is, why it's so fast (in-memory data structure store), common data structures (strings, hashes, lists, sets), and the most common real-world use case: **caching**. This directly continues yesterday's cluster lesson — Redis is precisely the shared, external store that solves the "workers don't share memory" problem you saw on Day 15.
