# 3D Pool — Vulkan

A fully 3D billiards game rendered with the Vulkan graphics API, built on the **libmx2 (MX2)** engine. Pocket all 15 object balls in as few shots as possible and post your name on the high-score board.

---

## Screenshots

<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/e717f114-c559-49cc-bfd2-0ea3a8b8ee01" />
<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/738fdf5a-eb7d-4d8b-bbff-4ea67335d7f3" />
<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/f4826233-4346-4efd-b1b4-c3f92a635175" />
<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/532f8c3e-9abf-4e01-8cc1-6b3ba143779a" />


---

## Features

- Real-time 3D Vulkan rendering with perspective camera (Requires your Graphics Card supports Vulkan)
- Full physics simulation — friction, ball–ball collisions, wall bounces, pocket detection
- Animated ball-sink effect when a ball drops into a pocket
- Rack of 15 object balls + cue ball, 6 pockets
- Power charge system — hold and release to set shot strength
- Freeform camera rotation and zoom (keyboard & gamepad)
- Configurable cue ball re-placement after a scratch
- Persistent high-score table (top 10, saved to `pool_scores.dat`)
- Gamepad support via SDL2 Game Controller API
- Wireframe debug mode

---

## The MX2 Engine (libmx2)

3D Pool is built on **libmx2**, a cross-platform C++20 multimedia engine that wraps SDL2 and provides three independently-built modules:

| Module | Purpose |
|--------|---------|
| **mx** | Core: SDL2 window management, input, font rendering (SDL2_ttf), audio (SDL2_mixer), PNG loading, config, utilities |
| **mxgl** | Optional OpenGL/GLAD layer: GL shader helpers, 3D model loading, in-app console |
| **mxvk** | Vulkan layer: Vulkan initialisation (via volk / MoltenVK), pipeline & shader helpers, 3D model rendering, sprite batching, text rendering |

3D Pool uses **mx** + **mxvk** — no OpenGL dependency.

Key MX2 classes used by this project:

- `mx::VKWindow` — Vulkan-backed SDL2 window; provides the swap chain, render pass, command buffers, semaphores, and the `initVulkan() / proc() / draw() / event() / cleanup()` lifecycle.
- `mx::MXModel` — Loads and renders `.mxmod.z` (compressed binary mesh) files through Vulkan vertex/index buffers.
- `mx::VKSprite` — Batched 2D sprite renderer; used for background and screen overlays.
- `mx::UniformBufferObject` — Standard MVP + params UBO uploaded per draw call.
- `mx::Exception` — Engine exception type.

3D models are stored as `.mxmod.z` files (zlib-compressed MX model format) and SPIR-V shaders are pre-compiled to `.spv` files — both live under `data/`.

**libmx2 source & documentation:** https://github.com/lostjared/libmx2  
**Tech spec page:** https://lostsidedead.biz/libmx2/mx.html

---

## Dependencies

| Dependency | Role |
|------------|------|
| libmx2 (mx + mxvk) | Engine — Vulkan window, models, sprites, text |
| Vulkan loader **or** MoltenVK (macOS) | GPU API |
| SDL2 | Window, input, events |
| SDL2_ttf | In-game text rendering |
| SDL2_mixer | Audio subsystem (engine requirement) |
| GLM | Math — vectors, matrices, transforms |
| libpng + zlib | PNG texture loading |

### Install dependencies

**Arch Linux**
```bash
sudo pacman -S sdl2 sdl2_ttf sdl2_mixer glm libpng vulkan-icd-loader vulkan-headers
```

**Ubuntu / Debian**
```bash
sudo apt install libsdl2-dev libsdl2-ttf-dev libsdl2-mixer-dev libglm-dev \
                 libpng-dev libvulkan-dev vulkan-validationlayers
```

**macOS (Homebrew)**
```bash
brew install sdl2 sdl2_ttf sdl2_mixer glm libpng molten-vk
```

Then install **libmx2** (built with Vulkan support):
```bash
git clone https://github.com/lostjared/libmx2.git
cd libmx2
mkdir build && cd build
cmake -B . -S ../libmx -DVULKAN=ON -DOPENGL=OFF
make -j$(nproc)
sudo make install
```

---

## Building

```bash
# Linux / Windows (Vulkan loader)
mkdir build && cd build
cmake ..
make -j$(nproc)

# macOS (MoltenVK — auto-detected from Homebrew)
mkdir build && cd build
cmake ..          # MOLTEN=ON is the default on Apple platforms
make -j$(nproc)

# macOS — specify MoltenVK path manually if needed
cmake .. -DMOLTEN_PATH=/opt/homebrew/opt/molten-vk
make -j$(nproc)

# Override MoltenVK detection and use the Vulkan loader on macOS
cmake .. -DMOLTEN=OFF
make -j$(nproc)
```

The CMake configure step checks that every required header (`vk.hpp`, `SDL.h`, `glm/glm.hpp`, `vulkan/vulkan.h`, etc.) and every runtime asset (`data/*.mxmod.z`, `data/*.spv`, `data/*.png`, `font.ttf`) is present before compilation begins. It will print clear `FATAL_ERROR` messages with install hints if anything is missing.

After a successful build the binary and the `data/` directory are placed together in the build output folder.

---

## Running

```bash
# From the build directory — '.' tells the engine where to find data/ and font.ttf
./3DPool -p .

# Fullscreen
./3DPool -p . -f

# Custom resolution (default: 1280×720)
./3DPool -p . -r 1920x1080

# Show all options
./3DPool -h
```

| Flag | Description |
|------|-------------|
| `-p <path>` / ---path <path> | Path to the directory containing `data/` and `font.ttf` (required) |
| `-f` / `--fullscreen` | Fullscreen mode |

---

## How to Play

### Objective

Pocket all 15 coloured object balls in as few shots as possible. Your score is the total number of shots taken. Lower is better.

### Game Screens

| Screen | Description |
|--------|-------------|
| **Intro** | Animated logo plays for 5 seconds, then transitions to the Start screen |
| **Start** | Main menu — choose to play or view the leaderboard |
| **Game** | The billiards table — aim, charge, and shoot |
| **Scores** | High-score table; enter your name if you qualify |

### Keyboard Controls

#### Menus

| Key | Action |
|-----|--------|
| `Enter` | Start game (Start screen) |
| `Space` | Open high scores (Start screen) |
| `ESC` | Quit; or return to Intro from Scores screen |

#### In-Game — Camera

| Key | Action |
|-----|--------|
| `A` | Rotate camera left |
| `S` | Rotate camera right |
| `W` | Zoom in |
| `E` | Zoom out |

#### In-Game — Aiming & Shooting

| Phase | Key | Action |
|-------|-----|--------|
| Aiming | `←` / `→` | Rotate the cue left / right |
| Aiming | `Space` | Begin charging the shot |
| Charging | `←` / `→` | Still adjust aim while charging |
| Charging | Release `Space` | Fire — power is determined by how long you held Space |
| Placing (after scratch) | `←` `→` `↑` `↓` | Move the cue ball around the table |
| Placing | `Enter` | Confirm cue ball position and return to Aiming |

#### Other

| Key | Action |
|-----|--------|
| `R` | Rack and restart the current game |
| `P` | Toggle wireframe debug view |

### Gamepad Controls

| Stick / Button | Action |
|----------------|--------|
| Left Stick (L/R) | Rotate cue (Aiming / Charging) or move cue ball (Placing) |
| Left Stick (U/D) | Move cue ball forward / back (Placing) |
| Right Stick (L/R) | Rotate camera |
| Right Stick (U/D) | Zoom camera in / out |
| **A** or **B** | Charge shot (hold) then release to fire; confirm placement; start game |
| **Y** | Open scores (Start screen) |
| **Start** | Start game |
| **Back** | Quit |

### Shot Mechanics

1. **Aim** — use `←`/`→` (or Left Stick) to rotate the cue around the cue ball.
2. **Charge** — press `Space` (or A/B). A power percentage is displayed; it fills automatically.
3. **Shoot** — release `Space` (or the button). The cue ball launches in the aimed direction with the charged power.
4. **Physics** — the simulation runs at a variable sub-step rate to prevent tunneling at high speeds. Friction is applied per step; balls rebound off the cushions with 80 % energy retained.
5. **Scratch** — if the cue ball is pocketed it is placed back on the table and you enter Placing mode.
6. **Game Over** — once all 15 object balls are pocketed you are taken to the Scores screen.

### High Scores

Scores are saved to `pool_scores.dat` in the working directory. The table holds the top 10 entries sorted by lowest shot count. If your score qualifies you will be prompted to enter your name (up to 15 characters). Press `Enter` to confirm or `Backspace` to delete.

---

## Project Structure

```
3DPool/
├── main.cpp          # Entire game — PoolWindow class + physics + rendering
├── font.ttf          # TrueType font used for all in-game text
├── CMakeLists.txt    # Standalone CMake build with full dependency checks
└── data/
    ├── *.mxmod.z     # Compressed 3D models (sphere, cylinder, cube, pool table parts)
    ├── *.spv         # Pre-compiled SPIR-V shaders
    └── *.png / *.bmp # Textures and UI images
```

---

