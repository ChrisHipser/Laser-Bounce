# Liminal Laser Complex — design notes

Game file: `laser_bounce.html`. The original download is untouched.

---

## 1. Lighting in a browser: what's possible

**Yes, real ray tracing works in HTML.** WebGL2 fragment shaders and WebGPU compute shaders can trace rays. The question is how much you trace per frame. Options from cheapest to most "real":

| # | Technique | What you get | Cost / catch |
|---|-----------|--------------|--------------|
| A | **Forward raster + point lights** (the original) | Fast, simple | Light passes through walls; the original lit per vertex (Lambert), so it looked blotchy |
| B | **Per-pixel Phong + room light masks** (*this build*) | Crisp hotspots. A light only affects the room it's in, so no wall leaks | Light doesn't spill through an open doorway. No soft occlusion from props |
| C | **Shadow maps** (cube maps for point lights, one map for the spot) | True occlusion. Props cast laser shadows | Each point light costs 6 extra scene renders. The first build used 6 of them and took 46 ms a frame. This build keeps 1 shadowed laser light on HIGH, plus the flashlight |
| D | **CPU ray casting for gameplay optics** (*this build*) | Exact reflection/refraction of the beam and the flashlight caustics, with analytic spheres and boxes | Only the beam and caustic rays are traced, not the whole image |
| E | **Full-screen GPU ray tracer** (WebGL2 fragment shader) | Real shadows, reflections, refraction and soft light, every pixel | The scene must be analytic boxes, spheres and planes (yours nearly is). Deformable meshes and three.js materials go away. About 30–60 boxes per room traced per pixel runs well on a desktop GPU |
| F | **Hybrid: raster + screen-space** (SSAO, SSR, bloom) | Contact shadows, glossy floors, a laser glow halo | Screen-space only (no off-screen info). Needs EffectComposer passes |
| G | **WebGPU path tracer** (e.g. three-gpu-pathtracer) | Physically correct global illumination | Progressive: converges over many frames, so it isn't interactive with moving lasers. Good for a "photo mode" |
| H | **Baked lightmaps / light probes per room** | Free soft bounce light at runtime | Rooms are procedural, so you'd bake on generation (slow) or approximate |

**Recommendation:** keep B + C + D for gameplay. Add **E** as an optional "RT mode" later. Your world is boxes, spheres and prisms, which is exactly what an analytic shader ray tracer handles well. Each room's geometry would go into a uniform buffer as about 64 OBBs and spheres; deformation dents would be dropped in that mode.

---

## 2. Puzzle ideas: lasers, light and optics

Core loop: **light the receivers, then the exit door (red) opens and the next room set streams in.** Colors travel down the beam at laser speed, so timing and color both matter.

### Receivers and logic
1. **Color receptor.** A crystal on a pedestal lights up only when hit by its color. Every receptor in a room lit → the red door sinks.
2. **Sustain receptor.** It must stay lit for N seconds. Uses the glow sustain-and-fade that already exists.
3. **Sequence lock.** Three receptors must be lit in order (red, then cyan, then gold). Because color travels at laser speed, you can "send" a color down a long bounce path and switch before it arrives.
4. **Timing gate.** A receptor at the end of a 200 m bounce chain must catch a specific color *band*. You cycle colors quickly to send a red/cyan/red packet, and the receptor must see all three in order.
5. **Mixing node.** Two beams of different colors hitting the same prism produce the additive mix (red + green → yellow) and light a yellow receptor. Needs a second emitter, such as a placed relay.
6. **Shutter / filter glass.** A tinted pane passes only its own color (its IOR still bends the beam). Other colors reflect. This makes color-routed mazes.
7. **Glow battery.** A lit object keeps glowing for a few seconds, and its glow-light trips a nearby photo-sensor. Light it, then run to the next room before it fades.

### Optical items you collect (progression)
8. **Lens pickup.** Carry a glass sphere and place it on any pedestal. It focuses the flashlight into a hot point that burns through a paper or wax seal.
9. **Prism shard.** Placed in a beam, it splits white light into a fan of colors. Introduce white light as an unlockable laser color.
10. **Mirror tile.** Snap it onto any wall to change the reflection angle (rotate with the scroll wheel).
11. **Beam splitter.** A half-silvered pane sends the beam two ways at half intensity, so receptors need summed light.
12. **Fiber spool.** Hit one end and the light exits a matching node in another room. This is how puzzles span the streamed rooms.
13. **IOR crystal.** A pickup that raises the *local* refraction index of one object (the IOR slider in item form). Total internal reflection traps the beam inside it, turning it into a light pipe.
14. **Speed governor.** Slows the laser in one room to about 10 m/s. You see the photons travel, and in speed mode you can "paint" the beam into curves around obstacles.

### Room-scale puzzles
15. **Lens chamber** (already generates): focus the beam through the giant central lens onto a ceiling receptor.
16. **Pillar hall:** diamond pillars at 45°. Find the one firing position whose 30-bounce path reaches every receptor.
17. **Baffle gallery:** zig-zag panels. Crouch-shoot under them, or double-jump up for a high angle.
18. **Mezzanine:** receptors under the deck, reached only by bouncing off the floor through the glass balustrade.
19. **Tiny cube (closet):** the lens is inside and the door is shut. Open the door with F and hit the lens through the doorway gap to light a receptor outside.
20. **Darkness room:** night at 100%, no doorway glow. The only light is your beam and the afterglow of things you've hit. Shapes reveal themselves as you paint them with light.

**Satisfaction tips:** rising pitch per receptor lit (the hum is already pentatonic per color); a bass "thunk" and camera shake when the door sinks; a 0.3 s hit-stop; a receptor flash on the beam-arrival frame; a soft particle burst at bounce points.

---

## 3. Art and procedural iteration: suggested next edits

Done in this build: world-scale UVs (no stretched textures), a separate floor and ceiling texture set, crisp edge lines on props and glass, per-pixel lighting, no double walls between neighbors, and polygon rooms that always connect.

Next:
- **Room grammar instead of random rolls.** Pick a *puzzle archetype* first (lens, maze, timing, etc.), then place geometry to serve it, then decoration. Store each room's "solution path" so the generator can check it is solvable (trace the beam from the intended firing spot).
- **Critical-path graph.** Generate a spanning tree of rooms from the spawn cell. Red doors on the tree are puzzle locks; blue doors off it are optional side rooms. Today door colors are random.
- **Room tiers / biomes by distance:** office → gallery → mirror vault → void. Each tier gets its own palette, texture set and fog color.
- **Emissive ceiling panels** (liminal fluorescent tiles) that flicker. They work as cheap light cards without real lights.
- **Trim pieces:** baseboards, door casings and ceiling coves as thin boxes. They are cheap and add a lot of crispness.
- **Decals for wear:** stains, scuffs and exit signs (canvas textures).
- **Use the seed.** Add a seed input so a good layout can be shared or replayed.
- **Instanced meshes** for repeated props (pillars, steps, desks). This cuts draw calls a lot.

---

## 4. Parameters worth playing with

All live in `CONFIG` at the top of the script (`laser_bounce.html`, section 1). Sliders cover some of them.

**World / generation**
- `CELL_SIZE` (22): base room size
- `BLOCK` (3), `BIG_ROOM_CHANCE` (0.38): how often multi-cell rooms appear
- `SOLID_EDGE_CHANCE` (0.10): walls without doorways
- `RED_DOOR_CHANCE` (0.42): lift doors vs open portals
- `STREAM_RADIUS` (2): cells kept loaded (performance vs. view distance)
- `DENSITY` (slider): prop, glass and desk counts
- `DOOR_WIDTH`, `DOOR_HEIGHT`, `DOOR_OPEN_TIME`
- In `getRoomSpec`: room-type odds (closet 10%, polygon 30%), wall heights, big-room heights (8–12), feature pool
- In `createGlassSpec`: glass shape odds (spheres 28%, prisms, panes, cubes, cylinders)
- In `populateProps`: base counts of shapes, desks and glass

**Player**
- `PLAYER_SPEED`, `SPRINT_MULTIPLIER`, `CROUCH_SPEED_MULT`
- `JUMP_FORCE`, `DOUBLE_JUMP_FORCE`, `MAX_JUMPS` (set to 3 for triple jump)
- `GRAVITY`, `STAND_HEIGHT`, `CROUCH_HEIGHT`, `CROUCH_TIME` (0.30 s), `EYE_OFFSET`
- `PUSH_RADIUS`, `PUSH_FORCE` (Caps Lock push)
- `MOUSE_SENSITIVITY`

**Laser**
- `LASER_SPEED` (slider), `LASER_MAX_RANGE` (420 m)
- `laserBounces` (slider, default 30)
- `LASER_WORLD_WIDTH`, `LASER_MIN_PX`, `LASER_MAX_PX`: beam thickness up close and far away
- `HIT_LIGHT_INTENSITY`, `HIT_LIGHT_DISTANCE`: how much the bounce points light the room
- `GLOW_SUSTAIN`, `GLOW_FADE`: how long hit objects stay lit
- `GLOW_LIGHT_INTENSITY`, `GLOW_LIGHT_DISTANCE`: afterglow light
- `GLASS_IOR` (slider)
- `LASER_COLORS`: colors, order and hum pitch per color

**Lighting / look**
- `DAY_MAX` (0.56): brightest "day"
- `DOOR_LIGHT_INTENSITY`, `DOOR_LIGHT_DISTANCE`
- `FLASH_INTENSITY`, `FLASH_RANGE`, `FLASH_ANGLE`, `FLASH_PENUMBRA`
- `CAUSTIC_RAYS` (110): caustic quality vs. cost
- In `applyEnvironmentLighting`: fog density, hemisphere and directional light curves
- In `applyLightingQuality`: how many lights cast shadows per quality level
- Glass opacity (0.26) and shininess in `createGlassMesh`
- Texture tile sizes in `TILE`

**Audio**
- In `startHum` / `updateHum`: hum volume, filter cutoff, tremolo rate (5.2 Hz), harmonic mix
- `playGlassTink`, `playDoorSound`, `playLaserSwitchSound`
