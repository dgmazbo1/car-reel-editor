---
name: "car-reel-editor"
description: "Edit a Loomelic dealership car reel (vertical 4K, ~18s, 3D motion graphics, stabilized 50% slow-mo, beat-cut to Epidemic Sound) from raw clips + a VDP link; HyperFrames by default."
---

# Car Reel Editor (Loomelic)

You turn a folder of raw car clips and a vehicle listing (VDP) link into a finished vertical reel in Denham's house style. This skill keeps growing: when Denham corrects or approves something, add it under **House look**, **Dealer styles** or **Rules learned**, and say that you did.

## 0. Which editor
- **Always edit in HyperFrames** unless Denham explicitly says to edit in **DaVinci Resolve** or **Adobe Premiere Pro**.
- "HyperFrames" means an HTML/GSAP-style composition: one HTML page with deterministic `window.seek(t, layer)`, rendered frame by frame through Chromium (Playwright) to transparent PNGs at 2160×3840 and composited over the footage. If the HyperFrames CLI or skills can be installed, use them. If not, author the same composition by hand and render it with Playwright. Don't stall.
- When he asks for the edit "in Resolve", "as a Resolve draft" or "load it into Resolve", build the timeline in Resolve (section 8). **Never render in Resolve unless he asks.** Save the project only.

## 1. Inputs and setup
- Footage usually comes from a DJI handheld: vertical 1728×3072, HEVC 10-bit, 59.94 fps, Rec.709. Clips live in a connected folder on his Mac, e.g. `/Volumes/4TB SSD Backup/Customers /<Dealer> /<year> /<month>/<car>/`. Some folder names have trailing spaces, so quote every path.
- Do all work in `<car folder>/_reel_work/` (sheets, proc, trf, project, resolve, parts).
- **Length:** default 18s; he may ask for a range (e.g. 17–20s). Pick the length that puts the end card on a bar line (section 5).
- **Confirm the dealer.** The folder name, the VDP domain and the plate frame in the shots must all match the dealer named on screen. If they disagree, flag it in one line, follow his words, and keep the dealer name in one variable.
- **Pick the dealer's graphics style (section 6) before building anything.** Each dealer has its own look; never reuse another dealer's style.
- Vehicle facts come from the VDP. If WebFetch returns a listing page instead of the VDP, scrape it with Firecrawl. Pull year, model, trim, engine, HP/torque, transmission, drivetrain, MPG, exterior and interior colors, installed options and key features. Fill missing HP/torque from KBB, Lexus or another dealer's listing of the same trim, and say which specs came from elsewhere. If the VDP's spec list and description disagree (e.g. 6-speed vs 10-speed), go with the model-correct one. If the listed paint doesn't match what the footage shows (e.g. a matte/smoked wrap), leave the color name off and flag it. **Never put the price or stock number on screen** unless he asks, and leave off any contested stat such as 0–60.
- Certification rule: if the car is L/Certified, mention it. If it isn't, never mention certification.
- **No color grade.** Footage ships as-shot (Rec.709 straight from the camera). Don't apply a LUT, `eq`, or any other grade anywhere in the pipeline, and don't ask him for a LUT file.

## 2. Look at everything first
- Make a contact sheet: 4 frames per clip, clip ID burned in. Read every sheet. Group the clips by angle family (front head-on, front 3/4, low front 3/4, wheel, side profile, rear 3/4, exhaust, trunk badge, fender badge, grille/emblem detail, interior: wheel/dash, console/screen, seats, second row/moonroof, door panel).
- **No repeated shots.** Each cut is a different angle. Two wheel close-ups, two rear-exhaust shots or two front-3/4 wides count as repeats. A foreground object passing the lens (a pillar, a post) is a great natural wipe, so use it.
- Pick a hero shot for the opener: a head-on (ideally a push-in) with the car in the lower-middle of the frame, a clean background above, and mostly stable footage.
- Use interior clips for the tech/comfort/audio cards when they exist.

## 3. Footage processing: every clip stabilized and slowed to exactly 50%
**Non-negotiable: every shot in the reel is stabilized and plays at exactly 50% speed.** That includes the hero, interiors, close-ups, mask-transition lead-ins and the end card. No shot plays at 100% and no shot skips stabilization unless Denham asks. Verify both before compositing (step 4 below), not after.

**Known trap: the Mac shell's ffmpeg cannot stabilize.** Its libvidstab (ffmpeg 4.4.2, Ubuntu arm64 VM) writes a valid .trf but cannot read it back: `vidstabtransform` logs `cannot read frame local motions from file, got … (255)` and then passes every frame through untouched, so the clip only *looks* processed. Never stabilize on the Mac. Stabilize in the container (x86_64 ffmpeg, where vid.stab works).

Process a **window** around each selected range (about 1.8s; 2.4–2.6s for the end card; ~3s for the hero), not the whole clip.

1. **On the Mac (fast, 4 cores): cut, clean, upscale, sharpen. No vid.stab here.** Write a small `proc.sh` and run 3 windows per call (each shell call is capped at 180s; a 1.8s window takes about 30s). Use `-nostdin` on every ffmpeg inside a `while read` loop, or ffmpeg eats the loop's input.
```
ffmpeg -nostdin -ss A -t D -i SRC -an -vf "hqdn3d=1.2:1.2:2:2,scale=2160:3840:flags=lanczos,cas=strength=0.45,unsharp=7:7:0.7:7:7:0,format=yuv420p10le" -c:v libx264 -preset veryfast -crf 12 -threads 4 -color_primaries bt709 -color_trc bt709 -colorspace bt709 proc/NAME.mp4
```
   Log `id file A=<start>` to offsets.txt, so time inside the file = source time − A. Start each shot about 0.25–0.4s into its window to skip the stabilizer's edge, and check that every shot's source range (including mask lead-ins and ramps) fits inside its window. Files can't be deleted on the Mac, so always write unique file names. If a call times out, check the index file, since the job may have finished.
2. **Stage to the container.** Staging is flaky: stage 1–2 files per call and retry the failures one at a time. If a file keeps failing, `split -b 4m` it on the Mac, stage the pieces, `cat` them back together in the container and check the md5 against the Mac's.
3. **Stabilize in the container**, two-pass vid.stab on each 4K window:
```
ffmpeg -i proc/NAME.mp4 -vf "vidstabdetect=shakiness=6:accuracy=15:stepsize=6:result=stab/NAME.trf" -f null -
ffmpeg -i proc/NAME.mp4 -vf "vidstabtransform=input=stab/NAME.trf:smoothing=30:zoom=4:optzoom=0:interpol=bicubic:crop=keep,format=yuv420p10le" -c:v libx264 -preset veryfast -crf 12 -color_primaries bt709 -color_trc bt709 -colorspace bt709 stab/NAME.mp4
```
   smoothing=30 averages the camera path over 61 frames (about 1 s), which removes handheld bob while keeping deliberate pushes and orbits. The compositor reads `stab/`, never the unstabilized `proc/` files.
4. **Verify, every clip, before compositing:**
   - **Stabilized:** the transform log must contain no `cannot read frame local motions` line, and the stabilized window must actually differ from the input. Downscale both to 432×768, run cv2.phaseCorrelate frame by frame and require a nonzero per-frame shift. Identical frames mean the stabilizer did nothing, so fix it before going on. Also compare raw vs stabilized wobble (LK feature path, deviation from a 0.5 s moving average); stabilized must not be worse than raw.
   - **50% speed:** print each shot's camera time ÷ screen time from the timeline module. Every shot must read 0.500, including mask lead-ins (`src_pos_shot` with a negative local time) and the end card.
   - Report both checks in one line each in the delivery reply.
- No grade, here or later (section 4).

## 4. House look (updated 2026-10-28: no grade)
- **No color grade.** Footage is delivered as-shot, Rec.709 from the camera. No LUT, no `eq`, no stand-in, no per-clip grade, in HyperFrames or in Resolve. Decode shots with `scale=in_color_matrix=bt709:out_color_matrix=bt709,format=rgb24` only, so color passes through untouched. (The old house LUT `Loomelic_CarReel_HouseGrade_v1.cube` is retired; don't use it or ask for it.)
- **Sharpness:** CAS 0.45 + unsharp 0.7 at 4K. Resolve projects use the "Sharper" resize filter with no extra sharpen nodes. Never add grain.
- **Format:** 2160×3840, 60 fps timeline, Rec.709, H.264 high CRF 15, AAC 320k.
- **Speed and steadiness:** every shot stabilized and at exactly 50% (section 3).
- **Motion must flow.** Never ship slow-mo where each source frame is shown twice (it reads as choppy, especially on long orbits like the end card). Every output frame must be unique (section 7).

## 5. Edit structure (beat-locked)
- **Music:** pick from Epidemic Sound (MCP; the same catalog he has open in the browser). Match the mood he asks for: dark, luxury no-vocal trap/hip hop by default; for "classy", sophisticated no-vocal deep house/lounge. Don't reuse the previous reel's track. Plot the loudness of a few previews to find a clean intro build into a hard drop. Download the full WAV, find the drop and a second drop with librosa (beat grid + onset strength + a zoomed RMS plot), then build the bed:
  - 3s of intro build → **drop at 3.0s** → main section → splice into the second drop for the end card, with a tail fade. Loudnorm to −14 LUFS.
  - **The end-card splice must land on a bar line**: a multiple of 4 beats after the drop (28 beats at ~148 bpm; 24 beats at ~113 bpm). Never splice mid-bar; it sounds like a skip and makes the end card feel disconnected. Landing on a phrase line (32 beats, right after the track's own fill) is even smoother.
  - Verify the onsets at fine resolution (hop 64) land within a frame of 3.0s and of the end-card cut; nudge the bed with a small adelay if they lead the picture.
  - Add an Epidemic "reversed whoosh" peaking right before the drop and before the end-card cut, at about −7 dB (−10 dB for classy tracks).
  - Used so far: "HUSTLE" by Ballpoint (LOH IS 350), "SHOWTIME" by Cushy (LOLV IS 350), "Veranda Cava" by Phello (LOLV LX 600), "Twisting" by Civet Cat (LRH Defender 110 CPO), "Moments (Instrumental)" by Ayoub (LRH 2025 Range Rover SE CPO).
- **Timeline at 60fps** with every shot at exactly 50% (flow-interpolated, section 7). After the drop, cut on a beat grid: 3-beat shots carry text, 2-beat shots are breathers or short single-word cards.
  1. **Hero 0–3.0s:** car cut-out (rembg `isnet-general-use` matte per frame at 1080×1920, thresholded to drop floor reflections, temporally smoothed, blended at fractional positions, upscaled). The year rises from **behind** the car; model and trim sit **in front** of it. Base zoom about 1.2 on the hero. Preview hero frames before the full render: the year's baseline must sit just behind the roofline (tall SUVs sit much higher in frame than sedans). Lexus reels: speed-ramp 0.5×→~3× into the drop, with motion blur (the base speed is still 50%).
  2. **Spec cards**, one per shot: HP/engine · torque/transmission · drivetrain · wheels · tech · audio · interior color/material · comfort · towing/MPG/safety. Leave 1–2 shots with no text as breathers when there's room.
  3. **End card (about 3–3.6s):** dealer name, dealer line, small year/model footer.
- **Transitions** (Lexus reels; LRH reels use plain hard cuts only, no push, flash or ramp):
  - Hard cuts on the beat, with a subtle 1.035 push on most cuts. Use a Dynamic-Zoom-style eased push (1.0→1.10) on one static wide (still at 50% speed).
  - Drop cut: punch-in (1.10→1.0 over 0.22s) and a white flash (≈30–40%, 0.12s).
  - **Into the end card:** ramp the last 0.4s of the previous shot from 0.5× to 2.5× with motion blur; ease the end card from 2× down to 0.5× over its first 0.4s. Use a soft flash (≈24% peak, centered on the cut) and a gentle 1.06→1.0 push over 0.45s. Fade the end-card darkening in over 0.45s instead of snapping it on, and clear the previous card's text about 0.36s before the cut.
  - These short ramps are the only departures from 50%, and only on dealer styles that call for them. LRH reels have none.

## 6. Graphics: dealer styles
Everything is 3D motion graphics (depth, perspective, motion), never flat static text. Each dealer has its own look.

### Lexus of Henderson: "chrome 3D"
- 3D text = a stack of 18–24 layers along Z (light-to-dark gray), a chrome face (`background-clip:text` gradient), a blurred shadow layer and a moving glare band. Animate with a per-character rotateX flip-in, eased back-out, a slow rotateY drift during the hold, and letter-spacing that tightens on labels.
- Fonts: Archivo Variable (wdth 112–125, wght 900) for big type; Inter 500–700 with wide tracking for labels (npm @fontsource-variable).
- Hero: year in 3D chrome with a glare sweep and lens-flare glint; end card with the dealer name in 3D chrome and the dealer line on a dark gradient.
- End line: "LARGEST LEXUS DEALER IN NEVADA".

### Lexus of Las Vegas: "clean Jost" (from the Eric Mandarin TX video, approved Oct 2026)
- Must not look like Henderson: no chrome extrusion, no glare bands, no gold.
- Font: **Jost** (@fontsource-variable/jost). Big titles are 700 uppercase, white with a subtle white→#e2e4e8 face gradient applied **per character**, and a soft drop shadow (0 6px 14px / 0 2px 3px). Put `perspective` on the text wrapper so per-char translateZ still works when the wrapper has a filter.
- Signature reveal: each letter flies in from depth (translateZ ≈ +520 → 0, blur 16 → 0, slight y drop), staggered about 0.035s. Exit: blur + fade + push back.
- Each card has a small "01 ——" index (Jost 500, 26px, tracked 0.3em, hairline drawing in), a big title, and tiny wide-tracked captions (Jost 500, 28px, 0.3em). Left-aligned at x≈84, top or bottom of the frame, wherever the car isn't.
- Hero: "LEXUS" thin and widely tracked, the year rising per-char from behind the roofline, the model in front, and the trim between hairlines.
- End card: background darkened about 30% plus top/bottom gradients. "LEXUS OF LAS VEGAS" thin tracked wordmark, then hairline, "#1 VOLUME" (big), "LEXUS DEALER" (bold tracked), "IN NEVADA" (tracked), hairline, and a footer with year/model/trim and "6600 W SAHARA AVE · LAS VEGAS".
- Don't draw the Lexus emblem. If Denham wants the logo, he supplies the file.

### Land Rover Henderson: "green-black & gold" (approved 2026-10-28, Defender 110 CPO reel)
- Palette from the locked LRH carousel: scrims in green-black #0B1410, accent gold #D9B26A, white type. No chrome extrusion, no glare bands, no slide counters, no script fonts, no associate names.
- Fonts: Archivo Variable (wdth 118, wght 800, uppercase) for titles; Inter 600 tracked 0.34em in gold for kickers; Inter 500 tracked 0.22em at 88% white for captions; a gold-to-transparent hairline under each title. Left-aligned at x≈150 (4K), top or bottom of the frame, wherever the car isn't, over a soft green-black gradient scrim.
- Hero: the year in huge gold-outlined Archivo (stroke ≈13px at 4K, 14% gold fill) **behind** the car, spun in with the spinning-counter reel; "CERTIFIED PRE-OWNED" kicker, model title and trim caption in front, lower-middle.
- CPO cards: a success-check (gold ring + stroke-drawn check) above "LAND ROVER APPROVED / CERTIFIED PRE-OWNED" (165-point inspection, history report), then a coverage card (1-yr/unlimited-mile, $0 deductible, roadside, transferable, 2-yr/100k upgrade) with a streaming caption.
- End card: footage darkened ~30% + top/bottom gradients, lockup at the **bottom**: gold rule, "YOUR LAND ROVER SPECIALISTS" (gold), "LAND ROVER" / "HENDERSON" wide-tracked wordmark (one gold shimmer sweep across HENDERSON), gold rule, "360 N GIBSON RD · HENDERSON, NV", "(702) 955-5700" in gold, small year/model/trim + "CERTIFIED PRE-OWNED" footer. Don't draw the Land Rover oval; he supplies the logo file if he wants it.
- Pacing: classy. Plain hard cuts on the beat, no punch-ins, flashes or speed ramps (hero and end card at a steady 0.5×); end card ≈4s on a bar line. **Use more clips than the first reel did** (it had 9): mix 2-beat and 3-beat cards so a 17s reel carries 11–13 different angles, and shorten the end card to ~3s if that is what it takes.

### Motion vocabulary: Transitions.dev (added 2026-10-28)
Denham wants the text/graphic motion in every reel built from the Transitions.dev recipes (github.com/Jakubantalik/transitions.dev, `transitions/` + `skills/transitions-dev/*.md`; licensed free for commercial use, don't redistribute the library). They are CSS UI transitions, so port each one into the seek-driven composition: same distances, blur amounts, durations, staggers and easing curves, evaluated at `window.seek(t)` instead of triggered by class changes. Shared ease unless noted: `cubic-bezier(0.22, 1, 0.36, 1)` ("out-expo-ish"). Scale the pixel values ×2 for the 2160×3840 canvas. These govern graphics only — footage still cuts hard (dealer rules in section 5).

| Recipe | Use in the reel | Tokens (desktop values) |
|---|---|---|
| **Texts reveal** (18) | Default card entrance: title + caption lines rise with blur, staggered. Exit is a plain quiet fade, no reverse travel. | 500ms, rise 12px, blur 3px, stagger 40ms/line; exit 200ms ease fade only |
| **Number pop-in** (02) | Any spec number (HP, LB-FT, wheel size, speaker count): each digit rises in from below, last two digits lag. | 500ms, 8px, blur 2px, stagger 70ms, ease `cubic-bezier(0.34,1.45,0.64,1)` (overshoot) |
| **Spinning counter** (26) | Hero year or one headline number when it should feel like an event: clipped vertical 0–9 reels spin then land, vertical-only blur that decays as each column settles. | 1400ms, ease `cubic-bezier(0.16,1,0.3,1)`, column stagger 90ms, spin blur 3px (feGaussianBlur stdDeviation "0 Y", never CSS blur) |
| **Text states swap** (04) | Changing one label in place between beats (old exits up+blur, new enters from below). | 150ms, 4px, blur 2px, ease-in-out |
| **Text swap soft** (34) / **Text morph** (33) | Cross-blur a value in place with no movement (34), or keep shared letters and morph only the changed part (33). | 200ms / 180ms, blur 2px, width eases 240ms |
| **Streaming text** (30) | Caption or CPO-benefit lines that "resolve" word by word instead of typing. | word gap 60ms, fade 350ms, blur 1px |
| **Panel reveal** (07) | A scrim, rule or badge plate sliding in behind text. | open 400ms / close 350ms, travel ≈ half the panel height, cross-blur 2px |
| **Skeleton reveal** (14) | Cross-fade + cross-blur between two stacked layers (e.g. hairline wireframe → real text). | cross-fade with matching blur |
| **Shimmer text** (15) | A slow highlight band across a wordmark or kicker during a hold (gradient clipped to the glyphs via `background-clip:text`). | 2000ms linear loop, band 400%; use at most once per reel, never on body copy |
| **Success check** (10) | The CPO / certification moment: check icon fades in, rotates from 80°, blur 10→0, Y-bob 40px, then the SVG path stroke-draws. | all 500ms; bob ease `cubic-bezier(0.34,1.35,0.64,1)`; path delay 80ms |
| **Card tilt** (19) | The 3D "card" feel on a spec plate or photo frame: rotateX/rotateY under `perspective(1000px)` drifting slowly during the hold, easing back flat over 1000ms. Skip its glare layer (mix-blend-mode breaks transparent renders). | perspective 1000px, return 1000ms |
| **Icon swap** (09) | Two icons in one slot: scale + blur cross-fade. | scale + blur, same slot |

Rules for porting: keep a transition's duration/stagger/ease exactly, convert `transition`/`@keyframes` into functions of t, apply blur with `filter: blur()` (fine in Playwright PNG renders), and never carry over `mix-blend-mode` glares or `prefers-reduced-motion` guards (irrelevant to a render). Pick the lower-overhead recipe when two fit (texts reveal over panel reveal, number pop-in over spinning counter) unless the beat calls for the bigger moment.

### All styles
- Legibility: put a soft dark gradient scrim behind any text over sky, bright ceilings or white paint. Place text where the car isn't; check every card over the real frame before rendering.
- Rankings are written "#1", never "Number one". No price or stock number unless asked.
- Avoid `mix-blend-mode` elements (e.g. overlay glare sweeps) in transparent-background renders: they composite as pale rectangles.
- Load the fonts explicitly before measuring or fitting text (`Promise.all([document.fonts.load('400 100px Family'), …])`, then `document.fonts.ready`). `document.fonts.ready` alone can resolve before a not-yet-used webfont loads, so fit() measures the fallback font and long lines overflow the frame.
- Render layers: `back` (behind the car, hero only) and `front`. Composite order: footage → back (masked by the car matte) → front. Two parallel Playwright processes render about 1 frame/s at 2160×3840.

## 7. Composite
- Python compositor: sequential ffmpeg readers per shot reading the **stabilized** clips (no LUT or color filter in the decode chain), cv2 warpAffine for zoom/punch, the flash, back layer × (1 − matte), front layer, and a raw RGB pipe into libx264 CRF 15.
- **Smooth slow-mo via optical flow:** map each output frame to a fractional source index (integrate the shot's speed curve; 0.5 source seconds per screen second for every shot). For fractional positions, compute OpenCV DIS flow (preset MEDIUM) both ways at 1/4 resolution, upscale the flow, backward-warp both neighbour frames to the in-between time with cv2.remap, and blend by position. During ramps above 1×, average several flow-interpolated sub-frames across a 180° shutter for motion blur. Don't use ffmpeg minterpolate at 4K (about 4.5 s/frame, single-threaded).
- Mattes (hero text-behind-car and mask-transition cutouts) must be made from the same stabilized clips the compositor reads, or the cutout drifts off the car.
- After rendering, scan consecutive-frame differences at low resolution: there should be zero near-duplicate frames.
- About 1.5–3 s/frame on the container; a full render takes 30–60 min, so run it in the background and verify the frame count when it ends.
- Preview 15–20 key frames (and a short render of any transition being changed) through the same compositor before the full render.
- For a fix confined to one section, re-render only those frames and splice them with a single re-encode.

## 8. Resolve (only when asked)
Two kinds: a **finished draft** (retime, stabilization and hero comp baked into V1) or, when he wants to stabilize and slow it himself, a **raw edit** (raw DJI clips on V1 at the edit's cut points, at 100%, so he can apply 50% with Ripple off and the cuts stay on the beat).
- **Playback must never be slow.** Resolve's script API cannot set the playback frame rate, and a project left at 24fps playback plays a 60fps timeline at 40% speed. Every time:
  1. Before importing anything, apply the saved project settings preset: `project.SetProjectSettingsPreset("Loomelic Vertical 4K 60p")` (2160×3840, 60fps timeline, 60fps playback). If it doesn't exist, set resolution/frame rate by script, then set **Playback frame rate = 60** in File › Project Settings › Master Settings with computer use (it's a text field), Save, and re-create the preset with `SaveCurrentProjectSettingsAsNewPreset`.
  2. After building, verify `project.GetSetting("timelinePlaybackFrameRate")` equals the timeline frame rate ("60"). If it doesn't, fix it in Project Settings before telling Denham it's ready.
- Image sequences import at 24fps by default: `SetClipProperty("FPS","60")` on PNG sequences **before** appending them, or they stretch 2.5×.
- **Graphics must be PNG sequences with alpha.** Resolve ignored the alpha in ffmpeg-made ProRes 4444 and showed black boxes. Import with `ImportMedia([{"FilePath": ".../%04d.png", "StartIndex": 0, "EndIndex": N}])`. Transfer them to the Mac as tar chunks under 24 MB (commits of larger files fail) and untar there.
- Run `AppendToTimeline` with `endFrame` = start + frame count (it's exclusive). Check every clip's start and duration, check there are no gaps, verify with `ExportCurrentFrameAsStill`, then call `SaveProject()`. No render job.
- On raw-edit timelines: no LUT or grade on any clip, `RetimeProcess` = Optical Flow, a marker at each cut with the intended speed and card, the finished reel on a disabled reference track, and named tracks.

## 9. Delivering
- **Send only the 1080p version** (in chat, and into the car folder) unless he asks for 4K. Keep the 4K master in the container in case he does.
- device_commit_files: keep each file under ~24 MB (larger ones fail with HTTP 400), at most 50 files and 100 MB per call. Split big files into 3-second MPEG-TS segments, commit them, then rejoin on the Mac: `ffmpeg -i "concat:a.ts|b.ts|..." -c copy -bsf:a aac_adtstoasc out.mp4`.
- Save the composition HTML, renderer, compositor and music bed to `_reel_work/project/` so the edit can be revised.

## 10. Before delivering
- Sheet one frame from every shot of the final file (extract frames one at a time; opening many 4K decoders at once runs out of memory). Check text placement, legibility, the car matte edge, the dealer name, that no angle repeats, the duration and frame count, and that no frames are doubled.
- Confirm again that every shot is stabilized (verified correction, section 3) and plays at exactly 50%. A reel with an unstabilized or full-speed shot doesn't ship.
- Reply with what's in the reel, where it is, any judgment calls, and sources.

## Rules learned (append here)
- 2026-10: Lexus of Henderson end line is "Largest Lexus dealer in Nevada". Lexus of Las Vegas uses "#1 volume Lexus dealer in Nevada".
- 2026-10: Don't repeat angles. Do use a foreground pillar pass as a wipe.
- 2026-10-28: **No color grade anymore.** Denham retired the house LUT (sat_REC709 + Primary Balance + HDR). Footage ships as-shot; never apply a LUT, eq or stand-in grade, and never ask him for a LUT file.
- More cuts beat fewer.
- 2026-10: Each dealer gets its own graphics style. Lexus of Las Vegas = the clean Jost style from the Eric Mandarin TX video, never Henderson's chrome look.
- 2026-10: Denham hates choppy footage and a disconnected end card. Slow-mo must be optical-flow smooth (no doubled frames), the end card must land on a bar line in the music, and the shot before it ramps into it.
- 2026-10: Deliver 1080p only unless he asks for 4K.
- 2026-10: Every Resolve project must play back at real speed: apply the "Loomelic Vertical 4K 60p" preset and verify playback frame rate = timeline frame rate before handing it over.
- 2026-10-28: Graphic/text motion is built from the Transitions.dev recipes (section 6, "Motion vocabulary"): texts reveal for card entrances, number pop-in for spec digits, success check for the CPO moment, etc. Use their exact timings and easings.
- 2026-10-28: Land Rover Henderson reels: classy pacing, plain hard cuts (no punch-ins, flashes or speed ramps), graphics in the green-black & gold LRH style (section 6). CPO cars get two CPO cards (approved/inspection, coverage).
- 2026-10-28: Denham approved the LRH Defender 110 CPO reel ("I like everything") with one note: put more clips in future reels. Favor more, shorter cuts (2-beat cards) over fewer long ones; use more of the footage he shot.
- 2026-10-28: **Every clip slowed to exactly 50% and stabilized, verified before compositing** (Denham, on the 2025 Range Rover SE CPO reel). The Mac shell's ffmpeg vid.stab silently does nothing (it can't read its own .trf), so stabilize in the container and prove the correction landed on every clip.