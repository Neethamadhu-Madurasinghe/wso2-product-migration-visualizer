# WSO2 product migration visualizer

Simple 2D animations that explain WSO2 product migrations to customers, step by step.

**Products covered today: WSO2 API Manager** (including WSO2 Identity Server when it is API Manager's Key Manager). WSO2 Identity Server migrations as a product of their own are planned; see [Limitations](#limitations).

## Getting started

### What it does

You pick a migration (source version, target version, deployment and platform), and the page plays an animated diagram of how that migration runs:
- production keeps running while the new version is built and tested on fresh databases;
- the downtime or freeze window starts;
- the data is copied and migrated (SQL scripts, migration clients, temporary nodes where needed);
- the new version starts and traffic is switched over, with the old setup kept for rollback.

A caption explains each step, a status panel shows whether APIs are available and changes are allowed, and moving dots show API calls and data changes. The page picks the documented migration approach for the version pair you chose, so the steps match the WSO2 migration guides.

You can also hand the result to a customer as a standalone HTML file, an animated GIF or a silent MP4 video.

### How to use it

1. **Open `index.html` in a browser** (Chrome, Edge, Safari or Firefox). Nothing needs to be installed or built. The page needs internet access to load GSAP from cdnjs.
2. **Fill in the setup screen:**
   - current (source) and target API Manager version; only targets with a documented path are offered;
   - deployment: all-in-one or distributed;
   - target platform: virtual machines or Kubernetes;
   - Key Manager: API Manager's resident Key Manager or WSO2 Identity Server (offered for sources up to 4.2.0);
   - whether downtime is acceptable.

   The approach that will be used is shown under the form. Click **Show the migration ▶**.
3. **Watch and control the animation:**
   - **Play / Pause, Prev / Next, Restart**, or click a step button to jump to that step;
   - **speed** 0.5×–2×;
   - **With downtime / Without downtime (freeze window)** switches between the two ways of running the migration at any time;
   - **🔊 Narrate** reads each step aloud (see [Narration](#narration));
   - **Change setup** goes back to the setup screen.
4. **Share it:**
   - **⬇ Export for customer** downloads one standalone HTML file with this setup locked in. The customer can play it, change the speed and switch downtime/freeze, but cannot change versions. It works offline. See [Customer deliverable](#customer-deliverable).
   - **🎞 GIF** downloads the whole animation as an animated GIF at the current speed, for email, slides or tickets. See [GIF and video export](#gif-and-video-export).
   - **🎬 Video** downloads the whole animation as a silent MP4 (25 fps) at the current speed: smoother and sharper than the GIF. For Google Slides, upload the MP4 to Google Drive and use Insert → Video → Google Drive; PowerPoint and Keynote take the file directly.
   - The URL updates after setup, so you can also share or bookmark the link, for example `index.html?src=3.2.0&tgt=4.7.0&dep=aio&plat=vm&km=resident&downtime=yes`.

### Files

| File | Purpose |
|---|---|
| `index.html` | The whole tool: HTML, CSS and JavaScript in one file. GSAP is loaded from cdnjs; the GIF encoder and the MP4 muxer are loaded from jsdelivr only when you make a GIF or a video. |
| `README.md` | This document. |

## Limitations

- **WSO2 Identity Server migration is not covered as a topic of its own.** The tool is about API Manager migrations. A standalone IS migration (IS as an identity provider, not as a Key Manager) is not shown yet; it is planned as the next product.
- **IS appears only when IS is the Key Manager for API Manager**, and even then only at a high level: the IS migration is one "IS migration client" step on the target IS version, followed by connecting it to API Manager (and, for IS 7.x, the Key Manager conversion). The IS migration's own prerequisites, configuration migration, user stores, tenants, customizations and version-by-version details are not shown. Use the IS migration guides for those.
- **Distributed deployments are one generalized pattern** (Gateway, Control Plane, Traffic Manager). Real deployments with more node types, multiple gateways or separate Key Manager nodes are not drawn individually.
- **The steps are explanatory, not a runbook.** Commands and script names are shown to make the flow clear; some are placeholders (see [Accuracy notes](#accuracy-notes-to-check-before-showing-customers)). Always follow the official migration documentation for the actual procedure.
- **Some version pairs are not offered:** pairs with no documented direct path (for example 3.2.0 / 4.0.0 / 4.1.0 → 4.3.0 / 4.4.0) are left out of the target list.
- **Internet access is needed** to load GSAP in `index.html` (the exported customer file has GSAP inlined) and to load the GIF encoder or MP4 muxer.
- **GIFs and videos have no audio and no caption text**, only the diagram, the title and the step name. The narration voice comes from the browser's speech engine, which cannot be recorded.
- **Video export needs Chrome or Edge** (WebCodecs H.264 encoder). Other browsers show a message instead.

## What it shows

The setup screen asks for the source version, the target version, the deployment type (all-in-one or distributed), the target platform (VMs or Kubernetes), the Key Manager, and whether downtime is acceptable. The page then picks the migration approach and builds the matching animation:

| Source | Target | Approach | Steps after the DB copy |
|---|---|---|---|
| any source up to 4.1.0 | 4.2.0 | Direct migration with migration clients | Same as the row below, with `-DmigratedVersion=4.2.0` added to the API-M client |
| 3.2.0, 4.0.0, 4.1.0 | 4.5.0, 4.6.0, 4.7.0 | Direct migration with migration clients | SQL script on SHARED_DB → identity client (`-Dmigrate -Dcomponent=identity`) → API-M client (`-Dmigrate -DmigrateFromVersion=<src>`), both clients changing both DBs |
| 4.2.0 or later | any later version from 4.3.0 | Zero-data migration | SQL script on APIM_DB only |
| 2.6.0, 2.7.0, 3.0.0, 3.1.0 | 4.3.0 – 4.7.0 | Two-step migration via 4.2.0 | Temporary APIM 4.2.0 all-in-one node (a VM for Kubernetes targets): SQL script on SHARED_DB → identity client → API-M client (`-DmigrateFromVersion=<src> -DmigratedVersion=4.2.0`) → node removed (not tested) → 4.2.0 → target SQL script on APIM_DB |

Other pairs have no documented direct path, so the setup screen only offers valid targets. These rules match the WSO2 API Manager migration documentation (direct-migration catalog and the "upgrading to APIM 4.x" guides), checked 2026-10-16.

**Common steps:** production running → build target → **test target use cases with fresh DBs** → stop (downtime) or freeze → copy DBs → *approach steps* → start and test target → switch traffic. After the switch, the old setup is greyed out and a "↩ Rollback option if needed" badge appears in its header.

**Fresh-DB test:** applies to every version pair. The target gets the migrated configs and customizations, and starts against new empty databases. It goes through basic API Manager testing (portals, create/publish an API, subscribe, generate a token) and the customer's use cases are validated; the two-line badge reads "✓ Basic tests & / use cases passed". No traffic is sent to the backend during this test. When the tests pass, it is stopped and the fresh DBs are discarded. This happens before the maintenance window or freeze, so it adds no downtime.

**Downtime vs freeze window:** the downtime answer on the setup screen only sets where the **With downtime / Without downtime (freeze window)** switch starts. You can flip the switch at any time while the animation plays.

**Traffic and status:** API calls flow client → load balancer → gateway → **backend**. Purple dots are only data changes (new APIs, apps, subscriptions) or migration writes. A status panel shows API availability, whether changes are allowed, and which version serves traffic. The entry point is labelled **Load balancer / DNS**, since it can be either. The JAR full file names appear as hover tooltips on the JAR icons.

### Distributed deployments

Shown as one generalized pattern (the page says so on screen):
- **Gateway:** serves API calls (LB → Gateway → backend). It connects to the databases only for versions below 4.5.0.
- **Control Plane:** always connected to SHARED_DB and APIM_DB. Data changes come from here, and the migration clients run here because it is connected to all the databases.
- **Traffic Manager:** connected to the databases.
- **Start order after migration:** Control Plane, then Traffic Manager (connects to the DBs), then Gateway.
- Node-to-node links (Gateway–CP, CP–TM, TM–Gateway) are drawn as dotted lines.

### Kubernetes targets

Whatever the source platform, migration clients can't run inside the Kubernetes deployment. When the approach uses clients, the animation adds two steps:
- **Create a temporary VM:** an all-in-one APIM target-version node connected to the copied DBs. All-in-one is enough even when the target is distributed.
- **Remove the temporary VM:** after both clients finish, the VM is removed and only then does the Kubernetes deployment connect to the DBs and start.

Script-only (zero-data) migrations need no temporary VM.

### WSO2 IS as Key Manager

Setup option for sources up to 4.2.0, plus which DBs are shared: SHARED_DB + APIM_DB, or SHARED_DB only. Source IS versions: 2.6.0 → IS-KM 5.7.x, 2.7.0 → IS-KM 5.8.x, 3.0.0 → IS-KM 5.9.0, 3.1.0 → IS-KM 5.10.0, 3.2.0 → IS 5.10.0, 4.0.0/4.1.0 → IS 5.11.0, 4.2.0 → IS 6.x. (See [Limitations](#limitations): the IS side is shown at a high level only.)
- **Target APIM 4.2.0 → IS 6.1.0 (merged):** the DBs stay shared. First the IS migration client runs on IS 6.1.0 against the shared DBs. Then only the API Manager component migration runs (API-M client); the SHARED_DB script and the APIM identity client are skipped.
- **Target APIM 4.3.0+ → IS 7.x (split):**
  1. The DBs are split per product, so shared ones are duplicated and you end with 4 DBs.
  2. IS migrates on its own (IS migration client on IS 7.x).
  3. APIM migrates with its usual flow for the version pair.
  4. IS 7.x is connected as a third-party KM. The APIM ↔ IS Key Manager link is drawn **grey**: it is not usable for the migrated data yet.
  5. Both are stopped and the **Third-party to resident conversion** SQL script runs on APIM_DB. The Key Manager link then turns bright (IS green). The fresh-DB test keeps the bright link, since there is no migrated data to convert there.
  6. Both start, and IS 7.x works like the resident KM.
- Two-step sources are unchanged on the APIM side; IS goes straight to 7.x and is attached to the final target.
- On Kubernetes, the IS migration runs on a temporary IS VM (below the target row), removed afterwards.
- The fresh-DB test covers APIM and IS together.

### URL parameters

- `?src=3.2.0&tgt=4.7.0&dep=aio|dist&plat=vm|k8s&km=resident|is&kmdb=both|shared&downtime=yes|no` skips the setup screen. The page updates the URL after setup, so you can share the link.
- `&step=N`: open paused at the end of step N.
- `&at=identityRun`: open paused just after a timeline label (useful for screenshots).

### Accuracy notes to check before showing customers

- IS as KM: the IS migration is shown with the same arguments as the APIM identity step (`-Dmigrate -Dcomponent=identity`), and the conversion script name `km-conversion.sql` is a placeholder. Check both against the IS migration and KM conversion docs.
- 2.7.0: the migration docs have no "upgrading from 2.7.0 to 4.2.0" guide (only 2.6.0, 3.0.0 and 3.1.0). This affects both 2.7.0 → 4.2.0 and the two-step path. 2.7.0 is included based on CRE input and uses the same steps.
- 3.2.0 / 4.0.0 / 4.1.0 → 4.3.0 / 4.4.0 is not offered: the docs have no direct path. Decide whether these should also go via 4.2.0.
- Distributed on VMs: the clients are shown running on the Control Plane. An all-in-one node can also be used for this; only the Kubernetes path shows the temporary all-in-one VM.
- Distributed: the Gateway-to-DB rule (below 4.5.0 yes, 4.5.0+ no) comes from CRE input and has not been checked against the deployment docs per version. The TM link is drawn to SHARED_DB only, to stand for "connected to the databases".
- Freeze window: in an all-in-one node the key manager runs on the same node, so tokens issued during the freeze are written to the source DB and are not in the copy. Decide how to explain token re-issue, or show it.
- Client approach: the optional pre-migration validation run (`-DrunPreMigration`) and re-indexing are not shown.

## Customer deliverable

In the player, **⬇ Export for customer** downloads one standalone HTML file (about 165 KB), named for example `apim-migration-3.2.0-to-4.7.0-acme.html`:
- The current setup is locked in: versions, deployment, platform and Key Manager. There's no setup screen, export, GIF or video button, and URL parameters are ignored.
- The customer can switch **With downtime / Without downtime (freeze window)**, change the speed, play, pause, step through and use narration. The switch starts at the value chosen at export.
- An optional customer name is shown as "Prepared for …".
- GSAP is inlined, so the file works offline. If GSAP can't be fetched at export time, the file keeps the CDN link instead.

How it works: the page keeps a copy of its own source from load time (`PRISTINE`). The export inserts `window.MIGRATION_LOCK = {...}` before the main script and swaps the GSAP `<script src>` for the inlined library.

Note for editing: the main inline script must never contain a literal closing script tag. Write it as `<\/script>`, or the browser ends the script block there.

## Narration

**🔊 Narrate** (next to the speed selector) reads each step's caption aloud, so the viewer can watch the diagram instead of reading. It is off by default (browsers block speech until the viewer clicks) and is also in the exported customer file.
- It uses the browser's built-in speech engine (`speechSynthesis`): no audio files and no API keys, and it works offline. How good the voice sounds depends on the browser and OS. Neural voices in Edge, Chrome's "Google" voices and macOS enhanced voices sound best. The voice picker lists English voices, best ones first, and remembers the choice per browser.
- **The voice sets the pace.** At the start of each sentence, the timeline's `timeScale` is set so the rest of the step's animation lasts about as long as the rest of the narration. It never runs faster than the chosen speed. If the voice is still talking when the next step is reached, the timeline waits just before it (`narHold`). The speaking rate is measured as it plays and used for the next estimates.
- Play/Pause, Prev/Next, the step buttons and Restart control the voice too. Resume restarts the current sentence, because `speechSynthesis.pause()` is unreliable across browsers. The speed selector also sets the speaking rate (0.5×–2×).
- Captions are spoken one sentence at a time, because Chrome cuts off long utterances. Sentences over 25 words are also split at `;` and `:`. A watchdog moves on if the engine never reports the end of a sentence.
- `spokenText()` adapts the caption for listening: it spells out `APIM`, `IS`, `SQL` and `*_DB` letter by letter, reads `→` as "to" and ` / ` as "or", and reads `-D` flags as words, e.g. "migrate from version set to 3.2.0". When adding captions with new abbreviations, add them there.

## GIF and video export

The **🎞 GIF** and **🎬 Video** buttons (engineer page only; hidden in exported customer files) render the whole animation, from step 1, and download it as `apim-migration-<src>-to-<tgt>-<dep>-<plat>[-is-km]-<mode>-<speed>x.gif` or `.mp4`.

Shared by both:
- When pressed, narration is turned off and the speed in use is locked in: a 2× export plays twice as fast as 1×. All controls are disabled until it finishes; a status card shows progress and has a Cancel button. Afterwards the page goes back to where it was.
- The recorder steps the timeline and the traffic dots on a fixed clock instead of real time, so no frames are dropped on slow machines.
- Each frame is the SVG drawn on a canvas at 0.9× the viewBox (1080 px wide, or 1530 px with IS), with the title and setup chips above and "Step N of M · title" below. There is no caption text and no audio.

### Video (MP4)

- Silent H.264 MP4 at 25 fps, encoded by the browser's WebCodecs `VideoEncoder` (High profile, falling back to Main or Baseline) and packaged with [mp4-muxer 5.1.3](https://cdn.jsdelivr.net/npm/mp4-muxer@5.1.3/) from jsdelivr. A full keyframe every 5 s; the last frame is held for 3 s.
- Needs Chrome or Edge.
- Measured in headless Chrome at 1×: 3.2.0→4.6.0 AIO is 79 s, 4.5 MB, about 23 s to make; 4.1.0→4.6.0 distributed + IS + K8s is 126 s, 14.7 MB, about 34 s.
- Google Slides only inserts video from Google Drive or YouTube: upload the MP4 to Drive first.

### GIF

- Frames are rendered on a 10 fps clock (no frame limit; long animations can exceed 1000 frames, which Google Slides rejects. Use the video for Slides).
- One shared 255-colour palette (sampled from each step plus the theme colours), and pixels that did not change are written as transparent, which keeps files small. The last frame holds for 3 s and the GIF loops.
- The encoder ([gifenc 1.0.3](https://cdn.jsdelivr.net/npm/gifenc@1.0.3/)) is loaded from jsdelivr on first use, so it needs an internet connection.
- Measured in headless Chrome: 4.2.0→4.7.0 AIO at 2× is 258 frames, 1.2 MB, about 5 s to make; 4.1.0→4.6.0 distributed + IS + K8s at 1× is 1227 frames (2 min), 6.7 MB, about 15 s.

## How it is built

- One SVG scene built in JavaScript (`el()` helpers). Each component (APIM node, DB cylinder, region, pill, badge) is a small builder function.
- A GSAP timeline drives the step animations. Each step is a timeline label, so the timeline can seek, step back and forth, and change speed.
- Traffic packets run in a separate `requestAnimationFrame` loop (`frame()` → `tick(dt)`). The load balancer reads the timeline position to pick the target node, so seeking always gives the right traffic flow.
- Editing tips:
  - Don't GSAP-scale SVG groups. It leaves matrix offsets that detach boxes from their connector lines. Use opacity plus a small y slide instead.
  - Captions and narration explain the migration process only. Version or architecture details that are there for accuracy go in the diagram.

## Options for later

| Need | Tool |
|---|---|
| Interactive, in a browser, step by step (this tool) | SVG + GSAP |
| Narrated video (GIF and silent MP4 are built in) | In-browser neural TTS (e.g. Kokoro) or a cloud TTS service, muxed as an audio track |
| Designer-made reusable animated assets | Rive or Lottie |
| Many more scenarios | Turn the builders into a scene description (JSON/YAML) and generate the steps |
