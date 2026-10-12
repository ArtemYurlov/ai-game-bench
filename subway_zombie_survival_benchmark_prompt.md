Build LAST TRAIN: a complete, polished, first-person 3D subway zombie survival shooter, written in plain JavaScript using only Three.js. This is a one-shot build. The game must load and play correctly the first time it is opened, with no manual fixes, and it should look and feel like a finished, high-quality browser game.

Project root: `/tmp/3js_zombie`
This is the only absolute path in this brief. Every other path is relative to the project root, and no file you write may contain the absolute path.

YOUR WORKING CONDITIONS
- You have only the Read, Write, and Edit tools. You cannot run commands, execute the build script, open a browser, take screenshots, or hear audio. No one will fix your code before it is played.
- The project root starts empty. You cannot list directories, so the manifest in `build.sh` is the authoritative list of source files.
- Because you are working blind, correctness has to come from design: explicit interfaces, centralized data, automatic self-checks, defensive runtime guards, and a disciplined review of every file you write.

DELIVERABLES
- Many focused JavaScript files under `src/`.
- Exactly one build script, `build.sh`, at the project root.
- `bash build.sh` produces `dist/index.html`, a single self-contained HTML file. Opening it directly from disk (`file://`) in current desktop Chrome, Edge, or Firefox starts the game.
- Do not create package.json, node_modules, asset files, or any other build tooling.

THREE.JS AND DEPENDENCIES
- Use Three.js r160 core only. Load it at runtime with a dynamic `import()` of `https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js`. Define this URL once, as a variable at the top of `build.sh`, and inject it into the generated HTML.
- Do not use anything from `examples/jsm` (the addons). That includes PointerLockControls, GLTFLoader, EffectComposer, BufferGeometryUtils, RoomEnvironment, Sky, and RectAreaLightUniformsLib. Write the small equivalents you need.
- Use no other libraries, fonts, images, models, audio files, or network requests.
  - Build every model from Three.js geometry.
  - Generate every texture procedurally with canvas 2D or `DataTexture`.
  - Synthesize every sound with the Web Audio API.
  - Create every UI element with DOM and CSS from JavaScript.
- Write for the r160 API exactly:
  - `BufferGeometry` only; the legacy `Geometry` class no longer exists.
  - `renderer.outputColorSpace = THREE.SRGBColorSpace`, and `texture.colorSpace = THREE.SRGBColorSpace` on color textures.
  - `renderer.setAnimationLoop` for the frame loop.
  - Physically based light units: point and spot lights use `decay = 2` and need much higher intensities than older examples.
- Never guess at an API you are unsure of. Use a simpler one you know exists.

CODE ARCHITECTURE
- Source files are classic scripts, not ES modules. Never use `import`, `export`, or top-level `await` in `src/`. Dynamic `import(` (written with no space before the parenthesis) is used only by the boot loader.
- Use a single global namespace, `window.LT`.
  - `src/00_boot.js` defines `LT.define(name, deps, factory)`.
  - Every other file makes exactly one `LT.define(...)` call. Its factory receives `THREE` and an object of resolved dependencies, and returns the module's public API.
  - `src/99_main.js` is the last file and also calls `LT.start()`. That call loads Three.js, runs the factories in dependency order, runs the self-tests, and opens the main menu. It reports missing or circular dependencies by name.
- Start every factory body with `'use strict'`. Keep each file focused and under roughly 900 lines. Split large systems; for example, weapon data, weapon models, weapon logic, weapon effects, and viewmodel animation can each be a separate file.
- Put all content and tuning in data modules referenced by string IDs: weapons, attachments, enemies, stations, waves, economy, and upgrades. Systems never hard-code content.
- Define event names in one constants table and use the constants everywhere. A typo then becomes a detectable error rather than a silent no-op.
- Define conventions in one module and follow them everywhere: meters, seconds, radians, Y up, player eye height 1.65 m, and tracks running along the X axis unless a station specifies otherwise.
- Run gameplay on a fixed 60 Hz timestep with an accumulator. Clamp large frame gaps after tab switches. Render once per frame.
- Before calling another module, read that file and use its exact function names and signatures.

BUILD SCRIPT
`build.sh` must do all of the following.
- Be pure bash and run on bash 3.2 (the macOS default).
  - Do not use `mapfile`, associative arrays, or `${var,,}`.
  - With `set -u`, bash 3.2 treats an empty array as unbound. Guard for this.
  - Use only bash builtins plus `cat`, `mkdir`, `mv`, and `rm`. Do not use sed, awk, grep, python, curl, npm, or the network.
- Work from any current directory by resolving paths relative to its own location.
- Keep an explicit, ordered `FILES=(...)` manifest, with `src/00_boot.js` first and `src/99_main.js` last.
- Check the sources before writing any output:
  - Fail with a clear message if a manifest file is missing.
  - Warn about any `src/*.js` file that is not in the manifest.
  - Fail with `file:line` if any line contains `</script` in any letter case.
  - Fail with `file:line` if any line begins, after optional indentation, with `import ` or `export `.
- Run an optional syntax check. If `node` is on PATH and `--no-check` was not passed, run `node --check` on each source file separately, and fail with node's message on error. If node is missing, print that the check was skipped and continue. The build must never require node.
- Generate the HTML shell itself: doctype, charset, viewport, title, minimal reset CSS, and an inline script that sets `window.LT_BUILD = { threeUrl, files: [...] }`.
- Emit each source file in its own `<script data-src="src/...">` tag, ending with a `//# sourceURL=src/...` line. A syntax error in one file must not prevent the others from loading, and the diagnostics must be able to name the broken file.
- Write to a temporary file and move it into place only when everything succeeds, so a failed build never leaves a broken `dist/index.html`.
- Print a short summary: the number of files, which checks ran or were skipped, and the output path.

ERROR REPORTING AND SELF-CHECKS
The user will play the game and paste console output back to you if something breaks. That output must be enough to find and fix the bug without seeing the screen. Everything shown on screen is also printed to the console.

Logging and error capture
- `src/00_boot.js` installs `window.onerror` and `unhandledrejection` handlers before any other code runs.
- It also provides a tagged logger (for example `[LT:weapons] WARN ...`) that keeps a ring buffer of recent lines.
- `LT.define` records `document.currentScript.dataset.src`. At startup, compare the recorded files against `LT_BUILD.files`. Any file that produced no definition almost certainly failed to parse, so name it.

Fatal errors
- Fatal errors are: Three.js failed to load (explain that the library loads from a CDN and needs a network connection), WebGL2 is unavailable, or a core module (renderer, input, player, level, or game loop) failed to define or initialize.
- A fatal error shows a full-screen overlay with a plain-language explanation, the technical error and stack, and a "Copy diagnostics" button. If the Clipboard API is unavailable, fall back to a pre-selected textarea.

Non-fatal errors
- Non-fatal errors never freeze the game. Show a small corner badge with an error count. Log full details the first time, then count repeats instead of flooding the console.
- Wrap module initialization and each per-frame system update in try/catch. Log the context: system name, game state, station, wave, and frame number.
- If a system keeps throwing, disable it and log that you did.
- If audio, post-processing, or a decorative builder fails, continue without it.

Self-tests
Self-tests run automatically and print a pass/fail summary.
- At boot, validate every data table:
  - All ID references resolve: station waves to enemy types, weapons to attachment slots and options, and unlock requirements to real items.
  - Every number is finite and within a sane range.
  - Every weapon and enemy type has a model builder and a behavior.
- On each station load, check that:
  - The player spawn, every enemy spawn, and every interactable stand on walkable ground and are not inside colliders.
  - The navigation graph is connected, and every spawn can reach the platform.
  - The train stop lines the doors up with the boarding zones.
  - The player can get back from the track bed to the platform.
  - The secret area is reachable.
- For balance, compare each station's total enemy health with the damage available from the expected loadout and ammo supply at 50 percent accuracy. Warn if the margin is thin.
- After the first rendered frame of each station:
  - Read back a small downsampled frame and warn if the average luminance is near black or blown out.
  - Warn if the camera is inside a collider.
  - Warn if draw calls, triangles, or light counts exceed budget.
- Continuously:
  - Check player, camera, and enemy transforms for NaN or Infinity, and report the first system that produced one.
  - Detect enemies that have been stuck or unable to reach the player for more than a few seconds. Log their position and state, then relocate or remove them.
  - Detect a wave that can no longer complete, log why, and recover.

Diagnostics and debug mode
- `LT.diagnostics()`, the F8 key, and the overlay button all produce one copyable block. It is delimited by `=== LAST TRAIN DIAGNOSTICS BEGIN ===` and `=== LAST TRAIN DIAGNOSTICS END ===` and contains browser and WebGL info, settings, game state, self-test results, and recent warnings and errors.
- Debug mode is enabled with `?debug=1` and is off by default.
  - It shows FPS, draw calls, triangles, colliders, and the navigation graph.
  - It enables cheats: god mode, add tokens, skip wave, and unlock everything.
  - It allows starting at a given station with `?station=N`.

THE GAME
You are a night-shift transit worker riding the last running train toward Terminus, where the surface evacuation point waits. The driverless train can only run one section of the line at a time. At each station it drops you off, pulls back into the tunnel, and returns only when the station's signal turns green, which happens once the station is clear.

Design pillars:
- Gunplay that feels weighty, precise, and spectacular.
- Tense, readable horde combat where positioning and target priority matter.
- Constant forward momentum. Every station is a new place, and the train is both the reward and the most dramatic moment of each level.
- Meaningful build choices through weapon unlocks, attachments, and upgrades.

CORE LOOP
Each station follows the same five phases:
1. Arrival: the train brakes into the station and the doors open. The player gets a short calm period to scout, grab supplies, and use the vending machines. The train pulls away and leaves them stranded.
2. Waves: two to four waves per station, separated by short breathers. A visible countdown shows the breather's remaining time, and the player can skip it.
3. Signal: when the last wave falls, the signal turns green and the train is called. A 45 to 60 second hold-out follows, with spawns ramping up as the train approaches.
4. Boarding: the doors stay open for about 15 seconds, then close with the player inside and the infected outside. Missing the train triggers an extra, harder wave before it returns.
5. Ride: the moving car is the safe room. The player reviews the station summary, buys and upgrades weapons, and fits attachments, then rides into the next station.

Death restarts the current station from its arrival and restores the player's state from that moment: tokens, loadout, unlocks, and mastery.

The run ends with a boss fight at Terminus and a final ride to the surface.

The opening must be fast. After New Game, a skippable intro of about 15 seconds plays inside the train as it brakes into the first station. Within the first minute, the player should be on the platform shooting the first infected.

PLAYER AND CONTROLS
Controls
- Mouse look uses pointer lock.
- Movement and actions:
  - WASD to move, Shift to sprint, C to crouch (sprint plus crouch slides), and Space to jump.
  - R to reload, E to interact, F to toggle the headlamp, V for melee, and G to throw.
  - Q swaps to the last weapon, 1 to 3 select a weapon, and the mouse wheel cycles weapons.
  - Left mouse fires and right mouse aims down sights.
  - B changes fire mode, T inspects the weapon, Tab shows objectives and stats, and Esc pauses.
- Avoid Ctrl bindings, because Ctrl+W closes the tab.

Movement
- Movement should feel smooth and responsive:
  - acceleration and friction, and sprint stamina
  - step-up for stairs, ramps, and moving escalators
  - a landing dip, and head bob that can be turned off
  - footsteps that change with the surface (tile, concrete, metal grating, water)
- The player can mantle any ledge up to 1.3 m, including the platform edge from the track bed.

Health and feedback
- Health is split into four segments of 25. After a few seconds without damage, it regenerates only up to the top of the current segment.
- Armor plates absorb damage and are bought or found.
- Boarding the train fully heals the player.
- Damage feedback includes directional indicators, a red edge vignette scaled to the damage, view kick, and muffled audio with a heartbeat at low health.
- The player wears a dim headlamp from the start. The tactical light attachment is much stronger.

WEAPONS AND GUNPLAY
The weapon in the player's hands is on screen almost every frame. It deserves the most detail and polish in the game.

Viewmodel
- Build each weapon as a detailed hierarchical model with real parts:
  - Receiver, barrel, handguard, sights, trigger and guard, grip, stock, magazine, and ejection port.
  - Moving parts such as the slide, bolt, charging handle, pump, or lever.
  - Visible cartridges: the top rounds in a magazine, shells during the shotgun reload, and the LMG belt.
- Use several materials per weapon (blued steel, worn aluminum, polymer, wood, rubber), with procedural wear, scratches, and edge highlights.
- Show gloved hands and sleeves holding the weapon in plausible grip poses.
- Render the viewmodel in a second pass after clearing depth, using its own camera at a lower FOV, so it never clips into walls.
- Light the viewmodel to match its surroundings with four lights: a hemisphere light tinted to the current area, a key light that follows the nearest fixture, the muzzle-flash light, and the headlamp.
- Drive the viewmodel's motion with springs:
  - idle breathing sway, mouse-look lag, movement bob, and strafe tilt
  - the sprint pose, jump and landing response, and slide pose
  - lowering the weapon near walls, and smooth weapon switching (lower and raise)
  - a first-draw animation (racking the slide or bolt) and an inspect animation that turns the weapon to show off its attachments
- Aiming down sights must align the actual sight or optic exactly with the screen center. Compute the offset from the sight geometry; never hand-tune it per weapon.

Firing
- Every shot is a sequence:
  - The trigger is pulled.
  - The muzzle flashes: several crossed additive planes with randomized shape and rotation, plus a brief pooled light that illuminates nearby walls.
  - The slide or bolt cycles.
  - A casing ejects with physics: it spins, bounces, rattles on the floor, and lingers.
  - Recoil follows a per-weapon pattern, mostly vertical climb with random horizontal variance and smooth recovery.
  - The camera kicks, with an FOV punch on heavy weapons.
  - Layered sound plays.
- Sustained fire produces muzzle smoke. Automatic weapons also show heat shimmer over the barrel.
- An empty pistol's slide locks back. Dry-firing clicks. A low-ammo sound and HUD cue play near the end of a magazine.
- Tracers are visible streaks. Automatic weapons show a tracer every few rounds. Projectile weapons fire visible projectiles.

Ballistics and hits
- Hitscan rays test against analytic hit volumes and against the level's collision boxes. Each enemy has spheres and capsules for its head, torso, and limbs. Do not raycast against render meshes.
- Spread grows with sustained fire and movement, and shrinks while aiming or crouching.
- Damage falls off with distance. Shotguns fire pellets. Rifles and armor-piercing rounds penetrate several enemies.
- Impacts respond to the surface they hit:
  - chips and dust on tile, and grey puffs on concrete
  - sparks and ricochet whines on metal
  - splashes in water, and blood sprays on flesh
- Impacts leave pooled decals: bullet holes, blood spatter on walls and floors behind the target, and scorch marks.
- Hit feedback:
  - a hit marker, with a distinct headshot marker and sound
  - a kill confirmation
  - a crosshair that reflects the real spread
  - optional damage numbers
  - screen shake scaled by weapon

Reloading
- Reloads are staged animations tied to sound: magazine out, magazine in, chamber a round.
- A tactical reload, with a round still in the chamber, is faster than an empty reload. An empty reload also releases the slide or charges the bolt.
- Dropped magazines fall with physics and stay on the ground briefly.
- The shotgun loads shell by shell, and firing interrupts the reload.
- The LMG opens its feed cover and replaces a belt box, and the belt visibly shortens while firing.
- Sprinting cancels a reload. Fire and switch inputs are buffered so the controls never feel unresponsive.

Weapon roster
New weapons unlock roughly one per station. The player carries two primary weapons and the pistol, plus melee and throwables; the other weapons wait on the train's weapon rack.
1. Service Pistol: the starter sidearm. Semi-automatic, with unlimited reserve ammo, so the player is never left unable to fight.
2. Pump Shotgun: eight pellets, a visible pump on every shot, shell-by-shell reload, and strong knockback.
3. Compact SMG: high fire rate, accurate from the hip, quick handling.
4. Carbine: a versatile rifle with automatic, burst, and semi-automatic modes.
5. Marksman Rifle: heavy semi-automatic rounds that penetrate several targets.
6. Rail Spiker: an improvised pneumatic launcher built from track tools. Its slow, visible spikes pierce enemies and pin their bodies to walls.
7. LMG: a 100-round belt and a slow reload. It is steadier while crouched.
8. Arc Thrower: built from a third-rail collector shoe. Hold fire to charge, then release a jagged procedural lightning bolt that chains between nearby enemies and stuns them.

Melee, throwables, and ammo
- Melee: a heavy track wrench, with a quick strike on V, knockback, and a stamina cost. It is the best tool against crawlers.
- Throwables: a frag grenade that can be cooked, and a red track flare that lures nearby infected and lights the area.
- Ammo: the pistol never runs out. Other weapons share ammo pools by class, refilled by crates, enemy drops, and vending machines.

ATTACHMENTS AND GUNSMITH
Slots
- Each weapon supports some of these slots: optic, muzzle, underbarrel, magazine, stock, and ammo type. Each weapon defines which slots and options it supports.

Attachments
Every attachment visibly changes the weapon model and its stats.
- Optics: iron sights, reflex red dot, holographic sight, 2.5x combat scope, and 6x marksman scope.
  - Lenses have tinted glass and an illuminated reticle that stays centered while aiming.
  - Magnified scopes zoom the camera and show a scope overlay with a reticle.
- Muzzle: suppressor (quieter, smaller flash, different sound), compensator (less vertical recoil), flash hider, and a shotgun choke (tighter spread).
- Underbarrel: vertical grip, angled grip, tactical light (a strong shadow-casting spotlight), and laser. The laser shows a visible beam in dust and fog, puts a dot on surfaces, and tightens hip-fire spread.
- Magazine: extended, drum (SMG and carbine only), and quick-pull (faster reload).
- Stock: light (faster aiming and movement) or heavy (less recoil).
- Ammo: hollow point (more damage to flesh, worse against armor), armor-piercing (penetrates and cracks armor), and incendiary (burning damage, ember tracers, and flaming impacts).

Progression
- Attachments unlock through weapon mastery. Kills with a weapon raise its level, and each level unlocks options that can then be bought with tokens.
- Weapons also have three upgrade tiers that improve damage and handling.

The gunsmith
- The gunsmith is a workbench in the train car. Interacting with it moves the camera to the bench, where the weapon sits on a slowly rotating mount under good light.
- Slot hotspots appear on the model. Hovering over an option previews it on the model immediately.
- A stats panel shows bars for damage, fire rate, range, accuracy, recoil, handling, mobility, and magazine size, with green and red deltas.
- Buying, equipping, and leaving must all work, and loadouts persist.

ENEMIES
Models and animation
- Build the infected procedurally from primitives in a joint hierarchy.
- Animate them procedurally: lurching gaits with per-individual timing, reaching arms, head tilts, stumbles, hit flinches, and staggers.
- Vary their appearance: commuters, transit staff, construction workers, and riot police, with different clothes, body proportions, skin decay, and missing parts.

Fairness
- Every type has a distinct silhouette, sound, and attack telegraph.
- Damage lands only during an attack's active frames, and only within real reach.
- Never deal damage through walls, or from an attacker the player had no chance to see or hear coming.

Damage model
- Head, torso, and limb hit volumes have separate damage multipliers.
- Heavy hits stagger, and shotguns knock enemies back.
- Destroyed legs turn an enemy into a crawler.
- Enough damage blows off heads and arms in bursts of blood.
- Armor pieces crack and fall off visibly.

Death and drops
- Deaths are procedural collapses in the direction of the killing hit.
- Corpses are capped in number and fade out over time.
- Enemies drop tokens, collected automatically within a short radius, and sometimes ammo or armor.

Enemy types, in the order they are introduced
- Shambler: slow and durable. Attacks in groups.
- Sprinter: fast and fragile, with a telegraphed lunge.
- Crawler: low to the ground. It emerges from drains and vents and is hard to hit with rifles.
- Bloater: slow and swollen. It explodes on death, which can set off other bloaters. Kill it at range.
- Spitter: after a gurgling windup, it lobs acid in visible arcs that leave damaging puddles.
- Hardhat: an armored construction worker. Its helmet and plates must be broken first.
- Screamer: its scream blurs the player's vision and summons a burst of sprinters. It is a priority target.
- Brute: a huge mini-boss with a long-telegraphed charge and a ground slam that knocks the player back.
- Lurker: clings to the ceiling in dark stations and drops onto the player. Falling debris and a hiss give it away.

Navigation and spawning
- Each station defines a waypoint graph that includes stairs, escalators, and climb points along the platform edge.
- Enemies path with A* on the graph and steer with separation. They pursue directly when they have line of sight on the same level.
- Spread path recalculation across frames.
- Infected enter the arena with readable animations: climbing up from the track bed onto the platform, pouring down stairwells, crawling out of vents, breaking through maintenance doors, and dropping from the ceiling.
- When possible, avoid spawning enemies in the player's direct view or within about 12 m of the player.
- Cap living enemies at about 35 on High, and queue the rest.

STATIONS
Every station must be a distinct, fully dressed place, with its own layout, palette, lighting, sound, and gimmick. Stations grow larger and more vertical as the run progresses.

Scale reference:
- Platforms stand about 1.1 m above the rails.
- Side platforms are 4 to 6 m wide; island platforms are 9 to 12 m wide.
- Platforms are longer than the 4-car train, at least 90 m.
- Train doors line up with marked boarding zones.

The seven stations:
1. Mercer Street (the opening): a classic cut-and-cover station.
   - Two side platforms face two tracks under a low steel-beam ceiling.
   - Cream tile with a dark green trim band and mosaic name tablets.
   - Stairs lead up to a small ticket hall with turnstiles and a token booth with a cracked window.
   - A passage under the tracks connects the two platforms.
   - The waves introduce shamblers, then sprinters, and tutorial prompts appear in context.
2. Canal Market: a wide island platform with a center row of columns.
   - A half-level market mezzanine sits at one end, with abandoned stalls, string lights, and crates.
   - Exposed brick weeps water.
   - Once per wave, an express train thunders through on the far track and kills anything on that track, including the player. Flashing platform-edge lights and its horn announce it several seconds ahead.
   - Crawlers debut here.
3. Old Exchange Hall: a grand 1900s hall with a 10 m barrel vault.
   - Cast-iron columns, backlit stained-glass light wells, and brass chandeliers that can be shot down onto the infected.
   - A long balcony is reached by two grand staircases. This is the first truly vertical fight.
   - Bloaters and spitters debut here.
4. Deepcut Works: an unfinished extension of raw concrete, rebar, plastic sheeting, and scaffold towers.
   - Catwalks run at 3 m and 6 m.
   - A giant tunnel boring machine cutterhead sits at the end of an unfinished tunnel.
   - Mid-station, the power fails. Only emergency strobes and the player's own lights remain until the player pulls two breaker levers, one of them high on the catwalks. The lights then return bank by bank.
   - Hardhats debut here.
5. Crossway Interchange: a modern interchange with two stacked levels.
   - The upper platforms run along X and the lower platforms run along Z.
   - Working escalators and a central atrium with glass railings connect the levels, so the player can fire down into the lower level.
   - White panels, LED strips, and glitching digital signs.
   - Screamers debut here, and the first Brute arrives in the final wave.
6. Blackwater Deep: a deep-level station reached by long inclined escalator shafts.
   - Circular tube tunnels.
   - The track bed and lower concourse are partly flooded; the water slows movement and splashes.
   - Red emergency lighting, sparking cables, and mist.
   - A breaker can electrify the water, killing everything standing in it, including the player.
   - Lurkers debut here.
7. Terminus: the largest station, with three levels.
   - Platforms with three tracks.
   - A mezzanine concourse with shops.
   - An upper ticket hall with an animated split-flap departure board, under a skylight shaft that lets in moonlight.
   - Two waves are followed by the final boss.

Final boss, The Motorman
A colossal infected fused with the front of a wrecked maintenance railcar, with a headlamp grown into its shoulder.
- Phase 1: it charges down the track bed, telegraphed by its headlamp and horn. The player dodges onto the platforms and shoots the glowing growths on its back as it overshoots.
- Phase 2: it abandons the railcar and climbs onto the platform. It slams the ground with a telegraphed shockwave ring and hurls debris.
- Phase 3: it becomes enraged and faster, summons sprinters with a roar, and exposes a weak point on its chest.
- It has a boss health bar and a dramatic death sequence.
- After the boss dies, the last train arrives. The final ride climbs out of a tunnel portal onto a dawn-lit surface with a distant city skyline, followed by credits and run stats.

Every station also includes:
- Dense, purposeful detail:
  - benches, bins, vending machines, ticket machines, and payphones
  - CCTV cameras, fire extinguishers, and emergency call points
  - maps, procedural ad posters, and illuminated exit signs
  - cable trays, pipes, and fluorescent fixtures, some dead or flickering
  - debris, abandoned luggage, newspapers, overturned barriers, blood trails, and graffiti
- A detailed track bed:
  - ballast, sleepers, and rails with polished tops
  - a covered third rail that damages anyone who touches it
  - signals at the tunnel mouths, and tunnel lights receding into darkness
- Interactables that animate and sound physical:
  - vending machines that light up and drop items into the tray
  - ammo crates whose lids open
  - breaker levers and maintenance doors
  - fire extinguishers that burst into a slowing cloud when shot
  The off-hand reaches out where that makes sense.
- One secret area, such as a breakable panel or a maintenance closet, holding a weapon case or a token stash.
- Cover and routes that support movement instead of blocking it. There are no dead ends where the player can be trapped without an escape route.

THE TRAIN
The train is the game's most important set piece. The player sees it arrive at every station, rides inside it between stations, and boards it under pressure. It must look and sound spectacular.

Exterior
The train has 4 cars, each about 18 m long and 3 m wide.
- Fluted stainless-steel side panels with a colored line stripe and car numbers.
- A front cab with:
  - a raked windshield and wipers
  - bright headlights and marker lights
  - an LED destination sign showing the line name and "TERMINUS"
  - a coupler and an anticlimber
- Below the body: bogies with wheels, axle boxes, springs, brake units, and third-rail collector shoes, plus undercarriage equipment boxes and cables.
- Roof air-conditioning units and vents, and gangways between the cars.
- Four doors per side per car, each with windows, rubber edges, and a status light above it.
- Damage: dents, bloody handprints, a cracked window, and a boarded-up cab window.

Interior
- Seating and handholds: scuffed vinyl bench seats, stainless poles and grab bars, and hanging straps that swing with the car's motion.
- Lighting and signage:
  - fluorescent ceiling panels, a few of them flickering
  - ad frames with procedural posters
  - a lit line-map strip above the doors, marking the current and next stations
  - a scrolling LED next-stop sign
- Ribbed rubber flooring, an emergency intercom, and a gangway door to the next car.
- The player's car is a converted armory car with:
  - the gunsmith workbench, with a vise and tools
  - a rack showing every owned weapon as its real model
  - ammo crates, a token terminal, sandbags, and a work lamp

Arrival, seen from the platform
1. The signal turns green.
2. A distant rumble and a rising wind begin, and the headlight glow grows on the tunnel walls.
3. The horn sounds, and the train bursts out of the tunnel in a gust of dust and paper.
4. The brakes squeal and the wheels throw sparks as the train decelerates along a smooth, physically plausible curve. It stops exactly at the marked position.
5. The air brakes hiss, the door chime sounds, and the doors slide open with easing.
6. Interior light spills across the platform, and the boarding zones light up.

Departure
1. A chime and a "stand clear" message play.
2. The doors close. If an infected is in a doorway, that door stutters and reopens briefly, then shoves it out.
3. Air releases, and the train accelerates.
4. The platform slides away, and the red tail lights recede into the tunnel.

Ride
- The car sways and bounces subtly. Rhythmic wheel clacks at the rail joints follow the speed, and the lights dip briefly at gaps in the third rail.
- Tunnel lights and cable runs stream past the windows.
- Occasionally the train passes a dark abandoned platform, or a train going the other way with a blast of light and sound.
- The station summary, gunsmith, and upgrades are all available during the ride.
- From the third ride on, a breach event can occur. Infected climb onto the roof, smash a window, or come through the gangway from a dead car, and the player defends for about 30 seconds before arriving.
- Approaching the next station, station lights appear ahead in the tunnel, the train brakes, and the doors open onto the new platform.

Continuity
- While the player is aboard a moving train, simulate them in the car's local frame so they move with it smoothly.
- Swap between station scenes and the ride scene only while the train is in the dark tunnel. Preserve the player's position and view relative to the car, so there is no visible jump.
- During the ride, keep the car near the origin and move the tunnel around it, to avoid floating-point drift.

WAVES, PACING, AND PROGRESSION
Wave structure
- Waves use threat budgets. Each enemy type has a cost, and each wave has a budget and a roster.
- Spawns arrive in groups with brief lulls between them.
- Introduce each new enemy type alone or in a small group before mixing it in.

Wave signaling
- Announce each wave start with a station PA chime, flickering lights, and horde audio from the spawn directions.
- Show the number of enemies remaining.
- If a wave stalls with only a few enemies left, reveal them with outlines.

Difficulty
- The first station should be survivable by a newcomer, and the final station should challenge a skilled player on Normal.
- Offer Easy, Normal, and Hard settings.

Economy
- The currency is tokens, styled as old subway tokens.
- Players earn tokens from kills, with bonuses for headshots, multi-kills, and melee kills.
- Each station also pays a completion bonus based on a Bronze, Silver, or Gold rating for time, damage taken, and accuracy.
- Tokens buy weapons, attachments, upgrade tiers, armor plates, ammo, and player upgrades (maximum health segments, stamina, reload speed, and headlamp strength).
- Tune prices so an average player can afford roughly one significant purchase and two attachments per station.

Persistent progress
- Persist unlocked weapons, attachments, mastery, settings, the best station reached, and the current run's checkpoint.

VISUAL DIRECTION AND RENDERING
Mood
- Gritty, cinematic survival horror, lit like a film:
  - practical light sources
  - deep shadows that are never pure black
  - wet, reflective floors and dust in the light beams
  - haze fading into the tunnels
- Each station has its own palette, fog color, and exposure.

Procedural textures
- Generate the textures once and cache them:
  - subway tile with grout, grime gradients, and cracks
  - terrazzo, concrete, and brick
  - rusted and painted steel, ribbed rubber, and wood
  - tactile yellow paving at the platform edge, and ballast
  - mosaic station names, signage, posters, and graffiti
- Generate roughness maps, and convert height maps to normal maps in code, so surfaces read as physical.
- Give metals an environment map by rendering a small procedural room of emissive panels through `PMREMGenerator.fromScene`.

Lighting
- Emissive fixture meshes do most of the visual work.
- A fixed pool of real point lights is repositioned, with smooth fades, to the fixtures nearest the player.
- Never change the number of lights in the scene at runtime, because that forces shader recompiles and stutter. Keep the muzzle-flash light in the scene permanently, at zero intensity when idle.
- Shadows come from at most two shadow-casting spot lights: the headlamp or tactical light, plus one key light per area.
- Use additive light cones and dust motes for volume, ACES filmic tone mapping, and fog.

Post-processing
- Optionally, add custom post-processing using render targets and `ShaderMaterial`:
  - bloom, vignette, and film grain
  - per-station color grading
  - chromatic aberration when the player is hurt
- The game must look good with post-processing off, and it must fall back to that automatically if the post-processing shaders fail.

Avoid
- Flat untextured boxes, empty platforms, uniformly lit rooms, and pure black scenes.
- Z-fighting on coplanar surfaces.
- Repeated props that all face the same way.

AUDIO
- Synthesize everything with Web Audio.
- Weapons:
  - Each weapon has a distinct layered report: a transient crack, a body, a low thump, and a tail through a station-sized reverb made from a generated impulse response.
  - Mechanical reload sounds, casing tinkles that vary by surface, suppressed variants, dry fire, and impact sounds by surface.
- Infected: groans, shrieks, and gurgles built from oscillators, noise, and moving filters, with per-type signatures and spatialized panning. Attack windups must be audible.
- Train: a low rumble that rises with speed, wheel clacks, rail singing, the horn, brake squeal, air hiss, the door chime, and whooshes as other trains pass.
- Ambience for each station: fluorescent hum, drips, distant screams, and wind in the tunnels.
- Music:
  - adaptive layers that build with wave intensity
  - a boss theme
  - stingers for wave start, wave clear, and train arrival
- PA announcements appear as on-screen text with a chime. When `speechSynthesis` is available, they can also be spoken, with a setting to turn that off.
- Resume the AudioContext on the first click. Limit and pool voices. If audio fails, the game continues silently and logs the reason.

UI, MENUS, AND SAVING
Style
- Style the interface like transit signage: bold sans-serif type, line-colored bars, and round route bullets.
- Build it with DOM and CSS from JavaScript.
- Menus and the gunsmith release pointer lock and use the normal cursor. Clicking back into play re-locks the pointer.

Main menu
- The main menu sits over a live 3D background of the idle train interior.
- It offers Continue, New Game, Settings, Controls, and Credits.

HUD
- Segmented health and armor.
- Ammo in the magazine and in reserve, fire mode, a weapon icon, and throwables.
- Tokens.
- The station name and a route-bullet progress strip.
- The wave number, enemies remaining, the train countdown, and the current objective.
- Interaction prompts and the boss health bar.
- Hit markers and the damage-direction indicator.

Pause menu
- Resume, Settings, Controls, Restart Station, and Quit to Menu.
- Losing pointer lock pauses the game, and clicking resumes it.

Settings
- Mouse sensitivity, a separate aim-down-sights sensitivity, and invert Y.
- FOV from 70 to 100.
- Quality preset: Low, Medium, High, or Ultra.
- Master, music, and effects volume.
- Toggles for camera shake, head bob, damage numbers, and spoken announcements.

Other screens
- The station summary, shown during the ride.
- A game-over screen with Retry Station.
- An ending with credits and full run stats.

Saving
- Save to localStorage with a versioned schema.
- Validate saves on load. Corrupt or outdated data is reset with a logged warning and never crashes the game.
- Wrap storage access in try/catch for browsers that block it.
- Autosave on each station arrival and in the train car.

PERFORMANCE
Targets
- Target 60 fps at 1080p on a mid-range GPU - no further settings are required.

Geometry and budgets
- Use instancing for repeated props.
- Write your own merge utility for static geometry, grouped by material. Merged geometries must share the same attributes.
- Budget roughly 500 draw calls and 1.5 million triangles on High.
- Cap particles, decals, casings, and corpses.

Memory
- Pool everything spawned during combat.
- Avoid per-frame allocations in hot paths by reusing temporary vectors.
- Dispose of each station's geometry, materials, and textures on unload.

Avoiding stutter
- After loading a station, pre-warm shaders with `renderer.compile`.
- Pre-spawn pooled effects so the first shot, the first explosion, and the train's arrival never stutter.

Display
- Cap the device pixel ratio by quality preset.
- Handle window resizing, and pause when the tab is hidden.

HOW TO WORK
1. Plan before writing. Decide the full module list, the dependency graph, public APIs, data schemas, event names, and station coordinate layouts.
2. Write `build.sh` with the complete manifest early. Keep it in sync whenever you add, rename, or remove a file.
3. Write the files in this order:
   1. boot and diagnostics
   2. core: constants, events, math utilities, input, renderer, and the audio engine
   3. collision and navigation
   4. procedural textures and materials
   5. props
   6. the train
   7. stations
   8. the player
   9. weapons and the viewmodel
   10. enemies
   11. waves and progression
   12. UI
   13. the gunsmith
   14. main
4. Review every file right after writing it, then do a full cross-file review at the end. Check that:
   - every called function and property exists with the exact name and signature
   - every `LT.define` dependency exists, and there are no circular dependencies
   - every data ID resolves
   - every file is syntactically valid on its own, with balanced brackets and no stray tokens
   - no addon or removed Three.js APIs are used, no file uses `import` or `export`, and no source contains `</script`
   - no lights are added or removed at runtime
   - every pooled object is returned to its pool
   - every station's coordinates pass that station's self-tests when you check them by hand
5. Mentally play through a full run and fix every break you find. Walk through:
   - boot, the main menu, New Game, and the intro
   - the first shot, the first kill, and the first wave clear
   - the train's arrival, boarding, the ride, and a gunsmith purchase
   - the next station's arrival
   - a death and retry, and a save followed by Continue
   - each station's gimmick
   - the boss and the ending

Everything in this brief is required. If you must choose between an ambitious but fragile implementation and a robust, polished one, choose robust and polished, but try to avoid droping features where possible - and if you have to drop any try to drop the least important ones to first experience.
