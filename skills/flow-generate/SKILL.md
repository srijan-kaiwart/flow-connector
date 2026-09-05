---
version: 1.0.0
name: flow-generate
description: |
  Generate images and videos for free via Google's Flow tool
  (flow.google.com — Nano Banana Pro/2/Lite for images, Omni Flash /
  Veo 3.1 Lite|Fast|Quality for video) using the user's own browser session —
  no API cost. Driven by the `flow-auto` CLI and the Flow Connector
  browser extension.
  Use when: "generate an image with flow", "make an image with nano banana",
  "flow image-to-image", "generate with google flow", "make a video with veo",
  "flow text-to-video", "first frame last frame video", "ingredients to video";
  plus extend (longer clips), merge (montage via ffmpeg), and clean (remove
  the Gemini sparkle mark from your own Flow/Veo generations — also "remove
  watermark", "clean this reference image").
  NOT for: paid image/video APIs, or generation on any site other than
  Google Flow.
argument-hint: "[prompt] [--model <model>] [--image <path>] [--asset <name>] [--duration <s>]"
allowed-tools: Bash
---

# Flow Generate

Generate images and videos on flow.google.com through the Flow Connector extension: single prompts or bulk runs, reusable characters and voices, longer clips by extending, and the finished files renamed and saved to disk. Everything runs in the user's own logged-in Google browser session — no API key, no per-image cost, and real files land in the Downloads folder.

The CLI is `flow-auto`. One binary drives the extension; nothing else needs installing.

## Step 0 — Bootstrap

1. Run `flow-auto status --site flow`. The bridge daemon auto-starts on first use.
   - First time in a session it reports `extension (flow): connected` — say so once, don't repeat it later.
   - `flow-auto: command not found` means the CLI is not installed on this
     machine. Install it and nothing else: `npm install -g flow-connector-cli`
     (needs Node.js 20+; if `npm` is missing too, the user installs Node from
     https://nodejs.org and opens a NEW terminal). Then re-run the check.
2. If it prints `NOT CONNECTED`, or anything at all looks wrong later, run:
     ```bash
     flow-auto doctor --site flow
     ```
   It checks the whole chain — Node, bridge, extension, sign-in, tab visibility,
   window width, plan, ffmpeg, skill — and every line that is not `OK` is followed
   by what to do about it. Exit code is non-zero only when something actually stops
   generation, so you can branch on it. **Read its fix lines to the user rather than
   diagnosing yourself** — it knows things you cannot see from here, like whether the
   running daemon is a stale build.
3. **If it prints `flow tab: HIDDEN`** — stop before generating and tell the user to bring the Flow window to the front (see the arrangement below). This is not optional: this is pure UI/browser automation, not an API — every generation depends on real composer clicks/typing landing on a tab that's actually rendering, and a hidden/minimised/covered tab gets throttled by the browser and either stalls or silently fails. Check this every time you run `status`, and re-check right before a batch.
4. A refusal is a single failed job, not a queue-wide halt — see **Anti-bot** below.
5. If a job fails with `not logged in`, ask the user to log in to their Google account in that browser window and confirm.

The `status` output reports tab visibility live (the extension updates it the instant the window is minimised or covered), so you can catch a stall *before* wasting a generation. With `--json`, the field is `grokVisible` (reused field name — same meaning for both sites): `true` / `false` / `null` = unknown.

## Browser requirement — state this clearly, don't assume it

This is **UI/browser automation, not an API and not a bot** — it drives the real Flow interface with real clicks, typing, and uploads, exactly like a person would, which is also *why* it can't run headless or in a background tab. It needs a `flow.google.com` tab that is:

- **open**, and
- **logged in**, and
- **visible on screen** — not minimised, not fully covered by another window, not in a collapsed window.

That last one is easy to miss and not optional: hidden tabs get throttled by the browser, so a covered window makes generation stall or fail rather than run slowly.

**Tell them HOW, not just what.** Give them the arrangement:

> Put the browser on the left half of your screen and your editor on the right, so both stay visible. Quickest way: click the browser window and press **Win + ←**, then click your editor and press **Win + →**. (On a Mac, drag them side by side; on two monitors, just leave the browser on the second screen.)

Once that layout is set, it survives the whole session. **Whenever `NOT CONNECTED`, `flow tab: HIDDEN`, a timeout, or a stall occurs — stop and instruct the user, never retry silently.** If it stalls a second time, don't repeat the warning — ask directly whether the Flow window is currently visible, because the usual cause is that it got covered again.

**Chrome and Edge behave identically — use whichever is already set up.**
Measured across the same 35 prompts in both: 47.1 vs 47.4 s/image at batch 1,
14.6 vs 14.4 at batch 4, and **zero** refusals in either. Never tell the user
to switch browsers to escape a refusal; it changes nothing.

## Projects

Generation happens inside a Flow *project*. If the tab is sitting on the project gallery, the extension **creates one itself** — you do not have to ask the user to open a project first.

By default a batch reuses whatever project is open. Add `--new-project` to start a fresh one, which is worth doing when a batch is a distinct piece of work the user will want to find again:

```bash
flow-auto generate create flow-text-to-image --prompt "..." --new-project
```

**One project per RUN, not per prompt.** Put `--new-project` on the FIRST
prompt of a batch and nothing else. Every prompt carrying it asks for its own
project, so a 20-prompt batch with the flag on all 20 scatters the run across
20 projects. The rest of the batch lands in whatever project the first one
made.

The flag navigates to the gallery first, so that job reports a retryable
`navigating to a new project` **once** and then runs. That line is expected —
it is the tool changing pages, not a prompt failing, and it is not worth
reporting to the user.

You should see exactly one new project appear. If a run ever leaves several
empty ones behind, that is a bug worth reporting, not something to tidy up by
resubmitting: the extension cannot delete projects, so the user has to remove
them by hand in Flow.

## Speed: you control it, and it corrects itself downward

Prompts are submitted **in batches**: several go into the Flow tab back-to-back, all generate concurrently server-side, everything downloads, then the queue waits before the next batch. These flags on `generate create` set the pacing, and they apply to the whole batch:

| Flag | Default | What it does |
|---|---|---|
| `--batch <n>` | `4` | Outputs submitted back-to-back before any are watched. The main speed control. **Pro only** — on the free plan every batch is 1, whatever you pass. |
| `--delay <range>` | `10-15` | Seconds to wait after a finished batch. `--delay 0` runs batches back-to-back. |

### Running a batch — submit everything, THEN watch

`--wait` blocks until that one job is finished. Run a list of prompts that way
and two things go wrong, both of them badly:

- **Nothing batches.** A batch of 4 needs four jobs sitting in the queue at the
  same moment. Waiting for each one before submitting the next means the queue
  never holds more than one, so a Pro account runs at the free tier's speed —
  ~47s per image instead of ~14.5s.
- **You go silent.** The command owns the turn for as long as the run takes, so
  the user cannot ask you anything, change a prompt, or stop the run. That is
  not acceptable for a run measured in minutes.

So: **submit every prompt without `--wait`** (each command returns a job id
immediately), then watch the queue.

```bash
# submit — fast, non-blocking, one command per prompt
flow-auto generate create flow-text-to-image --prompt "..." --index 1 --folder run1
flow-auto generate create flow-text-to-image --prompt "..." --index 2 --folder run1
flow-auto generate create flow-text-to-image --prompt "..." --index 3 --folder run1

# then watch (cheap, returns immediately — repeat it every so often)
flow-auto generate list
```

While that run drains you are **free, and expected, to keep talking**: answer
questions, prepare the next prompts, edit a script. Check back with
`generate list` between replies. Never sit blocked on a run — and if a single
command genuinely has to block (one long video, an extend chain), run it in the
background rather than in the turn.

`generate list` shows each job's status; the run is over when nothing reads
`queued`, `running` or `downloading`. Then, and only then:

```bash
flow-auto generate list --failed   # this run's holes: number, prompt, reason
```

**Pass `--index <n>` on every prompt of a multi-prompt run.** It is the prompt's
1-based position, and it becomes the file number. Without it each job is
numbered by the order its download happens to land, so one refused prompt
silently shifts every later file up one — prompt 4 saves as "3", and the failed
list no longer matches the folder. With it, a failed prompt leaves a gap and
"3" is simply missing.

```bash
flow-auto generate create flow-text-to-image --prompt "..." --index 1 --folder run1
flow-auto generate create flow-text-to-image --prompt "..." --index 2 --folder run1
```
| `--image-delay <range>` | `1.5-3` | Seconds between the individual generations *inside* one batch. |
| `--speed <1-5>` | `3` | One dial for both delays, when you don't want to name seconds. 1 safest, 5 fastest. |

Every wait is random inside its range — a fixed gap is a rhythm, and a rhythm is
what a bot detector reads. Naming seconds beats `--speed`, and both are clamped
by the extension's **Safe mode**: with it on, nothing you pass can run faster
than what the user set in the panel. They can also type both delays there
themselves, which likewise overrides `--speed`.

```bash
flow-auto generate create flow-text-to-image --prompt "..." --batch 6 --delay 8-12 --image-delay 2-4
```

### Check the plan — it decides how fast you can go

`flow-auto status --site flow` prints it, and nothing else needs to change when a user upgrades. There is no separate setting to flip and no config to edit: the plan comes from the extension on every status call, so a licence activated a minute ago is already reflected.

```
plan: free (0/30 videos, 11/40 images today)   -> one prompt at a time
plan: pro (19 generated today)                 -> batching available
```

- **`plan: free`** — batching is off. Prompts run one at a time, so a list of 8 takes roughly 8x one prompt plus the delays. That is the plan difference, not a bug and not something to work around. If the user asks why a batch is slow, say so plainly and mention that Pro batches.
- **`plan: pro`** — batches. There are no daily limits and the side panel may be closed.

**Pass `--batch` (or just leave the default 4) whatever the plan says, and never
branch on the plan yourself.** The extension re-reads the licence for every
group, so the clamp is applied there, live: on free your 4 becomes 1, and the
moment a licence is activated — mid-run, mid-batch, with nothing restarted —
the very next group runs at 4. An agent that "helpfully" drops to `--batch 1`
because it saw `plan: free` an hour ago is the only thing that can stop that.

You will see this on stderr the first time a command runs after the switch:

```
plan: PRO is now active — batching is on from the next batch onward (default --batch 4, ~3x faster). Nothing to restart.
```

Say it to the user in one line and carry on — do not re-run `status`, do not
resubmit anything, do not restart the run.

### The panel has a pacing mode, and Manual outranks you

The side panel (Pro only — the card is absent on free) has **Auto / Manual**:

- **Auto** — you decide. `--speed`, `--delay`, `--image-delay` and `--batch` are
  taken as given, and the dial is hidden because it would not be doing anything.
- **Manual** — the user's dial is a ceiling. Your delays are floored at theirs
  AND `--batch` is capped by their notch (1x caps at 1, 3x at 4, 5x at 8). Pass
  `--batch 12` on a 2x Manual setting and the run uses 2.

So a batch that runs slower than the flags you passed is not a bug — check
whether they are on Manual before changing anything, and never tell a user to
switch to Auto so your run goes faster. That setting is their risk decision
about their own Google account.

Check it at the start of a session, not before every command.

**Raising these is allowed.** The defaults are a starting point, not a safety rating — pick what the user's situation needs. But understand what you are trading:

- Flow's anti-bot reacts to burst traffic. It is the user's real Google account at risk, not an API key.
- **A refusal is one generation, not a ban.** Flow's wording is *"We noticed some
  unusual activity... You have not been charged for this generation."* It refuses
  that generation and keeps working — the user can generate by hand immediately
  after one. It is reported as a normal failed job.
- **It is not retried.** Re-submitting the same prompt three times in forty
  seconds is exactly the burst that earns the warning, so a refusal is terminal
  for that job. Do not resubmit it by hand either; move on, or slow down.
- **Every refusal doubles your requested delay** for following groups (capped at
  8x), decaying on its own. Too fast corrects itself without you intervening.
- `--delay 0` is honoured until the first refusal. After that the band restarts
  from the 10-15s default and escalates — a zero delay cannot back itself off
  (0 x anything is 0). Nothing halts the queue behind it.

So: start at the defaults, go faster if the user wants throughput, and if refusals
start appearing **tell the user what is happening** rather than pushing harder.

**Measured** (nano-banana-pro, 35 images): batch 1 = ~47s/image, batch 4 =
~14.5s/image. Batch 4 is ~3.2x faster and produced zero refusals, so the
default of 4 is not a risk worth trading away — lowering it buys nothing.

## UX Rules

1. Be concise. No raw JSON or job ids in chat. Deliver the local file path(s) plus a one-line summary.
2. Don't narrate internals ("connecting to bridge", "polling job").
3. Detect the user's language and reply in it; technical flags stay English.
4. **One prompt: `--wait`. More than one: never `--wait`.** See **Running a batch** below — it is the difference between a batch that actually batches and one that runs at a quarter of the speed with the chat frozen behind it.
5. **Other agents sharing this bridge are not your problem.** One bridge serves
   every chat on the machine and they share one Flow tab, but the extension runs
   a single paced dispatch loop over it — the queue is serialised for you. A job
   in `generate list` that you did not submit is somebody else's work, not a
   conflict, not a reason to wait, and not a reason to slow down. Never stop a
   run because another agent's jobs are in the list.
6. Selectors in this extension are unverified until the user runs Healthcheck once (Debug Logs tab) — if a job fails with a `SELECTOR_TIMEOUT` or "not found" error the first time this is used, say so plainly and suggest that step rather than retrying blindly.

## Mode selection

| Task | Mode | Required flags |
|---|---|---|
| Image from text | `flow-text-to-image` | `--prompt` |
| Restyle/remix with reference(s) from disk | `flow-image-to-image` | `--prompt --image <path>` |
| Restyle/remix reusing assets already in the project | `flow-image-to-image` | `--prompt --asset "<name>"` |
| Video from text | `flow-text-to-video` | `--prompt` |
| Video from a start (and optional end) frame | `flow-frame-to-video` | `--prompt --image <start> [--image <end>]` |
| Video from reference images ("ingredients") | `flow-ingredients-to-video` | `--prompt --image/--asset (max 3)` |
| Restyle an existing clip | `flow-video-to-video` | `--prompt --video <file>` (1 clip, 3-10s, omni-flash only) |
| Make one clip longer, consistently | `flow-extend-video` | `--prompt "what happens next"` (see Extend below) |
| Reusable character (portrait + voice + body sheet) | `flow-create-character` | `--prompt --name "<NAME>"` |
| Set or change the voice on a character already open | `flow-set-voice` | `--voice "<preset>"` (free, generates nothing) |
| Rename something already in the project | `flow-rename-asset` | `--asset "<current>" --name "<new>"` (free, generates nothing) |
| Bulk-import a folder of references | `flow-upload-assets` | `--image a.jpg b.jpg …` (free, generates nothing) |

### Characters need the Flow window MAXIMISED — tell the user before you start

Flow decides what to render from the window's WIDTH. In a narrow window it
switches to a mobile layout where two things do not exist at all:

- the **Characters** sidebar entry (the extension works around this one), and
- the voice dialog's **"Customize performance"** field, which is what
  `--voice-style` types into. There is no fallback: the control is not on the
  page.

So before any `flow-create-character` or `flow-set-voice` run, **ask the user to
maximise the Flow window and keep it that way until the character is finished**.
It is a UI restriction of Flow's, not a limitation of this tool.

A job that passes `--voice-style` while the window is narrow is refused up
front, before it spends a generation, and says what to do. Without
`--voice-style` a narrow window is fine.

`--image` and `--asset` fill the same reference slot and share one cap: **10 references total** per prompt. Prefer `--asset` for anything already in Flow — past uploads *and* earlier generations (image or video) are all searchable by name in the composer's + panel, so reusing one costs no upload and no disk file. The name is matched as a substring of the asset's label, so `--asset "typewriter"` is enough for "Vintage typewriter on desk", but it must match something in that panel right now.

**`--image` always uploads.** It does not check whether "that file is already in
the project", because it cannot: Flow renames every upload to its own caption,
so the only thing such a check could ever match is an unrelated asset whose
label happens to contain your filename — and `1.png`, `a.png` or `start.png`
matched something almost every time, attaching the wrong reference and never
uploading the real file. So a file passed twice leaves two copies in the project
library. That is the cheap half of the trade; generating from the wrong
reference was the expensive half. Pass `--asset` when you mean "the one already
in there", and `flow-rename-asset` to give it a name worth referencing.

**Image** — models: `--model nano-banana-pro|nano-banana-2|nano-banana-2-lite`. Aspect ratios: `--aspect-ratio 16:9|4:3|1:1|3:4|9:16`. Outputs: `--outputs 1..4`.

**Video** — models: `--model omni-flash|veo-3.1-lite|veo-3.1-fast|veo-3.1-quality`. Aspect ratio: **16:9 or 9:16 only** (anything else is ignored with a warning). Outputs: `--outputs 1..4`.

Duration depends on the model, and this is not a soft preference:

| Model | Duration | Credits | End frame |
|---|---|---|---|
| `omni-flash` | `--duration 4\|6\|8\|10` | cheapest | start frame only |
| `veo-3.1-lite` / `veo-3.1-fast` | **fixed 8s** — no duration row exists, `--duration` is ignored | ~20 | first **and** last |
| `veo-3.1-quality` | **fixed 8s** | **~100** | first and last |

**`veo-3.1-quality` costs ~5x the other Veo models** — never pick it as a default, and confirm with the user before spending on it. Default to `omni-flash`; step up to `veo-3.1-fast` when the user needs an end frame or better motion. Flow shows the real cost ("Generating will use N credits") in the composer before submitting.

For `flow-frame-to-video` the FIRST `--image`/`--asset` is the start frame and the SECOND is the end frame.

**`flow-video-to-video`** takes ONE clip via `--video` and re-renders it from
your prompt. Flow itself only edits clips of **3-10 seconds** and offers this on
**omni-flash** only, so a longer source has to be cut first (`ffmpeg -t`). The
clip is uploaded like any other reference — it lands in the project library and
is searchable by name afterwards, so a second pass over the same clip can use
`--asset` instead.

**End frames are model-gated:** only `veo-3.1-lite|fast|quality` accept a first AND last frame. `omni-flash` takes a **start frame only** — it accepts the second one in the UI and then generates as though it were never there. The CLI refuses `omni-flash` + 2 frames outright; pick a Veo model when the user wants a first→last transition, or pass a single frame.

## Extend — making ONE clip longer instead of generating a new one

`flow-extend-video` lengthens the clip the user already has open, keeping the same scene, characters and camera — it is not a second generation stitched on afterwards. Two hard requirements, both from the site, not from this tool:

1. **Veo only.** The clip being extended must itself have been generated with a Veo 3.1 model. On an Omni Flash clip Flow shows no Extend item at all, and the job fails saying so.
2. **The clip must be open in Flow's scene editor** — the URL has to read `flow.google.com/project/<id>/edit/<clip>`. Ask the user to click the video in their project first; there is no `--asset` for this. Each extend adds one Veo generation (~8s of footage) and takes about as long as generating the base clip did.

Beats: `--prompt "…"` alone extends once. `--extend 3` repeats that same prompt three times. `--step "…" --step "…"` is a storyboard — one prompt per extension, in order, which is what you want for a scene that develops rather than continues.

An extend permanently lengthens the clip inside Flow, so **it is never retried**. If a chain breaks halfway (say beat 3 of 5), the command still downloads the clip as extended so far and prints a `warning:` line saying where it stopped — read that out, don't silently present it as the full-length result, and don't re-run the same command to "finish it" (that would extend the already-longer clip). Ask the user what they want the remaining beats to be, then run those.

## Characters and voices

`flow-create-character` turns a prompt into a reusable character: portrait, optional Character Info, optional voice, optional multi-angle body sheet. The `--name` is the handle everything else uses — reference it later with `--asset "<NAME>"`. Without `--name` Flow saves it as "Untitled Character" and nothing can address it.

**It will not build the same character twice.** Before generating anything the
job checks the project's Characters grid for that exact `--name`, and if it is
already there the job comes back `done` with:

```
warning: character "NARRATOR" already exists in this project — nothing was generated.
Cast it with --asset "NARRATOR", or pass a different --name to make a second one.
```

Nothing was rendered and nothing was charged. That message is also the
**confirmation** Flow gives you no other way to get: re-running the same create
command is a safe, free way to ask "did it get made?". So stop guessing after a
character run — if you are unsure, run it again and read the answer. The thing
this prevents is a project ending up with three near-identical characters,
none of which is the one the prompts reference.

### Voices are all English-native — `--voice-style` is what fixes that

Flow ships presets only (no upload, no cloning), and every one of them is an English speaker. Give a preset a line in another language and it reads it with an English accent. **Customize Performance** is the fix: free text describing accent, mood, emotion and pace together.

```
--voice "Achird" --voice-style "native Hindi speaker, elderly, slow raspy warm tone"
```

Accent, age, mood and pace all go in that one string — it is free text, not a
list of options, so describe the delivery the way you would to a voice actor.

What Flow actually does with that matters, because it is not "a preset with a setting": typing a performance makes Flow **save a NEW custom voice** built from that preset, and attaches that. The tool drives the whole chain — preset → performance → Voice Name → Save New Voice (a server round-trip, takes a few seconds) → Add to Character. The saved voice then lives in the project's voice list by name and can be reused with `--voice "<that name>"` on the next character, with no `--voice-style` needed.

`--voice-name` names the saved voice. Flow's own default is `"<preset> Custom"`, which collides the moment two characters are built from the same preset — pass a real name whenever more than one custom voice is in play.

### Changing a voice

Flow hides "Select a voice" once a voice is attached; it has to be removed first. `flow-set-voice` does that, on the character **already open in Flow's editor**:

```bash
flow-auto generate create flow-set-voice --voice "Algenib" --voice-style "gruff, weathered, unhurried" --voice-name "Narrator Gruff" --wait
```

`--name` renames the character at the same time, which is how you rescue one left as "Untitled Character". Pass `--character "<current name>"` and it opens the character itself; without it, the character has to already be open in the editor (URL containing `/character/<id>`).

## Commands

`--wait-timeout` is in **minutes** (default 15). Video needs more than the default; images rarely do.

```bash
flow-auto doctor --site flow   # anything wrong? this says what to FIX
flow-auto status --site flow
flow-auto modes
flow-auto generate create flow-text-to-image --prompt "a red fox in snow, cinematic" --model nano-banana-pro --aspect-ratio 16:9 --wait
flow-auto generate create flow-image-to-image --prompt "make it watercolor" --image ./photo.png --model nano-banana-2 --wait
flow-auto generate create flow-image-to-image --prompt "combine these into one scene" --image a.png --image b.png --image c.png --wait
flow-auto generate create flow-image-to-image --prompt "make it watercolor" --asset "Vintage typewriter on desk" --wait
flow-auto generate create flow-image-to-image --prompt "put these together" --asset "Running shoes on track" --image ./logo.png --wait
flow-auto generate create flow-text-to-video --prompt "a paper boat drifting down a rainy gutter" --model omni-flash --duration 4 --aspect-ratio 9:16 --wait --wait-timeout 15
flow-auto generate create flow-frame-to-video --prompt "she turns and smiles" --image ./start.png --model omni-flash --duration 8 --wait --wait-timeout 15
flow-auto generate create flow-frame-to-video --prompt "smooth transition" --image ./start.png --image ./end.png --model veo-3.1-fast --duration 8 --wait --wait-timeout 15
flow-auto generate create flow-ingredients-to-video --prompt "the cat wears the hat in this room" --asset "Tabby cat" --asset "Red hat" --duration 6 --wait --wait-timeout 15
flow-auto generate create flow-video-to-video --prompt "make it look like a hand-painted oil painting, same motion" --video ./clip.mp4 --model omni-flash --wait --wait-timeout 15
flow-auto generate create flow-extend-video --prompt "the lava reaches the treeline" --wait --wait-timeout 20
flow-auto generate create flow-extend-video --prompt "the smoke thickens" --extend 3 --wait --wait-timeout 40
flow-auto generate create flow-extend-video --step "the peak splits open" --step "lava floods the valley" --step "ash blots out the sun" --wait --wait-timeout 40
flow-auto generate create flow-create-character --prompt "an elderly storyteller, warm eyes, plain grey background" --name "NARRATOR" --character-info "sharp tongued, tells long stories" --voice "Achird" --voice-style "elderly, slow raspy warm tone" --voice-name "Narrator Warm" --wait
flow-auto generate create flow-set-voice --character "NARRATOR" --voice "Narrator Warm" --wait
flow-auto generate create flow-rename-asset --asset "Brass diving bell" --name "DIVING_BELL" --wait
flow-auto generate create flow-upload-assets --image ./plates/*.jpg --wait   # bulk import, one upload
flow-auto generate list          # recent jobs (both sites) — also how you poll a running batch
flow-auto generate list --failed # THIS RUN's holes: number, prompt, why it failed
flow-auto generate list --failed --all  # every failure ever recorded (rarely what you want)
flow-auto generate list --failed --run <name>  # exact, from the run's own record
flow-auto generate runs          # named runs this machine remembers
flow-auto generate resume <name> # re-submit only a run's unfinished prompts
flow-auto generate resume <name> --dry-run     # show them, submit nothing
flow-auto generate get <id>      # one job
flow-auto stop --site flow       # halt the queue NOW and cancel everything unfinished
flow-auto reload-extension --site flow  # restart the extension (un-wedge it, or load a new build)
flow-auto queue clear --site flow  # throw away finished/failed history
```

**`stop` is the kill switch — reach for it.** A batch that is going wrong keeps
burning the user's real generations while you think about it. `job.cancel` only
takes one id at a time; `stop` halts dispatch and cancels every pending job in one
call. A generation already in flight in the browser still finishes (there is no
value in half an image). Use it the moment the user says stop, or the moment a run
starts behaving unexpectedly — ask afterwards, not before.

Stdin prompt works: `echo "prompt" | flow-auto generate create flow-text-to-image --wait`.

On success the command prints absolute file paths of the downloaded media — deliver those to the user.

## Remove the Gemini mark

Flow stamps a small semi-transparent Gemini sparkle **inset from the bottom-right corner** — not flush in the corner the way Grok's mark is, and much smaller. `clean --mark flow` knows that geometry; **always pass `--mark flow`** on Flow output, or it will aim at Grok's corner box and miss.

```bash
flow-auto clean in.jpg --mark flow -o out.jpg          # stills  -> LaMa erase (~25s)
flow-auto clean in.mp4 --mark flow -o out.mp4          # video   -> crop (instant)
flow-auto clean in.mp4 --mark flow --mode delogo -o out.mp4
flow-auto clean in.mp4 --mark flow --mode cover --logo my_logo.png -o out.mp4
```

Measured from real Flow downloads — the mark is a **fixed pixel size**, not a fraction of the frame, and the box is picked automatically from whether the input is an image or a video:

| Source | Sparkle | Inset from right/bottom | Verified at |
|---|---|---|---|
| Images (Nano Banana Pro / 2 / Lite) | ~45 px | ~75 px | 1376x768, 1024x1024 |
| Video (Veo 3.1 *and* Omni Flash) | ~47 px | ~96 px | 1280x720, 720x1280 |

Video geometry is identical for Veo and Omni Flash, and identical in 16:9 and 9:16 — orientation doesn't change it.

**The default splits by media type, and you rarely need to override it:**

| Input | Default mode | Cost | Why |
|---|---|---|---|
| **Still** | `inpaint` (LaMa) | ~25s | Erases it outright — no crop, no patch. The mark is small and has real content on all four sides, which is LaMa's best case. |
| **Video** | `crop` | 13.5% zoom, instant | LaMa is ~11s *per frame*; `delogo` is free but its patch **pops visibly on dark or low-texture footage**. Crop's cost is predictable and never leaves an artifact. |

Other modes: `--mode delogo` (zero crop, use when the corner is busy enough to hide the patch — check the result), `--mode cover --logo x.png` (rebrand it instead).

- **`delogo` on video is a judgement call, not a safe default.** On a bright, textured Veo clip it is genuinely invisible; on a dark Omni Flash night scene the box showed up as a bright rectangle on some frames. If you use it, look at the output before delivering.
- **Crop costs the same in both orientations** — 16:9 takes a right-cut, 9:16 takes a bottom-cut, both landing on 13.5%. The mark sits an equal distance from both edges, so there is no cheaper side and no per-model difference.
- `--cut right|bottom|center|topleft` overrides which side the crop sacrifices when `auto` cuts something important.

### `--mode inpaint` (LaMa)

The default for stills. Erases the mark with the LaMa model instead of blurring or cropping: no patch, no zoom, and every pixel outside the mark stays bit-identical. Verified against a roof-tile shot and a pottery scene where `delogo` visibly smeared a tile edge and a pot rim.

- **Needs `torch`, `numpy`, `pillow`** plus a one-time ~200 MB model download, cached at `~/.cache/lama/big-lama.pt`. If they're missing, `clean` says so and **falls back to `delogo` automatically** — so a still can silently come out worse than intended. If the fallback line appears, tell the user, and quote the ~1.5 GB install (`pip install torch pillow numpy`) before setting it up. An explicit `--mode inpaint` fails loudly instead of falling back.
- Do **not** install `simple-lama-inpainting` — it pins `numpy<2`, which has no wheel for current Pythons and falls back to a source build needing a C compiler. The bundled script loads the same model directly and skips that.
- **Video is supported but never the default**: ~11s per frame at 720p on CPU, so a 4s clip is ~18 minutes. Audio and frame timing are preserved exactly. Only offer it when crop's 13.5% zoom is unacceptable AND delogo's patch shows — and say the wait out loud first.
- Don't inpaint a mark sitting on text — it mangles letterforms. Regenerate with that corner clear instead.
- `--cut auto` picks the cheaper window on its own (`right` for 16:9, `bottom` for 9:16 and square). Only override with `--cut right|bottom|center|topleft` if the auto choice cuts something important.
- `--box WxH+X+Y` overrides everything. Use it if Flow ever ships a higher-resolution export — the preset is fixed-pixel and only verified on 720p/1024p output.
- **Clean reference images before reusing them** as `--image`/`--asset` in a video job. A marked reference risks the mark being painted *into* the new scene, where it is no longer an overlay and can't be removed. Cheap insurance, do it automatically.
- Only use this on the user's own generations. It changes nothing about disclosing that media is AI-generated (Flow output also carries invisible SynthID, which this does not touch and should not claim to).

## Bulk-importing files

Flow's file input takes **any number of files at once** — measured at 25 files
in a single trip. So importing a folder costs ONE upload, not one per file:

```bash
flow-auto generate create flow-upload-assets --image ./plates/*.jpg --wait
```

- Free: no generation, no credits, no daily quota.
- **Flow imposes no file-count limit** — its input is `multiple` and took 25 in
  one click. The 500-file cap here is about THIS side: the files travel to the
  browser as base64 and are held in memory together, and a 2 MB jpg is ~2.7 MB
  encoded. The upload itself is chunked 50 at a time, so 300 files is 6 trips
  rather than 300. If you genuinely have more than 500, run it twice.
- Reports `done` only once Flow actually LISTS them, not when the input is
  filled — so the moment it returns, `--asset "<filename>"` works.
- Uploads keep their **filename** as the asset label (`shot.jpg` → a row reading
  `shot.jpg`), and a repeat of the same name becomes `shot (1).jpg`. That is how
  you reference them afterwards.
- Batches do this for you: a run of image jobs uploads every image it is going
  to need in one trip before submitting anything, so you do not have to
  pre-import by hand. This mode is for when you want the files in the project
  *first* — building a reference library, or reusing one set across many runs.

## Renaming what is already in the project

Flow titles everything itself. A clip you just generated lands as "Brass diving
bell overgrown barnacles"; an upload gets whatever caption Flow's own vision
model produced — never the filename you sent. That is fine until a long run
wants to reuse its own earlier output, because `--asset "<name>"` then needs a
label nobody chose and nobody wrote down.

`flow-rename-asset` makes the name yours:

```bash
flow-auto generate create flow-rename-asset --asset "Brass diving bell overgrown" --name "DIVING_BELL" --wait
```

- Generates nothing, downloads nothing, costs no credits and no daily quota.
- `--asset` is matched as a **substring** of the current label, exactly like a
  reference — enough of it to be unambiguous is enough.
- Rename right after generating, while you still know which asset is which,
  then use your own name for the rest of the run.
- This is separate from the download filename, which `--index`/`--folder`
  already control. This one is the label **inside Flow**.

**Verified end to end on Flow's own generated media.** A picture generated in
the project, renamed to `SHOT_01`, then found again by that new name and
renamed to `SHOT_01_final` — both back to back, both taking effect on the page,
not just reported. So renaming a generation and immediately referencing it by
your own name is a supported thing to do, and you can chain it across a run.

If it ever does fail it names the step it failed at (`no Rename item in its
menu`, `never showed a text field`) and dumps that panel's real markup to the
extension's Debug Logs. Hand that dump over rather than retrying — Flow moved
its interface, and the dump is what fixes the selector.

## Merge separate videos

Concatenate independent clips (a montage — different scenes, not a continuous shot) into one file with ffmpeg. Same command as the grok side, it doesn't care which site made the clips:

```bash
flow-auto merge "clip1.mp4" "clip2.mp4" "clip3.mp4" -o "montage.mp4"
```

- Re-encodes by default so mixed resolutions merge cleanly with exact duration + A/V sync. `--fast` stream-copies (quicker, may drift the total duration).
- Needs ffmpeg on PATH — `winget install Gyan.FFmpeg` (Windows), `brew install ffmpeg` (macOS), `sudo apt install ffmpeg` (Linux) — or set `FFMPEG_PATH`.
- Use **extend** (above) for one continuous shot, **merge** for stitching unrelated clips. Merge has no length limit, so it's how you get past a single clip's duration ceiling.
- Merge only concatenates. Titles, captions, overlays and timed edits need a
  real editing tool, and **HyperFrames** is the one that pairs well with this:
  HTML compositions rendered to video, built for agents to drive.

  It is **not part of Flow Connector and not ours** — it is HeyGen's, Apache-2.0,
  open source at `github.com/heygen-com/hyperframes`, installed with
  `npx hyperframes@latest`. So do NOT assume it is present and do NOT install it
  silently. Ask the user whether they want it set up, and only then do it; if
  they say no, hand over the clean clips and let them edit however they already
  do.

  One thing to check before offering: HyperFrames needs **Node.js 22 or newer**,
  while this tool needs 20. A user happily running Node 20 will fail to install
  it, and that failure has nothing to do with Flow Connector — say so rather
  than letting it look like this tool broke.

## Errors

- `extension (flow) NOT CONNECTED` → ask the user to open Edge (or Chrome) with the Flow Connector extension and a visible flow.google.com tab.
- `Flow is open but the page never became ready` → **the extension already
  reloaded that tab twice before saying this**, so asking the user to press F5
  repeats what has already failed. The usual cause is a tab Chrome discarded
  under memory pressure, and two reloads did not clear it. What is left: the
  window is signed out, showing something other than Flow, or Flow itself is
  down. Ask about those.
  The extension deliberately does NOT open a second tab: a second tab hits
  whatever stopped the first, and the browser fills up with them.
- `still queued`/`still running` after a `--wait` → **the wait timed out, not
  the job.** Nothing failed, nothing was lost, and resubmitting that prompt
  spends a second generation for no reason. Poll `flow-auto generate get <id>`,
  or stop using `--wait` for multi-prompt runs (see **Running a batch**).
- **One Flow tab is all it uses.** Extra tabs do not make anything faster —
  batching happens inside the one tab — so if you see several Flow tabs, that
  is something to clean up, not a speed setting.
- `flow tab: HIDDEN` → stop before submitting; ask the user to bring the Flow window to the front (side-by-side layout above). This is UI automation, not an API — a hidden tab cannot generate.
- `not logged in to Google` → ask the user to log in, then retry once.
- `We noticed some unusual activity...` → Flow refused that ONE generation and did not charge for it. The queue keeps running and the next group waits longer automatically. Report the skipped prompt; **never resubmit it** and never suggest switching browsers to escape it. If several arrive in a row, raise `--delay` or say the run is being throttled — do not push harder.
- `policy-block: ...` (printed under an `error:` line) → the site refused the prompt on content policy. Flow's own wording is *"This generation might violate our policies. Please try a different prompt or send feedback"*. **Never resubmit the same prompt** — it fails identically, costs another minute, and the retry is already disabled on purpose. Rewrite it instead: drop named real people, branded/trademarked characters, and graphic violence or injury, and describe the shot generically ("a masked hero in a red cape" rather than the character's name). Then say out loud which prompt was refused and what you changed. Nothing was generated and no credits were spent.
### The loop for a long bulk run

Bulk runs are long and unattended, and some prompts WILL be refused
(minors, violence, real people). The tool flags each one and hands you the
site's own reason; deciding what to do with it is your job, not the
extension's. Per refused prompt:

1. Read the reason on the `error:` line — `policy-block:` marks a refusal.
2. Rewrite that one prompt and resubmit it **with the same `--index`**, so it
   still lands on its own number.
3. If the rewrite is refused too, stop retrying it. Move to the next prompt and
   leave the number as a hole — do not renumber, and do not let prompt 4 take
   the missing 3.
4. At the end run `flow-auto generate list --failed` and give the user that
   list: number, prompt, and why each one failed.

`--failed` reports **this run only**, and drops a failure that a later success
already replaced — so a prompt you rewrote and regenerated under the same
`--index` no longer shows up as a hole with a file sitting in the folder. If it
lists something, that number really is missing. (`--all` widens it back to the
whole history; you will rarely want that.)

### Name the run, and recovery stops being manual

Passing `--folder <name>` on a batch also RECORDS it: every prompt, its index,
and everything needed to submit it again. That record is what these read:

```bash
flow-auto generate runs                              # runs this machine remembers
flow-auto generate list --failed --run mybatch       # exact, not inferred
flow-auto generate resume mybatch --dry-run          # what would be re-submitted
flow-auto generate resume mybatch                    # re-submit only those
```

`resume` re-sends only what did not finish, on its original `--index`, so a
60-prompt run that lost 12 is one command rather than twelve. It **skips
anything already done**, which makes re-running a finished batch free rather
than a second bill, and it **skips anything still running**, so it cannot
generate the same prompt twice. Prefer it over resubmitting by hand.

Without `--folder` there is no record and `--failed` falls back to inferring the
run from timing, which is right most of the time and cannot be right always. On
a batch worth recovering, name it.

**`resume` does not rewrite prompts.** A `policy-block` will be refused again
in exactly the same way, so those still need the loop above — rewrite first,
then resubmit under the same `--index`.

One rewrite per prompt. A second refusal on a rewritten prompt means the
subject itself is the problem, and a third attempt just spends the user's
minutes.

- **Never treat a failed prompt as if it produced a file.** In a multi-shot run, a policy block on shot 3 leaves a hole: the assembly step must not run as though the clip exists. Report the missing shot, get the rewritten prompt generated, and only then continue.
- `generation timed out` / `SELECTOR_TIMEOUT:*` → Flow's UI may not match this build's (unverified) selectors, or is slow. Suggest the user run "Run Healthcheck" from the extension's Debug Logs tab and share the result so selectors can be fixed.
- `flow-image-to-image allows at most 10 reference(s)` → `--image` + `--asset` together exceed the cap; trim the list.
- `no clip open in Flow's editor` → `flow-extend-video` needs the clip open at `/project/<id>/edit/<clip>`. Ask the user to click the video, then retry.
- `could not enter Flow's Extend mode` → either the clip wasn't made with Veo (Extend doesn't exist for Omni Flash clips — the user has to regenerate it with `--model veo-3.1-fast` first), or the timeline/clip isn't selected. Not retryable. The extension dumps the live page to its Debug Logs on this failure; ask for that dump if the clip really is a Veo one.
- `no Flow asset matches "<name>"` → that name isn't in the composer's + panel right now. Not retryable — check the exact name, or pass the file with `--image` instead. Note an asset already attached to the current prompt drops out of that panel's list.
- `bridge not running and no spawnArgs given` → run any `flow-auto generate ...` command, which auto-starts the daemon.
- **Anything unexplained, at any point** → `flow-auto doctor --site flow` before guessing. It reports the things no error message mentions: a stale bridge daemon still serving old code, a Node too old, a hidden tab, a narrow window, a spent quota. Hand the user its fix lines verbatim.
- Extension connected but behaving as though it were an older build, or wedged
  after a long idle → `flow-auto reload-extension --site flow`. It restarts the
  extension in place and reconnects in a few seconds. **Anything generating dies
  with it**, so stop the queue first, and never reach for it mid-batch as a
  guess. Reloading the TAB is a different thing and does not restart it.
- Prompt gets typed, nothing generates, job dies ~300s later on `generation timed out` → the composer's Create button goes grey after the automation fills it and swallows the click; moving a real mouse over the tab makes it active again with the same text in place. `aria-disabled` stays `"false"` in both states, so nothing in the DOM distinguishes them. The extension sends real pointer motion over the composer before clicking, which clears it. If it recurs anyway, do not guess — capture a live element dump of that button while it is grey and compare it against the active state, and send that in.
- `clean` ran but the sparkle is still there → `--mark flow` was missing (the default targets Grok's corner, which on a Flow file is empty). Re-run with it.
- `<W>x<H> is too small for the flow mark preset; pass --box` → the frame is smaller than the mark's inset, so the preset can't be placed. Measure the sparkle and pass `--box WxH+X+Y`.
- `inpaint failed: ... No module named 'torch'` → LaMa deps aren't installed. Quote the ~1.5 GB one-time cost and ask before running `pip install torch pillow numpy`; offer `--mode crop` as the instant alternative.
- `--mode inpaint` seems to hang → it isn't; the model runs per frame and prints `N/M frames` as it goes. On a still it is ~25s. Only start it on video after telling the user the estimate (~11s per frame at 720p).
- `free plan daily limit reached — N/N images used today (this job needs M more). Resets at midnight, or go unlimited: <link>` → **STOP the run.** Not retryable, not a bug, and not something to work around by splitting the batch or waiting a few seconds. Tell the user plainly how much is left (the message names it), when it resets, and the upgrade link it printed — then ask what they want to do. Do not keep submitting; every later prompt fails the same way.
- **Anything you cannot explain, or a run that behaves oddly** → `flow-auto doctor --site flow` first. If it reports everything OK and the problem is still there, that is worth reporting: the extension's side panel has **Help → Report a problem**, which fills in the version, browser and recent log by itself. Point the user at that rather than asking them to describe the failure from memory.
