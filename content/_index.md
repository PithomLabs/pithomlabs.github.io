## Remember When Music Was Yours?


Folders full of MP3s. Hand-picked albums. Meticulously named files.
No algorithm deciding what you should feel next — just your collection, your taste, your rules.


Go to [https://github.com/PithomLabs/amp](https://github.com/PithomLabs/amp)



If that hits a nerve, you’re going to love AMP.

[Download here](https://github.com/PithomLabs/amp/releases)

So What Is AMP?

AMP is a single-binary, zero-dependency MP3 player with a gorgeous web UI. Download it. Run it. Open your browser. That’s it — you’re listening to music.

```
./amp
```

No install wizard. No node_modules. No Electron. No sign-in.
Just one Go binary with everything baked in, and your MP3s doing what MP3s do best: playing.

Why You’ll Dig It

One Binary to Rule Them All

AMP compiles to a single executable with all HTML, CSS, and JS embedded right inside. Drop it on a flash drive, carry it between machines, run it on a Raspberry Pi in your closet — it just works.

Synced Lyrics That Actually Sync

Got timed LRC lyrics? AMP highlights each line in real time, karaoke-style, auto-scrolling so you never lose your place. It’s unreasonably satisfying to watch.

Smart Lyrics Fetching

Don’t have lyrics yet? AMP searches LRCLIB for an exact match, falls back to a fuzzy search, and even validates synced lyrics against your file’s duration (within 15 seconds) so you don’t get garbage timing.
Found the wrong match? Paste an LRCLIB URL directly to import the right one.

An Approval Workflow (Yes, for Lyrics)

Become a member
Fetched lyrics land as pending. You can approve, reject, edit inline, or delete them entirely.
It’s your library — you get the final say.

Auto-Tagging That Just Makes Sense

AMP reads your ID3 metadata and automatically tags songs by artist, genre, and decade. Filter your playlist by “90s + Rock” in two clicks. No manual tagging. No spreadsheets. It just figures it out.

A UI You’ll Actually Want to Look At

Split-panel layout with a draggable divider — lyrics on one side, playlist on the other. Dark mode and light mode. Glassmorphism touches. Responsive and snappy. It looks like it was made by someone who cares, because it was.

Keyboard Shortcuts for the Mouse-Averse

Key Action
Space Play / Pause
N Next track
P Previous track
Arrow keys Seek forward/back 5s

Hands on the keyboard, eyes on the lyrics. As it should be.

Offline-Friendly, SQLite-Backed

Everything persists in a local SQLite database. Your last folder, your lyrics, your approvals — all remembered between sessions. No cloud. No account. No surprises.

Get Started in 30 Seconds (if you like vibe coding)

```
go build -o amp .
./amp
```

Open http://localhost:8080, point it at your music folder, and rediscover your collection.

Want a different port? A different database location? Two flags, that’s all:

```
./amp -port 3000 -db ~/music.db
```

Try It. Star It. Enjoy Your Music Again.

AMP is for people who never stopped loving their MP3 collections.
If that’s you, give it a spin.

And if it makes you smile, leave a star — it means more than you think.

Happy listening.
