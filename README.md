# TRAX

Browses the card's music -- `ms0:/PSP/MUSIC`, or `ms0:/MUSIC` as ARK-4 makes it -- and
plays what it finds. Folders open, cue sheets read as
one album, and playing something opens a Player view built from the file's own
tags: album across the top, cover art on the left, title and artist beside it,
and a progress bar along the bottom. Select swaps that for a cassette that turns
as the track plays.

The audio, font and graphics modules are shared with `hello-psp`; the code here
is the browser (`src/browser.c`), the cue parser (`src/cue.c`), the two player
views (`src/main.c`, `src/cassette.c`, `src/skin.c`) and the UI itself.

## Controls

In the list:

| Input | Action |
|---|---|
| Up / Down | Move the selection (held, it auto-repeats; the stick works too) |
| L / R | How the next group will play: once, repeat, shuffle, or both |
| Cross | Open a folder or cue album, or play a track |
| Cross, held | On shuffle, start the group on the highlighted track |
| Circle | Up a level, or out of a cue album |
| Square | Back to the Player, when something is playing |
| Start | Quit |

In the Player:

| Input | Action |
|---|---|
| Cross | Pause / resume |
| Left / Right | Scrub (the stick works too) |
| L / R | Previous / next in the queue |
| Triangle | Track information, as a sidebar; in the Lyrics view, fetches the words |
| Triangle, held | Equalizer -- only where `trax_kernel.prx` did not load, since there is no ♪ button to read |
| Select | Swap to the chosen visualization, and back |
| Select, held | The list of visualizations, with the equalizer at the end of it where the helper did not load |
| ♪ (music note) | Equalizer, in one press; from the list, goes to what is playing |
| Circle | Back to the list |
| Square | Stop |

In the cassette, Up and Down change the skin and Select returns to the Player;
everything else behaves as it does in the Player, over any visualization.

With the equalizer open, Left and Right pick a band, Up and Down move it,
Triangle steps through the presets and Circle puts it away. Held directions
repeat, and the stick does what the d-pad does.

## Settings and About

Start opens a page of two tabs, and closes it; L and R move between them.
HOME is the way out of TRAX, as it is for everything on the PSP, and
Settings has a Quit row too.

**Settings**, Up and Down to pick, Left and Right (or Cross) to change:

- **Sleep timer** -- 15, 30, 60 or 90 minutes; the music fades over the last
  half minute and pauses. For the evening, so not kept.
- **Screen off while playing** -- after 30 seconds to 5 minutes with nothing
  pressed, the backlight and display go off while music plays; any button
  wakes it and does nothing else, and it lights again by itself if the music
  stops. Most of the battery goes on the screen. Needs `trax_kernel.prx`.
- **Resume on launch** -- opens the last track again, paused where it was
  left: what was started, the track, its play mode and position, written to
  `resume.cfg` when a track starts, each minute while playing and on the way
  out.

These are kept in `settings.cfg`.

**About** shows the version -- set in one place, `project()` in
CMakeLists.txt, which the XMB's version is made from too, with a pre-release
name such as "beta 1" in `TRAX_PRERELEASE` beside it -- the build date,
the battery, the memory stick's free space, whether the helper is running,
which fonts are loaded, and the libraries TRAX is built from with their
licences.

## How a group plays

Five ways, shown as icons in the action bar with the chosen one in white and
picked with the shoulder buttons: play once, repeat, repeat one, shuffle, and
shuffle with repeat. Repeat one plays the same track again when it ends -- a
track of a cue album goes back to its own start within the album's file -- and
the shoulders in the Player still move along the group, round from either end. They are set in the list, before anything starts, because that is
where the choice belongs -- a folder can be put on shuffle and then started,
rather than started and then rearranged.

A queue takes a copy of the setting when it is built, so the group already
playing keeps the way it was started while the list offers something else for
the next one. The choice itself is kept in `play.cfg` beside the EBOOT.

Shuffling rearranges the play order, not the list: the folder still reads in
its own order while the tracks come out in another. Every track plays once.
With repeat on, each time round is shuffled again -- and never starts with the
track that has just finished.

Which track opens a shuffled group is the caller's choice: **tap** Cross and
the shuffle picks, **hold** it and the highlighted track goes first. Both are
wanted often enough that neither could be the only one. Outside the shuffle
modes there is nothing to choose between, so Cross acts on the press as it
always has, rather than waiting to see whether it becomes a hold.

Changing the mode names it across the bottom of the list for a moment, and on
the shuffle modes says what tapping and holding do -- an unannounced gesture
may as well not exist.

## Supported files

`.mp3`, `.wav`, `.flac`, `.m4a` and `.opus`, matched case-insensitively:

- **WAV** — PCM 8/16/24/32-bit, IEEE float 32/64, A-law, mu-law, and
  `WAVE_FORMAT_EXTENSIBLE`.
- **FLAC** — native, 8 to 32 bits per sample, via `libFLAC`.
- **MP3** — via `libmpg123`, in software.
- **Opus** (`.opus`) — via `libopusfile`, which does the Ogg framing, the decoding and
  the seeking. It decodes at 48 kHz, which the mixer's resampler takes to 44.1; its tags
  are read from the first pages by hand, since opening a stream through opusfile measures
  the track and that means seeking to the end of the file.
- **ALAC** (`.m4a`) — Apple's reference decoder, vendored under `src/alac/`, fed by a
  minimal MP4 demuxer in `src/mp4.c`. Tags come from the iTunes-style `ilst` box.

Everything else in the folder is *listed but dimmed* and marked `unsupported`,
rather than hidden. Hiding it makes a missing file look like a bug in the
scanner; showing it answers the question directly.

An `.m4a` holding AAC rather than ALAC still will not play: the demuxer looks for an
`alac` entry in the sample description and gives up if there is none. ATRAC3 (`.at3`,
`.oma`) would need `sceAtrac3plus`.
MP3 could also move to `sceMp3`, which decodes on the Media Engine instead of
the CPU — worth it only if CPU time gets tight.

Note that FAT uppercases names that fit in 8.3, so `TRACK.MP3` and `track.flac`
can appear side by side. Matching is case-insensitive, so it makes no
difference to what plays.

### ALAC and MP4

- `src/alac/` is Apple's reference ALAC codec, unmodified, under the Apple Public Source
  License 2.0 (`src/alac/APPLE_LICENSE.txt`). Only the decode path is vendored; the encoder
  is not. It is built with warnings off, and without exceptions or RTTI.
- `src/mp4.c` reads only what playback needs: the `alac` sample entry (which carries the
  magic cookie the decoder initialises from), and `stsz`/`stsc`/`stco` to resolve each
  packet's position. `moov` is read into memory, `mdat` never is.
- `meta` is a full box -- four bytes of version and flags sit before its children, unlike
  every other container box here. Miss that and `ilst` is never found.
- Verified bit-exact against ffmpeg: 21,010,416 samples of a 238-second track, zero
  differences. ALAC is lossless, so anything less than an exact match is a bug.
- Adds roughly 100 KB to the EBOOT and pulls in `libstdc++`, since the decoder is C++.

### Capturing what the PSP drew

`gfx_screenshot()` writes the framebuffer as raw 480x272 RGBA. The autoplay build drops a
`screen.raw` next to the EBOOT, which converts with:

```python
Image.frombytes("RGBA", (480, 272), open("screen.raw","rb").read())
```

That is pixel-exact and independent of any emulator window, which makes it far better than
screenshotting the host desktop.

### Accented characters: do not use INTRAFONT_CACHE_ASCII

That option pre-caches the ASCII range and then, in the header's own words, "free some now
unneeded memory" -- which takes the glyph data for everything outside ASCII with it. Ask for
`U+00ED` afterwards and you get the wrong glyph: `Los del Río` drew as `Los del RÍo`. The
default cache fills on demand and costs little, so the font is loaded without it.

Worth recording how this was pinned down, because two plausible explanations were both
wrong. Printing the same codepoint through `intraFontPrintUCS2()` skips string decoding
entirely and still drew the wrong glyph, which ruled out the encoding. Loading every PGF in
turn and asking each for the same codepoints then showed *all* of them -- ltn8 included --
rendering it correctly, which ruled out the font. The only difference left was the load
flag.

`pgf_style()` also reasserts the string encoding on every style change, since
`intraFontSetStyle`'s options word carries the encoding alongside the alignment and passing
`0` there resets the font to per-byte ASCII. That was a real bug, just not this one.

`PSP_FONTTEST=ON` builds the diagnostic screen that settled it.

## Cue sheets

A `.cue` is shown as **one** entry carrying the album name from its `TITLE`, with the audio
file it indexes and the cover beside it hidden from the list. Opening it lists its tracks,
numbered, like a folder, with the album's name after the folder in the header; Circle goes
back out to the album. Playing one of them queues every track as a slice of that one file,
with start and end offsets, starting from the one picked -- so the shoulder buttons move
through the album, and the times and the progress bar show the position within the track
rather than within the album. Resuming on launch reopens the album's track list.

Two things real sheets get wrong that the scan works around:

- **The `FILE` line and the directory can disagree about encoding.** A sheet written in
  Latin-1 refers to `R\xEDo` while the directory reports UTF-8 `R\xC3\xADo`. Matching on the
  ASCII characters alone still identifies the file; failing that, an audio file sharing the
  sheet's own basename is used.
- **Sheet text is as often Latin-1 as UTF-8**, and nothing in the format says which. Text
  that is not valid UTF-8 is widened from Latin-1, so intraFont can draw it.

Cover art is a `.jpg`, `.jpeg` or `.png` named after the sheet or its audio file, or the only
image in the folder. It is drawn as a row thumbnail in the list, decoded one per frame so a
folder of albums does not stall the scan.

### Art for tracks that carry none

A track with no picture of its own borrows one from its folder, which is where a scanned
sleeve usually sits: `folder` first, as Windows and most players write it, then `album`,
then the folder's own name, and failing those the first picture there in name order --
`.jpg`, `.jpeg` and `.png` all count. Only that last resort is second-guessed: a picture
named after a cue sheet beside it belongs to that album, so a folder holding one cue album
and a few loose tracks does not hand the album's sleeve to all of them.

Every track in a folder shares the picture, and skipping between them keeps the one already
decoded. It is decoded without writing a cache file beside it; that belongs to cue albums,
whose covers are asked for far more often.

## Cassette view and skins

Select, in the player, swaps to a cassette that turns as the track plays. It fills the
screen and is drawn in layers, so the artwork can be pictures that know nothing about
animation. Bottom to top:

1. **Black**, which is what shows through the middle of a hub.
2. **Tape packs** (`src/cassette.c`): a ring of wound tape round each hub. The supply reel
   on the left empties as the take-up on the right fills.
3. **Hubs**: `HubSmall.png`, turned about its centre at each reel.
4. **Template** (`src/skin.c`): a PNG from the `SKINS` folder beside the EBOOT, drawn over
   everything with its alpha intact. Wherever it is transparent, the layers beneath show.

Up and Down cycle through the templates and the choice is kept in `skin.cfg`, so a copy of
the app carries its own wherever it goes. Installing TRAX puts the `skins/`
folder there.
`HubSmall.png` lives in the same folder but is never offered as a template.

### A skin is a folder

    SKINS/
      hub.png                     the hub every skin uses unless it brings its own
      Maxell XLII 90/
        Maxell XLII 90.png        the picture: named after the folder, or skin.png
        skin.json                 optional; without it the skin carries no text
        hub.png                   optional; this skin's own hub

A loose `.png` straight in `SKINS` still works, as a skin with no text.

### What skin.json says

```json
{
  "name": "Maxell XLII 90",
  "background": {
    "colors": ["#1B0B3A", "#0B3A5C", "#0B5C3A"],
    "seconds": 6,
    "blend": "fade",
    "while": "playing"
  },
  "defaults": { "size": 12, "align": "left", "color": "#1E1E1E" },
  "areas": [
    {
      "rect": [62, 70, 263, 22],
      "size": 11,
      "align": "center",
      "lines": [ ["album", ": ", "artist"] ]
    }
  ]
}
```

| Key | Meaning |
|---|---|
| `rect` | `[x, y, width, height]`, in the template's own 480 x 272 |
| `lines` | one entry per line of the label, top to bottom |
| a line | pieces joined in order: `title`, `artist`, `album`, `track` ("3/12") and `playlist` are filled in, anything else is printed as written |
| `size` | text height in pixels |
| `align` | `left`, `center` or `right`, within the area |
| `color` | `#RRGGBB`; leave it out and black or white is chosen from what the picture does behind that area |
| `background` | a `"#RRGGBB"`, or the object above: colours to move between, how many seconds each takes, `fade` or `step`, and whether it holds still while paused |

A field the track has nothing for drops out, and so does a separator that was
only there to go between two fields: `["album", ": ", "artist"]` with no artist
gives the album, not the album and a colon. A line too wide for its area is
shrunk a little and then cut with an ellipsis. Lines are stacked about the
middle of their area.

**A skin with no `skin.json` gets no text at all.** Nothing in the picture says
where a title belongs, and a guess would print it across the artwork.

### Making a template

Author it at **480 x 272**, the PSP's screen; that size is drawn pixel for pixel, and any
other is stretched to fill. Positions are screen pixels (`src/cassette.h`):

| | Pixels |
|---|---|
| Hub centres | x 154.5 and 326, y 121 |
| Hub image | 94 x 94, radius 47 |
| Tape pack, empty to full | radius 46 to 89 |

The template's alpha alone decides what shows through: wherever it is transparent, the
hubs and tape beneath are seen, and wherever it is opaque -- black included -- they are not.
Export with transparency; a PNG flattened onto a background layer covers everything.

The tape packs reach out to a radius of 89, so a window between the hubs shows them winding
from one reel to the other.

The earlier, smaller skins are drawn by `tools/make_skins.py`, which still describes them in
the 1000 x 640 reference space they were made in.

## Visualizations

Select swaps the Player for a visualization; held, it lists them: Up and Down
to move, Cross to choose, Circle to close. The choice is kept in `vis.cfg`.

| | |
|---|---|
| Tape deck | The cassette and its skins |
| Orb | The orb in pink, purple and blue, each latitude swelling with its own band |
| Level meter | Stereo LED meters, -48 to 0 dB, with falling peak markers |
| Smoke | Twisting ribbons of light, gusting now and then, in the middle two-thirds |
| Starfield | Flying through stars, the near ones trailing |
| Sphere matrix | 16 x 8 spheres, each column a band, lighting up from the bottom |
| VU meters | The pair from a tape deck, needles on a spring, with a peak lamp |
| Lyrics | The words, in time with the music |

The words come from an `.lrc`, kept in a `LYRICS` folder at the root of the
card with the music's own folders mirrored inside it -- a track at
`PSP/MUSIC/The Cars/Greatest Hits/album.flac` keeps its words at
`LYRICS/The Cars/Greatest Hits/album.lrc`, and one track of a cue album at
`album.03.lrc`, since the album's tracks share one audio file. One folder
keeps the words out of the music, where a browser would list them with nothing
to be done with them, and lets both builds read what either has fetched --
which matters because the signed build cannot reach a network at all. A file
beside the track is still read, and is no longer listed. Triangle fetches from
lrclib.net over Wi-Fi (`src/net.c`), making the folders under `LYRICS` as it
goes; `LYRICS` itself is made at start-up, since the signed build can never
fetch and would otherwise never make one for somebody with words of their own.

The mixer copies what is heard, after the equalizer, into `analysis.c`, which
once a frame on the interface's thread folds a 1024-point FFT into 24 bands,
40 Hz to 16 kHz, each measured against its own level of the last few seconds
so loud and quiet masters move alike. The visualizations draw through
`vgfx.h`: `vgfx_psp.c` batches for the GE, and `tools/vispreview` draws the
same in software, rendering any of them to a video with a track's sound:

```bash
tools/vispreview/build.sh 3 song.flac 20 30   # Smoke, 20 s from 0:30
```

## Equalizer

Five bands -- 60 Hz, 250 Hz, 1 kHz, 4 kHz and 12 kHz -- each a peaking filter
of 1.4 octaves, adjustable by 12 dB either way. Measured on the host, a band
at +12 dB reads +12.0 dB at its centre, about +4 an octave away and nothing
at all two octaves out; cuts mirror it. Five presets ship: Flat, Bass,
Treble, Vocal and Loud. The bands are kept in `eq.cfg` beside the EBOOT.

It runs in the mixer, on the finished 44.1 kHz stereo stream, rather than
where a track is decoded. That way a band moved is heard in the next buffer
instead of once the ring buffer has drained a second and a half later, and
its cost does not change with the source format: five biquads a channel,
skipping any band left flat. Output is clamped rather than allowed to wrap,
since lifting a band on loud material can exceed what a 16-bit sample holds.

Select is the one button the Player had left: tapped it swaps to the
cassette, held for half a second it opens the equalizer. The equalizer opens
the moment the hold is long enough, so nothing has to be released first to
find out which it was.

The ♪ button does the same in one press, and from the list it goes to what is
playing with the equalizer open. It is one of the buttons a user-mode program
never sees, so it works only when `trax_kernel.prx` is loaded -- see below.

### The kernel helper

`trax_kernel.prx` sits beside the EBOOT. It is a small kernel-mode module
that returns the buttons a user-mode program cannot see (Volume, ♪, Screen)
and the system volume and mute, each call taking kernel rights only for its
own length. TRAX loads it at startup through kubridge, which ARK, PRO and
Adrenaline provide, and calls it through stubs generated from the same
`kernel/trax_kernel.exp`. If it does not load, TRAX runs as before without
the two things it brings:

- **The volume, shown.** The PSP changes its volume under TRAX without
  drawing its own bar -- TRAX redraws the whole screen every frame, over
  anything the system paints -- so a panel in the middle of the screen shows
  the level, 0 to 30, or "Muted", whenever it moves or a key is pressed at
  either end, fading out after a second and a half.
- **♪ for the equalizer**, as above.

### The tape maths

Tape is wound in a flat spiral, so what a reel holds is proportional to the **area** of the
ring, not its radius -- `r = sqrt(hub^2 + filled * (full^2 - hub^2))`. Interpolating the
radius directly empties the supply reel far too fast at the start, which reads as wrong even
to someone who could not say why. For the same reason the reels turn at different speeds:
angular speed scales with `1 / radius`, so the emptier reel spins faster.

Position drives everything, so scrubbing moves the reels with it.

## Player view layout

Three sections, all derived from constants at the top of `main.c` rather than hardcoded
positions:

1. **Album name** centred across the top, its section height set by the font plus 2 px.
2. **Album art**, square, filling the height from there to the bottom of the screen with
   10 px of padding, left justified -- 232 x 232 on a 480x272 panel. With no art, an empty
   area with a white outline.
3. **Track info** to the right of the art with 22 px of clearance, the title over the artist,
   the pair centred on the art's vertical midpoint. The title is measured before it is
   drawn, so a long one wraps to two or three lines and the artist moves down to clear it
   rather than being written over.
4. **Progress**, bottom justified so its last row lines up with the bottom of the art: a
   4 px bar with one-pixel rounded corners, then the elapsed and total time left justified
   with the bar, and a play, pause or stop icon right justified to the bar's end. Between
   them, dimmed, is how this group is playing -- repeat, shuffle, or both -- and nothing at
   all when it is playing straight through, which is what no icon means. It follows the
   queue, not the list, so choosing a mode for the next group does not misreport this one.

Album, title, artist and the times all use the PSP system font; only the track list and its
footer stay monospace.

### Track information

Triangle slides a panel in from the right with the title, artist and album, then what the
file actually is: container, codec, bit depth, sample rate and channel count. The audio
layer reports these through `sound_stream_info()`, which reads them where each format keeps
them -- the `fmt ` chunk for WAV, STREAMINFO for FLAC, `mpg123_getformat()` for MP3, and the
ALAC magic cookie for MP4.

Lossy codecs have no source bit depth, so MP3 shows "lossy" beside its rate rather than
inventing the decoder's 16 bits.

### Seeking

`music_seek_ms()` is requested by the UI thread but performed by the decoder thread, which
owns the stream. While a seek is pending the mixer is fed silence, which is what lets the
producer reset both ends of the ring buffer without racing the audio thread. Seeks are
frame-accurate for WAV, FLAC and MP3; ALAC lands on the enclosing packet, at most ~93 ms
away, because its packets decode independently.

Scrubbing with Left and Right, or the stick, runs at 9 seconds of track a
second for the first two seconds it is held, twice that until five seconds,
and three times it after that: slow enough at first to place a point
precisely, quick enough later to cross a long track. Letting go, or turning
round, starts again from the slowest.

## Text rendering

Two systems, deliberately:

- **The bitmap atlas** (`src/font.c`) draws the track list, timings and control hints, where
  fixed-width columns line up and the terminal look suits the app. ASCII only.
- **intraFont** draws the title, artist and album, using the PSP's own PGF system fonts. Those
  are variable-width, cover Latin, Cyrillic and Greek, and fall back through `jpn0.pgf` and
  `kr0.pgf` for Japanese and Korean, so a title in any of those renders correctly. Strings are
  passed as UTF-8, which is what every tag format gives us.

### Two traps when mixing intraFont with your own GU drawing

- **Restore `sceGuTexScale`.** `gfx_draw_texture()` sets it to 1/width for the artwork;
  leaving it there makes every intraFont glyph sample a single texel, so text renders as
  solid blocks.
- **Enable blending first.** intraFont sets its own texture state but not the blend state,
  and glyph coverage lives in alpha. With blending off each glyph paints its cell background
  opaque -- white letters stamped on black boxes.

### Where the fonts come from

`flash0:/font/*.pgf` on real hardware. **PPSSPP does not expose flash0 to homebrew file
I/O** -- its bundled PGFs only serve the emulated `sceFont` -- so `load_pgf()` falls back to a
copy beside the EBOOT. For emulator testing, copy `ltn8.pgf`, `jpn0.pgf` and `kr0.pgf` from
PPSSPP's `assets/flash0/font/` into the game folder. They are Sony's fonts: fine locally,
not something to redistribute, and not needed on hardware.

Chinese is the remaining gap -- PSP firmware has no Chinese font, so that would need FreeType
(`libfreetype.a` is in the toolchain) and a font on the memory stick.

## Metadata

- **FLAC** — Vorbis comments and the PICTURE block, parsed directly rather than through
  libFLAC's metadata API. That API drags in file rewriting, and with it `chown()` and
  `utimensat()`, which newlib does not have on the PSP; the block format is simple enough
  not to need it.
- **MP3** — ID3v2 through mpg123 (`MPG123_PICTURE` must be set before `mpg123_open` for
  embedded art), falling back to ID3v1. `mpg123_scan()` first, since tags sit at either end.
- **WAV** — no tags read; the filename is used.
- Missing fields degrade individually: no title falls back to the filename, no art draws a
  plain block, no duration shows elapsed time with an unfilled bar.

### Cover art

- Covers are commonly 1500x1500, which is 6.7 MB decoded -- more than a PSP wants to hold.
  libjpeg decodes at 1/2, 1/4 or 1/8 scale, so the largest scale that still covers the 64x64
  texture is chosen and a 1500x1500 JPEG costs about 105 KB instead.
- The compressed bytes are freed as soon as the texture exists; only the 64x64 copy is kept.
- Art is drawn 168 px square -- 62% of the screen height -- inset 20 px from the left edge,
  one texel to one pixel. GU textures must be power-of-two, so the 168 px image sits padded
  in the top-left of a 256 px texture and only that region is sampled.
- Three things keep it sharp, and all three matter:
  1. **Decode near the target, not far below it.** libjpeg is asked for at least 2x the
     final size, so the box filter has real samples to average. Decoding at 1/8 and
     scaling 188 -> 148 smears; 1/4 and 375 -> 148 does not.
  2. **Use GU_NEAREST when drawing 1:1.** Bilinear at a 1:1 mapping blends every pixel with
     its neighbour and softens the image for no benefit.
  3. **A light unsharp pass** after downscaling -- `SHARPEN_NUM/SHARPEN_DEN` in
     `artwork.c`, currently 1/16. Any area-average resize costs edge definition, but push
     this too far and the result stops looking sharp and starts looking pixellated: the
     stair-steps from the downscale get emphasised along with the detail.
- Dithering would not help here: it addresses banding from reduced colour depth, and these
  textures are full 32-bit RGBA. It would only become relevant if the artwork moved to a
  16-bit format such as GU_PSM_5650 to save memory.
- GU texture coordinates are in **texels**, multiplied by `sceGuTexScale`. Spanning a whole
  texture means UVs of 0..width, not 0..1 -- with 0..1 the quad samples a single texel and
  renders as one flat colour.

## Real hardware differences

- **The main thread gets a 256 KB stack.** A `struct Queue` is about 190 KB and a
  `struct Cue` about 32 KB, so holding either as a local overflows it: the app dies to a
  black screen on a PSP while running fine under PPSSPP, which does not enforce the guard.
  Both live in static storage instead. Anything of that size belongs in BSS or the heap.
- **Never dereference `SceIoDirent.d_private` without checking it.** It is device specific,
  and a bad pointer is an exception and a black screen rather than a NULL check away. The
  scan tests that it addresses real RAM first.

Things that behave differently on a PSP than in PPSSPP, all found by testing on a device:

- **Four-character extensions get mangled.** FAT keeps an 8.3 short name beside the long
  one, and `.flac` does not fit, so the device can report `TRACK~1.FLA`. Matching on the
  extension alone misses those files; the scan falls back to reading the first four bytes
  (`fLaC`, `RIFF`, `ID3`, or an MP3 frame sync) so detection does not depend on the name.
- **Non-ASCII filenames do not always round-trip.** A name containing, say, `í` can come
  back from `sceIoDread` in a form the device will not take back, so the file lists but
  never opens. The fix is FAT's own: `SceIoDirent.d_private` carries the 8.3 **short name**
  alongside the long one, and a short name is plain ASCII and always reopens. The browser
  keeps both -- the long name is what you see, the short name is what it opens with when the
  long one fails (`browser_resolve()`). `d_private` is device specific and absent under
  PPSSPP, so there the long name is all there is.
- Names that are not valid UTF-8 are widened from Latin-1 before drawing, so accented
  filenames render instead of showing gaps. Only the displayed copy is changed; the bytes
  used for opening are untouched.
- **Hi-res FLAC is far more expensive.** A PSP runs at 222 MHz by default; 96 kHz is 2.2x
  the decode work of CD audio and 192 kHz is 4.4x. Watch `buffer %` and consider
  `scePowerSetClockFrequency(333, 333, 166)`.
- **ALAC (`.m4a`) is not PSP content.** No firmware plays it and XMB reports "Corrupted
  data". Transcoding to FLAC is lossless and comes out the same size.
- Build with `-DPSP_SCAN_LOG=ON` to have the scan write `scan.log` next to the EBOOT,
  recording each name the device reported along with its raw bytes.

## Notes

- Tracks are streamed, never loaded whole: a decoder thread keeps a 128 KB ring
  buffer topped up, so a long file costs the same as a short one. See the
  streaming notes in `../hello-psp/README.md`.
- The decoder produces a run of samples per publish rather than one at a time,
  and skips resampling entirely when the source is already 44100 Hz. Per-sample
  atomics cost more than the decoding does.
- Hi-res sources work: 96 kHz and 192 kHz 24-bit FLAC both stream with the ring
  buffer holding at 98%, resampled down to the 44100 Hz the hardware runs at.
- **The sample rate must come from STREAMINFO**, via the metadata callback.
  `FLAC__stream_decoder_get_sample_rate()` can still read 0 right after
  `process_until_end_of_metadata()`, and a fallback to 44100 there is not a safe
  default -- it plays a 96 kHz file at 0.46x speed and a 192 kHz file at 0.23x,
  which sounds like the track dragging rather than like an error.
- The status line shows the source sample rate and the ring buffer fill. Those
  two numbers separate the two ways playback goes wrong: a rate far from 44100
  means pitch/speed is off, while a buffer trending to 0% means the decoder is
  losing and you are hearing underruns.
- `sceIoDread` returns > 0 while entries remain, and `SceIoDirent` must be
  re-zeroed between calls or stale fields leak into the next entry.
- Directories are skipped via `FIO_S_ISDIR(ent.d_stat.st_mode)`.
- The list is capped at 256 entries; past that the header says `(truncated)`.
- Long names are shortened in the middle so the extension stays visible.
- PPSSPP is more forgiving than real hardware about filename case, so a file
  that lists here may not on a PSP if the case differs.
