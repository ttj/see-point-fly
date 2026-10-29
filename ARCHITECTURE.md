# Architecture

Source map checked at `5621bcf43e9826d60df014541dd0498e743a92bd`.
This repository is the See, Point, Fly research implementation: language instructions and images become relative movement actions through a vision-language model (VLM). It supports three different execution backends. This guide describes the checked source; the paper and README's general navigation claims are not lab qualification evidence.

## Read first

- [README.md](README.md) describes installation, API configuration and the three modes.
- [pyproject.toml](pyproject.toml) declares Python 3.13+, dependencies and the `spf` / `spf-tools` console entrypoints.
- [src/spf/main.py](src/spf/main.py) owns command-line parsing and dispatch. The mode implementations own their processing loops.

## Module map

| Area | Responsibility and useful seam |
| --- | --- |
| [spf.main](src/spf/main.py) | `main()` parses the `sim`, `airsim` or `tello` mode and shared flags, then imports the selected mode. |
| [base/action_projector.py](src/spf/base/action_projector.py) | `ActionProjector` loads the mode YAML, initializes the VLM client, and supplies camera projection and visualization helpers. Subclasses implement VLM action extraction. |
| [base/drone_space.py](src/spf/base/drone_space.py) | `ActionPoint` carries relative `dx/dy/dz`, action type and image coordinates. `DroneActionSpace` defines command conversion and a duration-based internal state estimate. |
| [clients/vlm_client.py](src/spf/clients/vlm_client.py) | `VLMClient.generate_response()` encodes an image and calls Gemini or an OpenAI-compatible endpoint. `clean_response_text()` removes response fences before mode-specific parsing. |
| [tello/](src/spf/tello) | `main.py` manages camera readiness, interactive instructions and display; `TelloActionProjector` extracts actions; `TelloDroneActionSpace` converts them to timed commands; `TelloController` queues and executes DJI SDK RC control through `djitellopy`. |
| [sim/](src/spf/sim) | `main.py` captures a desktop monitor. `SimActionProjector` and `SimDroneActionSpace` feed `SimController`, whose execution backend presses keyboard keys through `pynput`. |
| [airsim/](src/spf/airsim) | `main.py` acquires simulator camera images. `AirSimActionProjector` / `AirSimDroneActionSpace` feed `AirSimController`, which calls the AirSim multirotor API. |
| [spf_tools/](src/spf_tools) | `cli.py:main()` exposes diagnostics, screen capture, VLM accuracy, monitor and resolution tools. [Its README](src/spf_tools/README.md) explains their use. |

## Data and control flow

The selected mode acquires an image and the current text instruction. Its controller's `process_spatial_command()` asks the mode projector for `ActionPoint` objects. The projector builds the VLM request, parses the returned coordinates/depth or obstacle information, and converts image points into relative actions. The mode's action space translates an action into backend commands; the controller sends those commands through its queue and execution loop. The display and saved visualizations expose intermediate image/action information.

The common projector treats `x` as lateral, `y` as forward depth and `z` as up; image `y` increases downward. These are local projection/action coordinates, not a surveyed lab ENU pose. `DroneActionSpace.update_state()` integrates nominal movement durations and yaw rates; it is not a sensor-fused localization provider. AirSim additionally has its API's own coordinate and velocity conventions, handled in its controller/action space.

The root [config_tello.yaml](config_tello.yaml), [config_sim.yaml](config_sim.yaml) and [config_airsim.yaml](config_airsim.yaml) select provider/model and mode behavior. `VLMClient` loads environment variables through `python-dotenv`: `GEMINI_API_KEY` or `OPENAI_API_KEY`, with optional `OPENAI_BASE_URL`. The source default for the OpenAI-compatible client is OpenRouter; do not assume that provider name means the request goes to OpenAI's endpoint.

## Setup and entrypoints

Follow the existing README's `uv sync` setup from the repository root. `uv run spf --help` describes the dispatcher; `uv run spf-tools --help` describes the companion tools after dependencies are installed. Package initialization imports the VLM client even for help, so the declared SDK dependencies must be available.

The existing operational entrypoints are `uv run spf sim`, `uv run spf airsim` and `uv run spf tello`. Use the README's backend-specific setup before selecting one: the first interacts with the desktop simulator, the second needs a running AirSim environment, and the third connects to a physical Tello. The tool suite is also active: capture reads the screen, VLM checks can make paid/network requests, and diagnostics can initialize projectors and create output directories. These commands are not an offline unit-test suite.

## Traps and change boundaries

- Run from the repository root: projectors and mode loops open configuration files by relative filename. Installing the console script does not make those YAML paths independent of the working directory.
- Backend constructors have effects. `TelloController.__init__()` connects and starts streaming; `AirSimController.__init__()` connects, enables API control and arms the simulated vehicle; `SimController` creates a real keyboard controller. Read a mode's `main()` before using flags such as `--test` as a dry-run promise.
- Screen capture dimensions and projection dimensions must agree. Simulator code and the resolution tools explicitly check this relationship; a desktop scaling change can change the coordinates presented to the model.
- Action conversion is backend-specific. Tello/simulator timed key or RC commands and AirSim velocity/yaw commands are not interchangeable movement guarantees.
- Recorders and visualizers write under the current working directory, including `action_visualizations`, `raw_frames` and `tello_videos`. Keep experiment outputs distinct from source changes.
- This source snapshot has no checked-in automated test directory or GitHub Actions workflow. This guide does not establish model accuracy, obstacle avoidance performance or flight readiness.

Index searched: the lab's `vu-autonomy/docs/CODE_MAP.md` and `CODE_INDEX.json`; safe sibling README/docs searches for map, overhead, route, tour, planner, pose, tag, guard, filing, render, metrics and recorder; this repository's README, package metadata, mode implementations, tools and architecture PR/issue search.

Reusing: `spf.main:main`, `ActionProjector`, `VLMClient`, `ActionPoint`, the existing mode projectors/action spaces/controllers and `spf_tools.cli:main`. New because: the repository lacked a root architecture onboarding guide; no executable capability is introduced.
