# 🇰🇿 Borat Terminal — Kazakhstan OS Error Detection System

> *"I will watch your terminal output. Very nice."*

A cross-platform terminal error notification plugin that plays **Borat Sagdiyev voice clips** whenever your terminal hits an error. Works as a standalone shell wrapper, a VS Code / Cursor extension, and includes a live TUI monitoring dashboard.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Verify & Test](#verify--test)
- [Usage](#usage)
  - [Shell Wrapper Mode](#shell-wrapper-mode)
  - [Live Dashboard](#live-dashboard)
  - [VS Code / Cursor Extension](#vs-code--cursor-extension)
- [Sound File Map](#sound-file-map)
- [Generating Missing MP3s](#generating-missing-mp3s)
- [Configuration](#configuration)
- [Error Categories & Patterns](#error-categories--patterns)
- [Platform Audio Requirements](#platform-audio-requirements)
- [Troubleshooting](#troubleshooting)
- [The 24 Error Scripts](#the-24-error-scripts)
- [File Layout](#file-layout)
- [License](#license)

---

## What It Does

Every time your terminal produces a recognisable error — a permission denial, a failed build, a segfault, a missing command — Borat Sagdiyev speaks. The system:

- Wraps your existing shell in a transparent PTY so it intercepts **all** terminal output
- Matches output against **60+ regex patterns** across **6 error categories**
- Plays the category-appropriate Borat MP3 via your OS's native audio player (non-blocking, cooldown-gated)
- Logs every detected error to `~/.config/borat-terminal/borat.log`
- Provides a real-time curses TUI dashboard for monitoring and sound testing

No shell plugins. No monkey-patching. Works with bash, zsh, fish, or any other shell, in any terminal, in any IDE.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR TERMINAL                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   PTY MASTER (Parent)                    │  │
│  │                                                          │  │
│  │   stdin ──► PTY slave ──► your shell (bash/zsh/fish)    │  │
│  │                │                                         │  │
│  │                ▼                                         │  │
│  │         Output interceptor                               │  │
│  │                │                                         │  │
│  │        ┌───────┴───────┐                                 │  │
│  │        │ Error Detector │  ← 60+ regex patterns          │  │
│  │        └───────┬───────┘    6 categories                 │  │
│  │                │                                         │  │
│  │       Match?   ├─ No  → stdout (transparent passthrough) │  │
│  │                │                                         │  │
│  │               Yes                                        │  │
│  │                │                                         │  │
│  │        ┌───────▼───────┐                                 │  │
│  │        │ Sound Resolver │  ← Maps category → MP3 file    │  │
│  │        └───────┬───────┘                                 │  │
│  │                │                                         │  │
│  │        ┌───────▼───────┐                                 │  │
│  │        │  Audio Player  │  ← Detached OS process         │  │
│  │        │  afplay/mpg123 │    Non-blocking                 │  │
│  │        └───────────────┘    Cooldown-gated               │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

- **PTY wrapper, not a shell plugin** — works with any shell, in any IDE terminal, transparently
- **Detached audio process** — playback never blocks your terminal
- **Per-category sounds** — `permission_denied` plays a different clip than `build_failed`
- **Smart cooldown** — won't fire 40 times during a Gradle failure cascade
- **False-positive filter** — `error.ts`, `console.error()`, `0 errors` won't trigger it

---

## Prerequisites

| Requirement | macOS | Linux | Windows (WSL) |
|---|---|---|---|
| Python 3.8+ | ✓ Built-in | ✓ Usually present | ✓ With WSL |
| Audio player | `afplay` ✓ Built-in | `mpg123` (auto-installed) | PowerShell MediaPlayer ✓ |
| Node.js | Only for VS Code ext | Only for VS Code ext | Only for VS Code ext |

> **Linux:** The installer auto-detects and installs `mpg123` on Debian/Ubuntu/Fedora/Arch if you have `sudo`. If it fails, run `sudo apt install mpg123` manually.

---

## Installation

### One command

```bash
git clone https://github.com/KovachaKz/Borat_terminal.git
cd Borat_terminal
bash install.sh
```

### With your own MP3 files

```bash
bash install.sh --sounds-dir /path/to/your/mp3s
```

### IDE extensions only

```bash
bash install.sh --ide vscode    # VS Code only
bash install.sh --ide cursor    # Cursor only
bash install.sh --ide all       # Both
```

### What the installer creates

```
~/.config/borat-terminal/
├── config.json           ← your configuration
├── borat_daemon.py       ← core detection engine (copied here)
├── borat_dashboard.py    ← TUI dashboard (copied here)
├── borat.log             ← error history
└── sounds/
    ├── permission_denied.mp3
    ├── command_not_found.mp3
    ├── connection_timeout.mp3
    ├── build_failed.mp3
    ├── crash.mp3
    └── generic_error.mp3

~/.local/bin/
├── borat                 ← CLI entry point
└── borat-dash            ← dashboard entry point
```

> **The installer asks two questions during setup:**
> 1. Whether to add `~/.local/bin` to your PATH automatically — answer **Y**
> 2. Whether to install the shell hook for background exit-code monitoring — answer **Y** for the best experience

---

## Verify & Test

After install completes, run the status check:

```bash
borat --status
```

Expected output:

```
═══ BORAT TERMINAL STATUS ═══
  Enabled  : ✓ YES
  Volume   : 0.85
  Cooldown : 3000ms
  Shell    : /bin/zsh
  Config   : ~/.config/borat-terminal/config.json
  Sounds   : ~/.config/borat-terminal/sounds

  Installed sounds:
    ✓ [permission_denied]  → permission_denied.mp3
    ✓ [command_not_found]  → command_not_found.mp3
    ✗ [connection_timeout] → connection_timeout.mp3
    ✗ [build_failed]       → build_failed.mp3
    ✗ [crash]              → crash.mp3
    ✗ [generic_error]      → generic_error.mp3
```

> A `✗` next to a sound category is not a crash — it means that category falls back to any available MP3 until you add a dedicated file. Your two confirmed clips cover all errors in the meantime.

Test each installed sound:

```bash
borat --test permission_denied   # plays the waa-waa-wi-waa clip
borat --test command_not_found   # plays the jaka-mash clip
borat --test crash               # falls back to permission_denied until crash.mp3 exists
```

---

## Usage

### Shell Wrapper Mode

The recommended way to use Borat Terminal. Type `borat` to launch your shell inside the wrapper. Everything works exactly as before — your prompt, aliases, history, colours. Errors just trigger sounds.

```bash
borat

# Now work normally. Try triggering errors:
ls /root/secret           # → Permission denied  → waa-waa-wi-waa plays
foobar                    # → command not found  → jaka-mash plays
cd /nonexistent/path      # → No such file       → plays

# Exit the wrapper
exit
```

**CLI flags:**

```bash
borat                     # Start Borat shell (wraps your default shell)
borat --status            # Show configuration and sound file status
borat --test [category]   # Test a specific sound category
borat --config            # Show full config file path and JSON
borat --help              # Show all options
```

### Live Dashboard

Open the real-time monitoring dashboard in a second terminal window:

```bash
borat-dash
```

The dashboard shows:
- **Live error feed** — timestamped list of every error detected with category and snippet
- **Sound controls** sidebar — file status (✓/✗) for each of the 6 categories
- **Error count bar charts** — per-category totals
- **Rotating Borat quotes** — cycling every 8 seconds
- **Status line** — enabled state, volume, cooldown, total errors caught

**Keyboard controls:**

| Key | Action |
|---|---|
| `↑` / `↓` | Navigate sound categories |
| `Enter` / `Space` | Test selected sound |
| `+` / `-` | Adjust volume live |
| `P` | Pause / resume detection |
| `C` | Clear error feed |
| `R` | Reload config from disk |
| `Q` / `Esc` | Quit |

Minimum terminal size: **70 × 20**.

### VS Code / Cursor Extension

After running `bash install.sh --ide vscode` (or `--ide cursor`), the extension activates automatically on IDE startup. Look for **🇰🇿 Borat** in the bottom-right status bar.

**Command Palette** (`Ctrl+Shift+P` / `Cmd+Shift+P`):

| Command | Description |
|---|---|
| `Borat: Test Error Sound` | Fires a test immediately |
| `Borat: Test Specific Sound Category` | Pick a category from a list |
| `Borat: Toggle On/Off` | Same as clicking the status bar |
| `Borat: Show Error Stats` | How many errors caught this session |
| `Borat: Open Settings` | Open `boratTerminal` settings |

The extension hooks into `onDidWriteTerminalData` and uses `onDidEndTerminalShellExecution` (exit-code fallback) for older VS Code versions. Config changes hot-reload without restarting the IDE.

---

## Sound File Map

The plugin maps 6 error categories to specific MP3 filenames in `~/.config/borat-terminal/sounds/`:

| Filename | Triggers on |
|---|---|
| `permission_denied.mp3` | Permission errors, sudo failures, EACCES |
| `command_not_found.mp3` | Unknown commands, missing modules, `No such file` |
| `connection_timeout.mp3` | Network errors, SSL failures, DNS, ECONNREFUSED |
| `build_failed.mp3` | npm/Gradle/make/rustc/TypeScript failures |
| `crash.mp3` | Segfaults, OOM, panics, NullPointerException, CrashLoopBackOff |
| `generic_error.mp3` | Catch-all fallback for all other errors |

**Currently included in `sounds/`:**

| File | Status | Clip content |
|---|---|---|
| `permission_denied.mp3` | ✅ Ready | *"waa-waa-wi-waa! Hold your horses! Who do you think you are, king of the castle?..."* |
| `command_not_found.mp3` | ✅ Ready | *"jaka-mash! What in the name of Kazakhstan is this? I looked everywhere for this command..."* |
| `connection_timeout.mp3` | ⏳ Generate | See [Generating Missing MP3s](#generating-missing-mp3s) |
| `build_failed.mp3` | ⏳ Generate | See below |
| `crash.mp3` | ⏳ Generate | See below |
| `generic_error.mp3` | ⏳ Generate | See below |

**Additional clips in `sounds/extras/`** — available to assign to any category by renaming and moving:

| File | Suggested use |
|---|---|
| `do-not-try-and-shrink-me-gypsy-i-serious.mp3` | `generic_error` or `permission_denied` alternate |
| `may-your-george-bush-drink-the-blood-of-every-single-man-woman-and-child-of-lraq.mp3` | `crash` alternate |
| `please-give-them-to-me-or-i-will-take-them.mp3` | `build_failed` alternate |
| `i-couid-not-concentrate-on-what-this-oid-man-was-saying.mp3` | `connection_timeout` alternate |
| `and-her-vagina-hang-like-sleeve-of-wizard.mp3` | `generic_error` alternate |

To use any extra clip, rename it and copy it to `~/.config/borat-terminal/sounds/`:

```bash
cp sounds/extras/do-not-try-and-shrink-me-gypsy-i-serious.mp3 \
   ~/.config/borat-terminal/sounds/generic_error.mp3
```

---

## Generating Missing MP3s

The best TTS source for the Borat voice is **[101soundboards.com/tts/747962](https://101soundboards.com/tts/747962)** (Borat SQ voice).

Paste each script below into the TTS field, download the MP3, rename it exactly as shown, and drop it into `~/.config/borat-terminal/sounds/`.

**`connection_timeout.mp3`**
```
I wait for the internet web to reply, but it never come.
It is too slow. Very slow. Like the handicap toilet in my village.
Connection timed out. Error: CONNECTION TIMED OUT HIGH FIVE.
```

**`build_failed.mp3`**
```
The Gradle has fail. Again. Like always.
Build failed. Very not nice.
Error code: BUILD FAILED GREAT SAD.
Chenquyeh.
```

**`crash.mp3`**
```
The computer has make a big dirty on itself.
The memory has dump in the well!
Error Code: FIVE HUNDRED VERY NOT NICE.
Please to not try again.
```

**`generic_error.mp3`**
```
Not success! Something has go very wrong.
I do not know what, but it is very bad.
Error: GENERIC VERY BAD PROBLEM.
High five? No. Not high five.
```

After adding a new file, verify it:

```bash
borat --test connection_timeout
borat --status
```

---

## Configuration

Edit `~/.config/borat-terminal/config.json`:

```json
{
  "enabled": true,
  "volume": 0.85,
  "cooldown_ms": 3000,
  "log_errors": true,
  "show_banner": true,
  "sounds": {
    "permission_denied":  "permission_denied.mp3",
    "command_not_found":  "command_not_found.mp3",
    "connection_timeout": "connection_timeout.mp3",
    "build_failed":       "build_failed.mp3",
    "crash":              "crash.mp3",
    "generic_error":      "generic_error.mp3"
  }
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Master on/off switch |
| `volume` | float | `0.85` | Playback volume, 0.0 – 1.0 |
| `cooldown_ms` | int | `3000` | Minimum ms between sounds. Prevents 40 sounds during a Gradle failure. Set to `0` to disable |
| `log_errors` | bool | `true` | Write detected errors to `borat.log` |
| `show_banner` | bool | `true` | Show Kazakhstan OS banner when `borat` starts |
| `sounds` | object | — | Map of category → MP3 filename. Files must exist in `~/.config/borat-terminal/sounds/` |

**Adding custom error patterns** (daemon config):

```json
{
  "custom_patterns": [
    "MY_CUSTOM_ERROR",
    "\\[FATAL\\]",
    "DEPLOYMENT FAILED"
  ]
}
```

**Recommended cooldown for heavy build tools:**

```json
"cooldown_ms": 8000
```

---

## Error Categories & Patterns

The detection engine runs 60+ compiled regex patterns ordered from most-specific to least-specific. A false-positive filter runs first on every line.

### `permission_denied`
`permission denied` · `access denied` · `operation not permitted` · `EACCES` · `sudo: incorrect password` · `you do not have.*permission`

### `command_not_found`
`command not found` · `No such file or directory` · `cannot find.*command` · `is not recognized as.*command` · `zsh:.*not found` · `ModuleNotFoundError` · `Cannot find module`

### `connection_timeout`
`connection timed? out` · `connection refused` · `network is unreachable` · `no route to host` · `could not resolve host` · `ssl.*error` · `certificate.*expired` · `ECONNREFUSED` · `ETIMEDOUT` · `EHOSTUNREACH`

### `build_failed`
`build failed` · `FAILURE:.*build` · `compilation failed` · `make.*Error \d+` · `gradle.*FAILED` · `npm ERR!` · `yarn.*error` · `pip.*error` · `error\[E\d+\]` (Rust) · `error TS\d+:` (TypeScript) · `SyntaxError:` · `ModuleNotFoundError:` · `ImportError:` · `Cannot find module` · `COMPILE ERROR`

### `crash`
`segmentation fault` · `core dumped` · `killed` · `out of memory` · `java.lang.NullPointerException` · `OutOfMemoryError` · `StackOverflowError` · `fatal error` · `panic:` · `thread.*panicked` · `Abort trap` · `Bus error` · `CrashLoopBackOff` · `OOMKilled` · `process exited with code [1-9]`

### `generic_error` (catch-all)
`^\s*error:` · `^\s*ERROR:` · `FAILED$` · `^\[error\]` · `Traceback \(most recent` · `AssertionError` · `ReferenceError` · `TypeError:` · `ValueError:` · `AttributeError:` · `KeyError:` · `IndexError:` · `RuntimeError:`

### False-positive filter (never triggers sounds)
`error.log` · `error.ts` · `error.js` · `errorHandler` · `console.error(` · `error_rate` · `no errors` · `0 errors` · `without errors` · `fixed.*error` · `catch(` · `rescue ` · `except ` · shell comments (`#`) · caret pointer lines (`^`)

---

## Platform Audio Requirements

| Platform | Player | Ships with? | Install |
|---|---|---|---|
| macOS | `afplay` | ✓ Built-in | Nothing needed |
| Linux | `mpg123` | Usually not | `sudo apt install mpg123` (auto on Debian/Ubuntu/Fedora/Arch) |
| Linux (alt) | `paplay` | Usually with PulseAudio | — |
| Linux (alt) | `ffplay` | With ffmpeg | `sudo apt install ffmpeg` |
| Windows (WSL) | PowerShell MediaPlayer | ✓ Built-in | Nothing needed |

The daemon tries `mpg123 → paplay → ffplay` in order on Linux.

---

## Troubleshooting

### No sound plays when an error occurs

1. Run `borat --status` — confirm all expected sound files show `✓`
2. Run `borat --test permission_denied` — if silent, the problem is audio, not detection
3. On Linux: confirm `mpg123` is installed: `which mpg123`
4. Check `volume` in `config.json` is not `0`
5. Check `cooldown_ms` — if set very high you may be in cooldown
6. Check `~/.config/borat-terminal/borat.log` — errors detected will be there even if audio fails

### `borat: command not found` after install

```bash
# Add to PATH for this session
export PATH="$HOME/.local/bin:$PATH"

# Add permanently to your shell rc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc    # zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc   # bash
```

### Sounds play but detection misses errors

Add custom patterns in `config.json` for your specific toolchain:

```json
"custom_patterns": [
  "MY_CUSTOM_ERROR",
  "\\[FATAL\\]",
  "DEPLOYMENT FAILED"
]
```

### Too many sounds during build failures

```json
"cooldown_ms": 8000
```

### VS Code extension not detecting errors

1. Open **Output panel** → select **Borat Terminal** from the dropdown
2. Check for `Hooked into terminal data stream ✓` in the log
3. If missing: your VS Code version may be below 1.85 — update VS Code
4. Run `Borat: Test Error Sound` from the command palette to verify audio works independently

---

## The 24 Error Scripts

The complete Borat-voice terminal error script collection — the creative foundation the MP3 clips are based on, and source material for generating new ones.

---

### Case 1: General System Crash

```
[borat@kazakhstan-os ~]$ run_application.sh
⚠ WARNING NOT SUCCESS! WARNING ⚠

The computer has make a big dirty on itself.
I try to run your program, but it is refuse to work--much like
my brother Bilo when we ask him to pull the plow.

The memory has dump in the well!

Error Code: 500_VERY_NOT_NICE
Please to not try again, or the Ministry of Information will confiscate my sister.
Chenquyeh!
```

---

### Case 2: Permission Denied

```
[borat@kazakhstan-os ~]$ cd /root/secret_files
🛑 waaaa-waaa-wi-waaa! STOP!

Who you think you are? King in the castle? King in the castle?
"Oh, look at me, I have the access to root directory, la-la-la-la-la!"

You do not have the golden key of Kazakh computer!
This folder is protect by the state to keep out gypsy and Uzbek.

Error: ACCESS_DENIED_BY_GOVERNMENT
```

---

### Case 3: Command Not Found

```
[borat@kazakhstan-os ~]$ run python3_script
🤔 jaka-mash? What is this "python3_script"?!

I look everywhere for this command.
I search in Astana, I search in US and A, I even search under
the floorboards of my neighbor Nursultan Tulyakbay--he is pain in my assh.

It does not exist. It is a fake! Like the Kazakh space program.

Error: COMMAND_NOT_FOUND_GREAT_SAD
```

---

### Case 4: Connection Timed Out

```
[borat@kazakhstan-os ~]$ ping google.com
PING google.com (142.250.190.46) ...

I wait for the internet web to reply, but it never come.
It is too slow. Very slow. Like the handicap toilet in my village.

Are you sure the wire is plug into the wall?
Or did my wife trade the copper cable for a very nice goat?
(She did a good deal).

Error: CONNECTION_TIMED_OUT_HIGH_FIVE
```

---

### Case 5: Segmentation Fault

```
[borat@kazakhstan-os ~]$ ./my_c_program
SEGMENTATION FAULT (core dump)

Wawaweewa! The program has step on a piece of memory that does not belong to him!
This is like when Nursultan Tulyakbay try to sit in MY chair at the town meeting.
That chair is MINE. You cannot sit there.

The program also cannot sit there. Now it is dead.
The core has been dump behind the shed with the other cores.
We have many core dump in Kazakhstan. We are number four exporter of core dump in all of Central Asia.

Error: SEGFAULT_PROGRAM_TOUCH_FORBIDDEN_MEMORY
Signal 11 received. The program did not survive.
```

---

### Case 6: Null Pointer Exception

```
[borat@kazakhstan-os ~]$ java -jar MyApplication.jar
Exception in thread "main" java.lang.NullPointerException

Great news! Your program has try to use something that does not exist.
Like my wife's cooking skills. She have none.

The pointer is point to NOTHING. It is empty, like the promise of Uzbekistan tourism board.
The variable say "I am here! I am real!" but when you look... nothing. A ghost. A phantom.
Like the economy of my village.

  at BoratApp.main(BoratApp.java:69)
  at KazakhstanRuntime.execute(Nice.java:420)
  at Government.approve(NotReally.java:0)

Error: NULL_POINTER_SHE_IS_NOT_REAL
```

---

### Case 7: Disk Space Full

```
[borat@kazakhstan-os ~]$ cp movie.avi /storage/
ERROR: No space left on device

I try to put your file on the disk, but the disk say
"No more! I am full! I cannot take any more!"
It is like the stomach of my mother-in-law at the harvest festival.
She eat EVERYTHING. Even the decorations.

Your disk have 0 bytes remaining.
It is more empty than the trophy cabinet of Uzbekistan national football team.

Maybe delete some file? I suggest you start with the photographs of your neighbor's wife.
Very nice pictures, but they take a lot of the space.

Error: DISK_FULL_LIKE_MY_MOTHER_IN_LAW
Available: 0 B / Total: 500 GB (used by Pamela screensaver)
```

---

### Case 8: Stack Overflow

```
[borat@kazakhstan-os ~]$ node recursive_app.js
RangeError: Maximum call stack size exceeded

Your program has call itself so many time that the stack is overflow
like the toilet in Room 3 of Astana Airport.

It go round and round and round, like my sister on the carousel at the fair.
She go for 47 minute and then she is sick on the mayor.

Function call Function call Function call Function call Function call
WHO IS CALLING? IT IS SAME FUNCTION!

This is an infinity loop of sadness.
Please to make your program stop calling itself.
Even I know when to stop calling my ex-wife.

Error: STACK_OVERFLOW_LIKE_AIRPORT_TOILET
```

---

### Case 9: npm Install Failure

```
[borat@kazakhstan-os ~]$ npm install
npm ERR! code ERESOLVE
npm ERR! ERESOLVE unable to resolve dependency tree

Jaka-mash! The npm has made a very big mess.
It try to install the package, but the package want different version of other package,
and THAT package want different version of ANOTHER package,
and now everyone is fighting like my family at wedding.

My uncle want sit next to window. My other uncle ALSO want sit next to window.
They both cannot sit there. This is same problem but with JavaScript.

  peer react@"^18.0.0" from some-library@2.0.0
  peer react@"^17.0.0" from other-library@1.0.0

Have you try turn it off and on again? Or maybe just delete the node_modules.
I hear it is like therapy for developer.

Error: NPM_DEPENDENCY_FAMILY_FIGHT
Total packages: 1,847 (none of them work)
```

---

### Case 10: Git Merge Conflict

```
[borat@kazakhstan-os ~]$ git merge feature-branch
Auto-merging app.js
CONFLICT (content): Merge conflict in app.js

OH NO. Two developer have touch the same file at the same time.
This is like when two man try to marry same woman in my village.
It end badly. Someone lose a finger.

<<<<<<< HEAD
Your code is here. You think it is very nice.
=======
Their code is here. They ALSO think it is very nice.
>>>>>>> feature-branch

Now you must choose. Like King Solomon with the baby,
except the baby is JavaScript and nobody want it.

Error: MERGE_CONFLICT_CHOOSE_YOUR_FIGHTER
Hint: fix conflicts then run "git commit" (or run away)
```

---

### Case 11: Docker Container Crash

```
[borat@kazakhstan-os ~]$ docker-compose up
ERROR: Service 'app' failed to build

The docker container has die. Very sad. We will have small funeral for him behind the server rack.
He was good container. He serve many request.
But then he try to allocate 64 gigabyte of RAM and his heart give out.
He is now with the angels. And by angels I mean /dev/null.

Container "app-1" exited with code 137 (OOMKilled)

The operating system has kill your container because it eat too much memory.
This is like when government put down the town horse because it eat all the hay.

Rest in peace, Container app-1. You were number one container in all of Kazakhstan.

Error: CONTAINER_DEAD_VERY_SAD_FUNERAL
```

---

### Case 12: Python Traceback

```
[borat@kazakhstan-os ~]$ python3 app.py
Traceback (most recent call last):
  File "app.py", line 42, in <module>
    result = process_data(None)
  File "app.py", line 17, in process_data
    return data.split(',')
AttributeError: 'NoneType' object has no attribute 'split'

You try to split the None. You cannot split the None!
The None is like my brother Bilo--you cannot make him do anything.
He just sit there. Being nothing.

"But Borat, I pass the data to the function!"
No you did not. You pass NOTHING. You are liar.
Like Uzbekistan when they say they have better potassium than us. LIES.

Your data is None because somewhere upstream someone forget to return a value.
Find that person. Make them pay.
In Kazakhstan we have word for this: "accountability." (We learn it from American textbook.)

Error: ATTRIBUTE_ERROR_NONE_IS_NOTHING
```

---

### Case 13: SSL Certificate Error

```
[borat@kazakhstan-os ~]$ curl https://api.production.com/data
ERROR: SSL certificate problem: certificate has expired

The security certificate of this website has expire.
It is like passport of my uncle Erkin--technically he is not allow to travel,
but he still try every year.

The certificate say it is valid until 2024.
It is now not 2024. Time has pass. Things have change.
My neighbor have new wife. Your certificate need new date.

Maybe you forget to renew? In Kazakhstan, if you forget to renew your goat license,
the goat become property of the state. Same thing happen here but with encryption.

Error: SSL_EXPIRED_LIKE_UNCLE_PASSPORT
Certificate valid: NOT ANYMORE
Issuer: Let's Encrypt (they did not let you encrypt today)
```

---

### Case 14: Out of Memory

```
[borat@kazakhstan-os ~]$ java -jar BigApplication.jar
java.lang.OutOfMemoryError: Java heap space

The Java has run out of the memory! All of it! Gone!
Like the money in my bank account after my wife discover the Amazon Prime.

Your program ask for more memory, but there is no more memory to give.
The computer has give everything it have. It is exhausted. It is done.
Like me after carrying Bilo up the stairs.

Heap: 4096 MB (used: 4096 MB, free: 0 MB)
GC overhead: 98% of time spent in garbage collection

The garbage collector try so hard to clean, but there is too much garbage.
Like the river behind our village. Government say they will clean it. That was 1997.

Error: OUT_OF_MEMORY_BLAME_JAVA
```

---

### Case 15: 404 Not Found

```
[borat@kazakhstan-os ~]$ curl -X GET https://api.myapp.com/users/999
HTTP/1.1 404 Not Found
{"error": "Resource not found"}

I go to the address you give me, and NOBODY IS HOME.
I knock on the door. Nothing. I look through window. Empty.
Like the promises of politician.

Maybe the resource is move? Maybe it is delete?
Maybe it never exist in the first place and you are make it up,
like the time Nursultan say he win the marathon but actually he take taxi.

The server look at your request and say:
"I do not know this person. I have never meet them. Please to leave."

Error: 404_NOBODY_HOME_VERY_EMBARRASS
```

---

### Case 16: TypeScript Compilation Error

```
[borat@kazakhstan-os ~]$ npx tsc --build
error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

TypeScript is very angry with you. You promise it a number, but you give it a string.
This is BETRAYAL.

It is like when I tell my wife I bring home a sheep for dinner,
but actually I bring home three sheep and a small horse.
She say: "This is not what we agree."
TypeScript also say: "This is not what we agree."

  src/calculator.ts:23:15 - error TS2345
  23     calculateAge("twenty-five")
                      ~~~~~~~~~~~~~

You see? "twenty-five" is not a number. It is a WORD.
A number is like 25. Or 69. These are numbers. Words are for poet and for liar.
Not for function parameter.

Error: TYPESCRIPT_TRUST_BROKEN_CANNOT_FORGIVE
Found 1 error. (But honestly, probably more.)
```

---

### Case 17: Kubernetes CrashLoopBackOff

```
[borat@kazakhstan-os ~]$ kubectl get pods
NAME                     READY   STATUS             RESTARTS   AGE
my-app-7d4b8c6f9-xk2j   0/1     CrashLoopBackOff   47         3h

The pod is crash, then restart, then crash, then restart.
It is like my cousin's marriage. They divorce, they remarry, they divorce, they remarry.
Every time they think "this time will be different." It is never different.

CRASH > RESTART > CRASH > RESTART > CRASH > RESTART

Kubernetes is try so hard to keep your pod alive, but your pod does not WANT to be alive.
It is depressed pod. It need therapy, not restart.

After 47 restart, even Kubernetes is lose patience. And Kubernetes have patience of saint.
Unlike my mother.

Error: CRASHLOOPBACKOFF_INFINITE_DIVORCE
BackOff: 5m0s (maximum). Next try: when hope returns.
```

---

### Case 18: Database Connection Refused

```
[borat@kazakhstan-os ~]$ psql -h localhost -U admin mydb
psql: error: connection refused
Is the server running on host "localhost" (127.0.0.1) and accepting TCP/IP connections on port 5432?

I try to connect to the database, but the database say "Go away. I am not accepting visitor."
It is like the nightclub in Almaty. They see me coming, they lock the door, they turn off the light,
they pretend nobody is inside. But I can HEAR the music, PostgreSQL! I know you are in there!

Maybe the database is not start?
Maybe the port is wrong?
Maybe the database has finally had enough of your unoptimized queries and is take a personal day?

Error: CONNECTION_REFUSED_DATABASE_GHOSTING_YOU
Suggestion: Check if PostgreSQL is running (or if it has filed a restraining order against your application)
```

---

### Case 19: Gradle Build Failed

```
[borat@kazakhstan-os ~]$ ./gradlew assembleDebug
FAILURE: Build failed with an exception.
> Task :app:compileDebugJavaWithJavac FAILED

The Gradle has fail. Again. Like always.
Gradle is the most reliable failure in all of software development.
More reliable than Uzbekistan postal service, and that is saying something
because they lose EVERYTHING.

What went wrong:
Execution failed for task ':app:compileDebugJavaWithJavac'
> Compilation failed; see the compiler error output

The compiler error output? You mean the 4,000 line of red text that scroll past so fast
even the computer cannot read it? THAT output? Very helpful. Thank you, Gradle. High five.

BUILD FAILED in 3m 47s
47 actionable tasks: 46 failed, 1 was confused

Error: GRADLE_FAIL_AGAIN_WHAT_A_SURPRISE
```

---

### Case 20: CORS Error

```
[borat@kazakhstan-os ~]$ # Browser Console Output:
Access to XMLHttpRequest at 'https://api.server.com' from origin 'http://localhost:3000'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present.

AH YES, the famous CORS. Cross-Origin Resource Sharing.
Or as I call it: "Computer Obviously Reject Something."

Your frontend try to talk to your backend, but the browser say
"NO! These two are not allow to speak! They are from different origin!"
It is like Romeo and Juliet, but with HTTP headers and more suffering.

In Kazakhstan, we do not have CORS. If two server want to talk, they talk.
We have freedom. (Well, some freedom. Not too much. Government is watching.)

Error: CORS_POLICY_STAR_CROSSED_SERVERS
Fix: Add 'Access-Control-Allow-Origin: *' header (or just cry. crying also work.)
```

---

### Case 21: Missing Semicolon

```
[borat@kazakhstan-os ~]$ gcc main.c -o program
main.c:42:1: error: expected ';' before '}' token

You forget the semicolon. THE SEMICOLON.
The tiny little dot with the tail. It is so small! How can something so small cause so much pain?
It is like the mosquito. Very small. But it bite you and then you have the malaria and then you are dead.
The semicolon is the mosquito of programming.

The compiler has stop ALL work because of ONE missing character.
The entire program--400 line of beautiful code--refuse to compile
because you are too lazy to press one key. ONE KEY.

Even Bilo can press one key. And Bilo cannot do most thing.

Error: MISSING_SEMICOLON_ONE_JOB_YOU_HAD
Line 42, Column 1. The loneliest semicolon that never was.
```

---

### Case 22: Environment Variable Not Set

```
[borat@kazakhstan-os ~]$ node server.js
Error: DATABASE_URL is not defined in environment

You try to start the server, but you forget to set the environment variable.
It is like try to drive car without putting in the key. The car just sit there.
Looking at you. Judging you.

"But Borat, it work on my machine!"
Yes. YOUR machine. YOUR machine have the .env file.
The server does not have the .env file. The server is naked. It have nothing.
You must dress it with the variable.

  Required: DATABASE_URL        Found: undefined
  Also required: API_KEY        Found: undefined
  Also required: JWT_SECRET     Found: undefined
  Also required: HOPE           Found: undefined

In Kazakhstan we have saying: "A server without environment variable is like a horse without legs."
It is not going anywhere.

Error: ENV_VARIABLE_MISSING_SERVER_IS_NAKED
```

---

### Case 23: Infinite Loop Detected

```
[borat@kazakhstan-os ~]$ python3 process.py
[Processing...]
[Still processing...]
[Very much still processing...]
[It has been 47 minutes...]
[CPU temperature: YES]

WARNING: Process consumed 100% CPU for 47 minutes

Your program is run in circle like donkey tied to pole.
Round and round and round. It go nowhere. It accomplish nothing. Like government committee.

The CPU is now so hot you can cook an egg on it.
In fact, my neighbor IS cooking egg on it. He say "thank you for the free stove."

The fan is spin so fast it achieve liftoff. Your laptop is now a helicopter. Congratulations.

Error: INFINITE_LOOP_DONKEY_ON_POLE
CPU: 100% | RAM: Yes | Fan Speed: HELICOPTER
Time elapsed: 47 min | Progress: 0%
```

---

### Case 24: Dependency Hell

```
[borat@kazakhstan-os ~]$ pip install -r requirements.txt
ERROR: Cannot install package-A==2.0 and package-B==3.1
because these package versions have conflicting dependencies.

Welcome to what the developer call "Dependency Hell."
And they are right. This IS hell.
Even Satan would look at your requirements.txt and say "no thank you, this is too much even for me."

Package A want version 1.0 of Library.
Package B want version 2.0 of Library.
Package C want Library to not exist at all.
Package D want to speak to the manager.

  package-A 2.0 requires library>=1.0,<2.0
  package-B 3.1 requires library>=2.0
  package-C 1.0 requires library==1.5.3.post2.dev7.rc1.final.really.final

You know what have no dependency conflict? Goat.
Goat just need grass and water.
Maybe you should program with goat instead.

Error: DEPENDENCY_HELL_SATAN_REFUSE_ENTRY
```

---

## File Layout

```
Borat_terminal/
├── README.md                        ← you are here
├── borat_daemon.py                  ← PTY shell wrapper + error detection engine (689 lines)
├── borat_dashboard.py               ← curses TUI live dashboard (503 lines)
├── install.sh                       ← one-command installer (293 lines)
├── sounds/
│   ├── permission_denied.mp3        ✅ waa-waa-wi-waa clip
│   ├── command_not_found.mp3        ✅ jaka-mash clip
│   └── extras/                      ← 5 additional clips, not yet assigned to a category
│       ├── and-her-vagina-hang-like-sleeve-of-wizard.mp3
│       ├── do-not-try-and-shrink-me-gypsy-i-serious.mp3
│       ├── i-couid-not-concentrate-on-what-this-oid-man-was-saying.mp3
│       ├── may-your-george-bush-drink-the-blood-of-every-single-man-woman-and-child-of-lraq.mp3
│       └── please-give-them-to-me-or-i-will-take-them.mp3
├── vscode/
│   ├── src/extension.ts             ← TypeScript VS Code / Cursor extension (492 lines)
│   ├── package.json                 ← extension manifest
│   ├── tsconfig.json
│   └── package-lock.json
└── docs/
    └── (screenshots, assets)

Installed at runtime to:
~/.config/borat-terminal/
~/.local/bin/
```

---

## License

MIT — Do whatever you want with it. Very nice.

---

*"Kazakhstan number one exporter of errors. All other countries make inferior bug."*
