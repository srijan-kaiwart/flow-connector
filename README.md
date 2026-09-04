# Flow Connector

Ask your coding agent for a picture, get a file.

Flow Connector is a Chrome extension that connects your coding agent to
[Google Flow](https://flow.google.com). You ask for images or video the same way
you ask for code, and the finished files land in your Downloads folder.

It also works on its own. Hand it a list of prompts, walk off, come back to a
folder of results.

Everything runs in **your** browser, on the Google account you are already
signed into. Nothing is proxied through a server of ours, which is why
generating costs nothing extra and why there is no API key to buy.

## Why it exists

Flow is built for making one thing at a time. Type a prompt, pick a model, wait,
download, repeat. That is fine for one shot. For twenty it is an hour of
clicking, and you have to be at the keyboard for all of it.

## What it does

- **Runs a queue.** Prompts go in, files come out, in order, with names you can
  read.
- **Downloads for you.** Every finished image and video is saved as soon as it
  is ready, sorted into folders.
- **All of Flow's modes.** Text to image, image to image, text to video, first
  and last frame, ingredients to video, video to video, characters, voices, and
  extending a clip.
- **Talks to your agent.** One paste and your agent can generate. There is a
  Claude Code skill, and an MCP server for everything else.

## The agent bridge

Open the extension, copy the setup prompt, paste it into your agent once. It
installs a small command-line helper and wires itself up.

After that:

```
> make 8 cyberpunk street shots, 16:9, then animate the best one
```

The two halves talk over loopback on your own machine. Nothing listens on a
public interface, and no prompt or file passes through anyone else's server.

You do not need an agent. The side panel does the whole job on its own if you
would rather type prompts into it.

## Install

From the Chrome Web Store — the listing is in review. This repo is the support
channel and the home of the agent skill and the docs; the extension itself ships
through the store.

## Two things worth knowing before you install

**The Flow tab has to stay visible.** Not minimised, not covered by another
window. Chrome throttles windows it thinks nobody is looking at, and a throttled
tab stalls mid-generation. The extension has a button that parks the browser
next to your editor. This applies on the paid plan too — closing the *extension
panel* is not the same as running in the background.

**Flow's own limits still apply.** This drives the real interface, so Google's
rate limits and content rules land exactly as they would if you were clicking.
The extension paces itself and backs off if Flow reports unusual activity. A
refusal is one generation, not a ban.

## Free and Pro

The free plan has a daily allowance of images and video, runs one prompt at a
time, and needs the panel open while it works. The current limits are shown in
the panel.

Pro is faster, has no daily cap, and lets you close the panel. It is a fixed
period paid once — nothing renews on its own and no card details are stored. The
price is shown in the extension before you pay.

Every model and every mode works on both plans. Nothing is held back.

## Something broken?

[Open an issue](https://github.com/srijan-kaiwart/flow-connector/issues/new).
Flow changes its interface fairly often and that is usually what breaks
automation, so the extension's Help tab fills most of the report in for you —
version, what the automation was doing, and where it stopped. Paste that in and
it is usually enough to find the problem.

## Not affiliated with Google

Flow Connector is an independent project. It is not affiliated with, endorsed
by, or sponsored by Google LLC, and Google product names appear here only to say
what this tool works with. Automating a website may be subject to that site's
terms of service, so please use it responsibly.
