# APIM migration visualization

Side project: simple 2D animations that explain WSO2 API Manager migrations to customers.

## What it shows

A setup screen asks for the source version, the target version, the deployment type (all-in-one or distributed), the target platform (VMs or Kubernetes), and whether downtime is acceptable. The page then picks the migration approach and builds the matching animation:

| Source | Target | Approach | Steps after the DB copy |
|---|---|---|---|
| any source up to 4.1.0 | 4.2.0 | Direct migration with migration clients | Same as the row below, with `-DmigratedVersion=4.2.0` added to the API-M client |
| 3.2.0, 4.0.0, 4.1.0 | 4.5.0, 4.6.0, 4.7.0 | Direct migration with migration clients | SQL script on SHARED_DB → identity client (`-Dmigrate -Dcomponent=identity`) → API-M client (`-Dmigrate -DmigrateFromVersion=<src>`), both clients changing both DBs |
| 4.2.0 or later | any later version from 4.3.0 | Zero-data migration | SQL script on APIM_DB only |
| 2.6.0, 2.7.0, 3.0.0, 3.1.0 | 4.3.0 – 4.7.0 | Two-step migration via 4.2.0 | Temporary APIM 4.2.0 all-in-one node (a VM for Kubernetes targets): SQL script on SHARED_DB → identity client → API-M client (`-DmigrateFromVersion=<src> -DmigratedVersion=4.2.0`) → node removed (not tested) → 4.2.0 → target SQL script on APIM_DB |

Other pairs have no documented direct path, so the setup screen only offers valid targets. These rules match the migration-docs catalog (`direct-migration-catalog/` and `upgrading-to-apim-4x0/`), checked 2026-10-16.

**Distributed deployments** are shown as one generalized pattern (the page says so on screen):
- **Gateway:** serves API calls (LB → Gateway → backend). It connects to the databases only for versions below 4.5.0.
- **Control Plane:** always connected to SHARED_DB and APIM_DB. Data changes come from here, and the migration clients run here because it is connected to all the databases.
- **Traffic Manager:** connected to the databases.
- **Start order after migration:** Control Plane, then Traffic Manager (connects to the DBs), then Gateway.

- Node-to-node links (Gateway–CP, CP–TM, TM–Gateway) are drawn as dotted lines.

**Kubernetes targets** (whatever the source platform): migration clients can't run inside the Kubernetes deployment. When the approach uses clients, the animation adds two steps:
- **Create a temporary VM:** an all-in-one APIM target-version node connected to the copied DBs. All-in-one is enough even when the target is distributed.
- **Remove the temporary VM:** after both clients finish, the VM is removed and only then does the Kubernetes deployment connect to the DBs and start.

Script-only (zero-data) migrations need no temporary VM.

The entry point is labelled **Load balancer / DNS**, since it can be either.

The downtime answer only sets where the **With downtime / Without downtime (freeze window)** switch starts. You can flip the switch at any time while the animation plays.

Common steps: production running → build target → **test target use cases with fresh DBs** → stop (downtime) or freeze → copy DBs → *approach steps* → start and test target → switch traffic.

The fresh-DB test applies to every version pair. The target gets the migrated configs and customizations, and starts against new empty databases. It goes through basic API Manager testing (portals, create/publish an API, subscribe, generate a token) and the customer's use cases are validated; the two-line badge reads "✓ Basic tests & / use cases passed". No traffic is sent to the backend during this test. When the tests pass, it is stopped and the fresh DBs are discarded. This happens before the maintenance window or freeze, so it adds no downtime.

API calls flow client → load balancer → gateway → **backend**. Purple dots are only data changes (new APIs, apps, subscriptions) or migration writes. A status panel shows API availability, whether changes are allowed, and which version serves traffic.

Open `index.html` in a browser. It needs internet access to load GSAP from cdnjs. URL parameters:
- `?src=3.2.0&tgt=4.7.0&dep=aio|dist&plat=vm|k8s&downtime=yes|no` skips the setup screen. The page updates the URL after setup, so you can share the link.
- `&step=N`: open paused at the end of step N
- `&at=identityRun`: open paused just after a timeline label (useful for screenshots)

The JAR full file names appear as hover tooltips on the JAR icons.

**WSO2 IS as Key Manager** (setup option for sources up to 4.2.0, plus which DBs are shared: SHARED_DB + APIM_DB, or SHARED_DB only). Source IS versions: 2.6.0 → IS-KM 5.7.x, 2.7.0 → IS-KM 5.8.x, 3.0.0 → IS-KM 5.9.0, 3.1.0 → IS-KM 5.10.0, 3.2.0 → IS 5.10.0, 4.0.0/4.1.0 → IS 5.11.0, 4.2.0 → IS 6.x.
- **Target APIM 4.2.0 → IS 6.1.0 (merged):** the DBs stay shared. First the IS migration client runs on IS 6.1.0 against the shared DBs. Then only the API Manager component migration runs (API-M client); the SHARED_DB script and the APIM identity client are skipped.
- **Target APIM 4.3.0+ → IS 7.x (split):**
  1. The DBs are split per product, so shared ones are duplicated and you end with 4 DBs.
  2. IS migrates on its own (IS migration client on IS 7.x).
  3. APIM migrates with its usual flow for the version pair.
  4. IS 7.x is connected as a third-party KM.
  5. Both are stopped and the **Third-party to resident conversion** SQL script runs on APIM_DB.
  6. Both start, and IS 7.x works like the resident KM.
- Two-step sources are unchanged on the APIM side; IS goes straight to 7.x and is attached to the final target.
- On Kubernetes, the IS migration runs on a temporary IS VM (below the target row), removed afterwards.
- The fresh-DB test covers APIM and IS together.

URL parameters for this: `&km=resident|is&kmdb=both|shared`.

### Accuracy notes to check before showing customers
- IS as KM: the IS migration is shown with the same arguments as the APIM identity step (`-Dmigrate -Dcomponent=identity`), and the conversion script name `km-conversion.sql` is a placeholder. Check both against the IS migration and KM conversion docs.
- 2.7.0: the migration-docs repo has no `upgrading-from-270-to-420` guide (this affects both 2.7.0 → 4.2.0 and the two-step path) (only 2.6.0, 3.0.0 and 3.1.0). 2.7.0 is included on the engineer's word and uses the same steps.
- 3.2.0 / 4.0.0 / 4.1.0 → 4.3.0 / 4.4.0 is not offered: the docs have no direct path. Decide whether these should also go via 4.2.0.
- Distributed on VMs: the clients are shown running on the Control Plane. The engineer noted an all-in-one node can also be used for this; only the Kubernetes path shows the temporary all-in-one VM.
- Distributed: the Gateway-to-DB rule (below 4.5.0 yes, 4.5.0+ no) comes from the engineer and has not been checked against the deployment docs per version. The TM link is drawn to SHARED_DB only, to stand for "connected to the databases".
- Freeze window: in an all-in-one node the key manager runs on the same node, so tokens issued during the freeze are written to the source DB and are not in the copy. Decide how to explain token re-issue, or show it.
- Client approach: the optional pre-migration validation run (`-DrunPreMigration`) and re-indexing are not shown.

## Customer deliverable

In the player, **⬇ Export for customer** downloads one standalone HTML file (about 165 KB), named for example `apim-migration-3.2.0-to-4.7.0-acme.html`:
- The current setup is locked in: versions, deployment, platform and Key Manager. There's no setup screen or export button, and URL parameters are ignored.
- The customer can switch **With downtime / Without downtime (freeze window)**, change the speed, play, pause and step through. The switch starts at the value chosen at export.
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

## How it is built

- One SVG scene built in JavaScript (`el()` helpers). Each component (APIM node, DB cylinder, region, pill) is a small builder function.
- A GSAP timeline drives the step animations. Each step is a timeline label, so the timeline can seek, step back and forth, and change speed.
- Traffic packets run in a separate `requestAnimationFrame` loop. The load balancer reads the timeline position to pick the target node, so seeking always gives the right traffic flow.

## Options for later

| Need | Tool |
|---|---|
| Interactive, in a browser, step by step (this POC) | SVG + GSAP |
| Export MP4/GIF clips for emails or slides | Motion Canvas or Remotion (code-driven, both render to video) |
| Designer-made reusable animated assets | Rive or Lottie |
| Many scenarios (distributed setups, other versions) | Turn the builders into a scene description (JSON/YAML) and generate the steps |
