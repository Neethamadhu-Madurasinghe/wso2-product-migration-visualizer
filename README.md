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

Common steps: production running → build target → **test target with fresh DBs** → stop (downtime) or freeze → copy DBs → *approach steps* → start and test target → switch traffic.

The fresh-DB test applies to every version pair. The target gets the migrated configs and customizations, and starts against new empty databases. No traffic is sent to the backend during this test. When the tests pass, it is stopped and the fresh DBs are discarded. This happens before the maintenance window or freeze, so it adds no downtime.

API calls flow client → load balancer → gateway → **backend**. Purple dots are only data changes (new APIs, apps, subscriptions) or migration writes. A status panel shows API availability, whether changes are allowed, and which version serves traffic.

Open `index.html` in a browser. It needs internet access to load GSAP from cdnjs. URL parameters:
- `?src=3.2.0&tgt=4.7.0&dep=aio|dist&plat=vm|k8s&downtime=yes|no` skips the setup screen. The page updates the URL after setup, so you can share the link.
- `&step=N`: open paused at the end of step N
- `&at=identityRun`: open paused just after a timeline label (useful for screenshots)

The JAR full file names appear as hover tooltips on the JAR icons.

### Accuracy notes to check before showing customers
- 2.7.0: the migration-docs repo has no `upgrading-from-270-to-420` guide (this affects both 2.7.0 → 4.2.0 and the two-step path) (only 2.6.0, 3.0.0 and 3.1.0). 2.7.0 is included on the engineer's word and uses the same steps.
- 3.2.0 / 4.0.0 / 4.1.0 → 4.3.0 / 4.4.0 is not offered: the docs have no direct path. Decide whether these should also go via 4.2.0.
- Distributed on VMs: the clients are shown running on the Control Plane. The engineer noted an all-in-one node can also be used for this; only the Kubernetes path shows the temporary all-in-one VM.
- Distributed: the Gateway-to-DB rule (below 4.5.0 yes, 4.5.0+ no) comes from the engineer and has not been checked against the deployment docs per version. The TM link is drawn to SHARED_DB only, to stand for "connected to the databases".
- Freeze window: in an all-in-one node the key manager runs on the same node, so tokens issued during the freeze are written to the source DB and are not in the copy. Decide how to explain token re-issue, or show it.
- Client approach: the optional pre-migration validation run (`-DrunPreMigration`) and re-indexing are not shown.

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
