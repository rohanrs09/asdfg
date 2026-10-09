# ADF Day 1 Reel – Prompt Pack

Oct 9, 2026 · @Rohan

Generate the voice once, make 13 short clips in Seedance 2.0 with the prompts below, and join them in a video editor: about 1:50 in total, 9:16, faceless.

## How to use

Do it in this order, because every clip length follows the narration.

1. **Voice first.** Generate the narration with the Voice prompt below and export it as WAV.
2. **13 clips.** In Seedance 2.0 (Higgsfield), set 9:16, paste the Master style prompt, then one clip prompt. Attach the icon images named in that clip as @image references.
3. **Edit.** Lay the narration on the timeline, drop the clips in order using the Edit timeline table, then add captions, music and sound effects.

AI video models draw logos and on-screen words badly by themselves. That is why every clip uses real icon files as references and keeps text to two or three words; the exact captions go in the editor.

**Prepare these 8 image files once** (PNG, transparent background, about 1024 px):

| Ref | Image | Where to get it |
| --- | --- | --- |
| @image1 | Azure Data Factory icon | Microsoft's free Azure Architecture Icons download |
| @image2 | SQL Server / Azure SQL Database icon | same download |
| @image3 | CSV file icon (green document, "CSV") | any icon site, or draw in Canva |
| @image4 | REST API icon (a {  } box) | any icon site, or draw in Canva |
| @image5 | Azure Data Lake Storage Gen2 icon | Azure Architecture Icons |
| @image6 | Azure Databricks icon | Azure Architecture Icons |
| @image7 | Azure Synapse Analytics icon | Azure Architecture Icons |
| @image8 | Microsoft Fabric icon | Microsoft Fabric documentation |

I could not run Seedance from here, so these prompts are untested. Expect one or two retries on some clips; the last section lists the usual fixes.

## Master style prompt

Paste this block first in all 13 clips. It keeps colours, icon style and camera feel identical, so the clips join into one video.

```text
GLOBAL STYLE LOCK
FORMAT: vertical 9:16, 1080x1920, 30 fps. Keep key visuals between 15% and 85% of frame height; main action in the top 60%.
STYLE: Motion Graphics Hybrid, tech explainer. Near-black background (#05070D) with a faint dot grid at 2% opacity drifting slowly upward. Floating glass 3D panels, rounded-square icon tiles with a thin glowing edge, thin glowing connector lines, data packets as small cyan light beads.
PALETTE: Azure blue #0078D4, cyan #50E6FF, white #FFFFFF for text, amber #FFB347 for triggers and schedules, red-orange #FF7A5C for problems, green #6EE7A8 for success.
LIGHTING: all light is internal (glowing icons, panels, lines). Cyan emitters 7000K, azure fill 6500K, amber accents 3000K. No ambient fill, deep shadows between elements.
CAMERA: slow, smooth, motivated moves only; never handheld. No cuts inside a clip. Shallow depth of field: foreground tiles sharp, background grid softly blurred.
MOTION: every frame has visible motion (drifting grid, pulsing glows, flowing packets). Nothing is static.
TEXT: only the exact words written in quotes inside the scene, two or three words at most, white, bold geometric sans-serif (Poppins style), correctly spelled, horizontally centered. No other letters anywhere in the frame.
ICONS: use the attached reference images exactly as given. Do not redraw, recolor, restyle or invent logos.
NEVER SHOW: faces, people, hands, watermarks, subtitles, interface chrome, extra logos, gibberish text, stock footage.
```

## Voice prompt

A clean male Hinglish narrator comes from a neural voice tool, not from a video model. Use ElevenLabs (or Microsoft Edge's "Prabhat" / "Madhur" voices as a free fallback).

**Voice to pick:** in ElevenLabs' Voice Library filter for Indian accent, male, young (20–30), conversational or educational. Audition 3 voices with the first sentence and keep the one that says "data", "pipeline" and "orchestration" clearly.

**Delivery to ask for** (in a voice-design box if the tool has one; never paste it into the script): friendly Indian male teacher in his late twenties, warm, clear, medium pace, a small smile in the voice, crisp consonants, English technical terms in a clean Indian-English accent.

| Setting | Value |
| --- | --- |
| Model | Multilingual v2 (or a newer multilingual model your plan offers) |
| Stability | 45–55% (natural, not flat) |
| Similarity | 75% |
| Style exaggeration | 10–20% |
| Speaker boost | On |
| Speed | 1.0 (0.95 if it sounds rushed) |
| Export | WAV, 48 kHz, no music inside |

Paste the script below. The pause tags make the narration match the clip lengths; generate each block as its own take if one sounds off, then join them.

```text
Data Engineering mein ek common problem hai… <break time="0.4s" /> data ek jagah nahi hota. <break time="0.9s" />

Maan lo ek company ka customer data SQL Server mein hai… <break time="0.4s" /> orders CSV files mein hain… <break time="0.4s" /> aur kuch data kisi REST API se aa raha hai. <break time="0.9s" />

Ab in sab data sources ko ek central data platform tak regularly kaise laoge? <break time="0.5s" /> Manually? <break time="0.3s" /> Obviously nahi. <break time="0.9s" />

Yahin pe aata hai… <break time="0.5s" /> Azure Data Factory — ADF. <break time="0.9s" />

ADF ko simple language mein samjho… ye ek cloud-based data integration and orchestration service hai. <break time="0.8s" />

Matlab ADF different data sources se data ko connect kar sakta hai… <break time="0.4s" /> move kar sakta hai… <break time="0.4s" /> aur poore data workflow ko schedule aur orchestrate kar sakta hai. <break time="0.9s" />

For example… suppose mujhe SQL Server se daily orders uthakar Azure Data Lake mein load karne hain. <break time="0.5s" /> ADF pipeline ye poora workflow automate kar sakti hai. <break time="0.9s" />

Aur sirf ek baar run karna nahi… <break time="0.5s" /> aap ise schedule kar sakte ho… <break time="0.5s" /> dependencies laga sakte ho… <break time="0.5s" /> failure handle kar sakte ho… <break time="0.5s" /> aur monitor bhi kar sakte ho. <break time="0.9s" />

Ab ek important point… <break time="0.5s" /> ADF khud har type ka data processing engine nahi hai. <break time="0.5s" /> Iska main strength hai… connect, <break time="0.3s" /> move, <break time="0.3s" /> and orchestrate. <break time="0.9s" />

Heavy transformation ke liye aap Databricks, Azure Synapse, SQL, Fabric jaise services ko ADF workflow ke andar trigger kar sakte ho. <break time="0.9s" />

So simple way mein yaad rakho… <break time="0.5s" /> ADF is the control center of your data pipeline. <break time="0.5s" /> It connects your sources… <break time="0.3s" /> moves the data… <break time="0.3s" /> and coordinates what happens next. <break time="0.9s" />

Aur ab jab humein pata hai ADF karta kya hai… <break time="0.5s" /> next question hai… <break time="0.4s" /> ADF ke andar actually kaun-kaun se components milke ye pipeline banate hain? <break time="0.9s" />

Chalo next samajhte hain… <break time="0.4s" /> ADF ke Main Components.
```

**Clean-audio rules:** record no music or effects in this file; if there is hiss, run it through a noise remover once, then normalize to −16 LUFS before editing. Add music and effects only in the editor.

## Clips 1–3 · The problem

Each clip: paste the Master style prompt, then the block below. The narration line is for your timing only and is not part of the prompt.

### Clip 1 · Hook · 8 s

Narration: "Data Engineering mein ek common problem hai… data ek jagah nahi hota."

```text
TOPIC: Data lives in separate systems, never in one place
PLATFORM: Instagram Reels 9:16
DURATION: 8s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Impossible Scene:
The first frame is already arresting: three glowing icon tiles (@image2 database, @image3 CSV file, @image4 API box) float far apart in a black void, each drifting away from the others. Thin red-orange dashed lines between them snap and fade like cut cables. Shallow depth of field: CSV tile sharp in the center, the other two softly blurred at the frame edges. Camera: imperceptible push-in.

BEAT 1 (0:02-0:05):
Camera starts a slow partial orbit (45 degrees) around the CSV tile; parallax widens the gaps between the three tiles. A spark leaps from the database tile toward the API tile, dies halfway, and the gap goes dark. Dot grid drifts slowly upward behind.

BEAT 2 (0:05-0:08):
Orbit settles into a slow push-in. The text "SCATTERED DATA" materializes letter by letter at about 60% frame height, white, and holds for 2 seconds (the linger). All three tiles keep drifting and pulse red-orange once on the last letter.

MOTION NOTE: tiles drift at different speeds and rotate under 5 degrees; broken lines flicker; grid drifts upward; no static element. Camera rates: imperceptible, then slow, then slow.

LIGHTING: tile emitters 6500K azure (database), 5500K green (CSV), 6000K violet (API); broken lines glow 3000K red-orange; no ambient fill, hard-edged shadows.

SOUND DIRECTION (narration-first): narration starts at 0:00. Music bed 10-15% under the voice: low dark synth pad, no drums, no melody. SFX: faint electric crackle each time a spark dies, soft low hum under the void, one soft whoosh as the text appears.

MATERIAL REFS: @image2 database icon, @image3 CSV icon, @image4 API icon.
```

### Clip 2 · The three sources · 10 s

Narration: "Maan lo ek company ka customer data SQL Server mein hai… orders CSV files mein hain… aur kuch data kisi REST API se aa raha hai."

```text
TOPIC: Three different data sources: SQL Server, CSV files, REST API
PLATFORM: Instagram Reels 9:16
DURATION: 10s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Data Reveal:
Black screen, then a white extruded 3D number "0" spins up through 1 and 2 and freezes on "3" in electric blue, pulsing once. The words "3 SOURCES" fade in under it. Camera: slow push-in. Shallow depth of field: number sharp, grid behind it soft.

BEAT 1 (0:02-0:05) - SQL Server:
The number breaks into cyan particles that fly down and form tile @image2 (database) at top-center. Below it a glass panel fills line by line with tiny cyan table rows, like a customer table. Camera: slow vertical drift downward.

BEAT 2 (0:05-0:07.5) - CSV files:
Camera drifts down to tile @image3 (CSV file), which slides in from the left. Rows of cells separated by thin commas scroll upward inside its glass panel like incoming orders.

BEAT 3 (0:07.5-0:10) - REST API:
Camera drifts to tile @image4 (API box), which slides in from the right. Curly-brace lines of data stream out of its panel toward the camera. Last frame: all three tiles in a loose vertical zigzag, each pulsing once; camera lingers 1.5 seconds.

MOTION NOTE: every tile keeps a slow bobbing float; cell rows and data streams never stop; grid drifts upward. Camera rates: slow, slow, slow.

LIGHTING: number emitter 7000K cyan-white; database tile 6500K azure; CSV tile 5500K green; API tile 6000K violet; no ambient fill.

SOUND DIRECTION (narration-first): narration starts at 0:00. Music bed 10-15% under the voice, same dark pad as clip 1. SFX: mechanical ticker sound during the number spin (0:00-0:02), a soft digital chime as each tile lands (0:03, 0:05.5, 0:08), faint data-stream hiss.

MATERIAL REFS: @image2 database icon, @image3 CSV icon, @image4 API icon.
```

### Clip 3 · The manual problem · 8 s

Narration: "Ab in sab data sources ko ek central data platform tak regularly kaise laoge? Manually? Obviously nahi."

```text
TOPIC: Moving data from many sources to one platform by hand does not work
PLATFORM: Instagram Reels 9:16
DURATION: 8s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Visual Metaphor:
Extreme close-up of a single cyan data packet crawling along a dashed line, sputtering and slowing. Shallow depth of field: packet sharp, everything behind it a soft blur of grid dots. Camera: imperceptible push-in.

BEAT 1 (0:02-0:05) - Pull-back reveal:
Camera pulls back slowly to show the packet is one of many on three dashed lines coming from three small tiles (@image2, @image3, @image4) at the top, all heading toward a dark empty platform tile (@image5 shown at 20% brightness, dim and inactive) at the bottom. The distance feels huge. Packets move in stop-and-go jerks.

BEAT 2 (0:05-0:08) - Manual work does not scale:
Near the platform, a looping conveyor of identical small tiles shows copy, paste, repeat, while a clock face spins fast and red-orange warning flickers pulse. At 0:06 a rapid crash zoom lands on a red-orange cross sign over the conveyor, with the text "MANUAL? NO" in white under it. Hold the final frame 1 second.

MOTION NOTE: packets stutter and stall; clock hands spin; warning flicker at 4 pulses per second; grid drifts. Camera rates: imperceptible, slow pull-back, rapid crash zoom (one time only).

LIGHTING: packets 7000K cyan; source tiles 6500K azure; dim platform 5000K at 20% intensity; warning flicker and cross sign 3000K red-orange; no ambient fill.

SOUND DIRECTION (narration-first): narration starts at 0:00. Music bed 10-15%: the dark pad with a slight tension rise toward 0:06. SFX: soft stuttering tick for packets, low warning buzz (very quiet) with the flicker, one short muted thud on the crash zoom.

MATERIAL REFS: @image2 database icon, @image3 CSV icon, @image4 API icon, @image5 data lake icon.
```

## Clips 4–6 · Reveal and definition

### Clip 4 · ADF reveal · 5 s

Narration: "Yahin pe aata hai… Azure Data Factory — ADF."

```text
TOPIC: Reveal of Azure Data Factory (ADF) as the answer
PLATFORM: Instagram Reels 9:16
DURATION: 5s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Visual Metaphor:
Pure black frame, then one cyan point of light ignites in the center and expands into a shockwave ring that sweeps outward, revealing the dot grid for the first time. Shallow depth of field: ring edge sharp, grid dots soft. Camera: imperceptible push-in.

BEAT 1 (0:02-0:04):
Out of the light, tile @image1 (Azure Data Factory icon) rises into the center, tilted 10 degrees, turning slowly to face the camera. Two more rings pulse outward from it at 0.7-second intervals. Camera: slow push-in.

BEAT 2 (0:04-0:05):
The text "AZURE DATA FACTORY" appears in white on two lines under the icon. Camera settles and lingers for the last second.

MOTION NOTE: rings expand continuously, icon floats with a 1% vertical bob, grid drifts upward, glow pulses once per second. Camera rates: imperceptible, then slow, then still.

LIGHTING: ignition point 7500K cyan-white; icon tile emits 7000K cyan with a 6500K azure halo; rings 6500K; no ambient fill, deep black around the glow.

SOUND DIRECTION (narration-first): narration starts at 0:00 with a short beat of silence. Music bed 10-15%, swelling to 30% for one second at 0:02, then back down. SFX: low riser 0:00-0:02, one clean soft impact at 0:02, light shimmer tail under the text.

MATERIAL REFS: @image1 Azure Data Factory icon.
```

### Clip 5 · What ADF is · 8 s

Narration: "ADF ko simple language mein samjho… ye ek cloud-based data integration and orchestration service hai."

```text
TOPIC: ADF is a cloud-based data integration and orchestration service
PLATFORM: Instagram Reels 9:16
DURATION: 8s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Impossible Scene:
Tile @image1 (ADF icon) floats at center. Above it a cloud outline assembles itself from glowing circuit-board lines, trace by trace, hovering in the black void. The text "CLOUD" fades in under the icon. Shallow depth of field: icon sharp, circuit lines slightly soft. Camera: slow partial orbit (45 degrees) that continues through the whole clip.

BEAT 1 (0:02-0:05) - Integration:
The cloud dissolves into particles. From the left and right edges, many small dots stream in along curved lines and merge into the ADF tile as one. The text "INTEGRATION" replaces the first word.

BEAT 2 (0:05-0:08) - Orchestration:
The stream calms. Around the ADF tile, five small round nodes linked by thin lines light up one after another in a clockwise order, like a flow being conducted. The text "ORCHESTRATION" replaces the second word and holds for the last 1.5 seconds.

MOTION NOTE: particles never stop flowing; nodes pulse after lighting; grid drifts upward. Camera rates: slow throughout.

LIGHTING: cloud lines 7000K cyan; merging dots 6500K azure; orchestration nodes 3000K amber, each lighting in turn; ADF tile 7000K cyan; no ambient fill.

SOUND DIRECTION (narration-first): narration starts at 0:00. Music bed 10-15%, same pad. SFX: soft airy shimmer while the cloud forms, a gentle converging whoosh at 0:02, five soft ascending tones as the nodes light up between 0:05 and 0:07.

MATERIAL REFS: @image1 Azure Data Factory icon.
```

### Clip 6 · Connect, Move, Orchestrate · 10 s

Narration: "Matlab ADF different data sources se data ko connect kar sakta hai… move kar sakta hai… aur poore data workflow ko schedule aur orchestrate kar sakta hai."

```text
TOPIC: What ADF does: connect to sources, move data, orchestrate the workflow
PLATFORM: Instagram Reels 9:16
DURATION: 10s
STYLE: Motion Graphics Hybrid
SOUND PRIORITY: narration-first

HOOK (0:00-0:02) - The Satisfying Loop:
Tile @image1 (ADF icon) sits at center. Three cyan lines extend from it to three small tiles at the top (@image2 database, @image3 CSV file, @image4 API box) and lock in with a soft plug-in glow, then pulse outward and inward in a smooth hypnotic loop. The text "CONNECT" fades in at 0:00.5. Camera: imperceptible push-in. Shallow depth of field: ADF tile sharp, top tiles slightly soft.

BEAT 1 (0:02-0:03.5):
The three connection lines finish pulsing; each source tile flashes green once to show it is connected.

BEAT 2 (0:03.5-0:06.5) - Move:
Text changes to "MOVE". Cyan data packets stream from the three source tiles into the ADF tile and out of its bottom toward tile @image5 (data lake) which fades in at the bottom. Camera: slow vertical drift downward following the flow.

BEAT 3 (0:06.5-0:10) - Orchestrate:
Text changes to "ORCHESTRATE". Beside the flow, three round nodes in a vertical chain light up in order, and an amber clock ring sweeps once around the ADF tile like a schedule. Camera: slow partial orbit (45 degrees). Last frame holds for 1 second.

MOTION NOTE: packets and the clock ring never stop; the tile floats with a 1% bob; grid drifts upward. Camera rates: imperceptible, slow, slow.

LIGHTING: ADF tile 7000K cyan; connection lines 6500K azure; packets 7000K cyan; connection flash 5500K green; clock ring and chain nodes 3000K amber; no ambient fill.

SOUND DIRECTION (narration-first): narration starts at 0:00. Music bed 10-15%. SFX: soft plug-click as each line locks (0:00.5, 0:01, 0:01.5), a smooth flowing whoosh loop during the move (0:03.5-0:06.5), quiet amber clock ticking and three soft rising tones during the orchestrate beat.

MATERIAL REFS: @image1 ADF icon, @image2 database icon, @image3 CSV icon, @image4 API icon, @image5 data lake icon.
```
