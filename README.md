# CodEval

[![CI](https://github.com/SJSU-CMPE-195/group-project-team-29/actions/workflows/test.yml/badge.svg)](https://github.com/SJSU-CMPE-195/group-project-team-29/actions/workflows/test.yml)
[![codecov](https://codecov.io/gh/SJSU-CMPE-195/group-project-team-29/branch/main/graph/badge.svg)](https://codecov.io/gh/SJSU-CMPE-195/group-project-team-29)
[![PyPI](https://img.shields.io/pypi/v/assignment-codeval)](https://pypi.org/project/assignment-codeval/)

A Python utility to download student submissions to programming assignments from Canvas and GitHub and evaluate them using codeval scripts.

## Team
- Sabira Abdolcader (sabdolc)
- Chelsie Chen (cChe1z)
- Aisha Syed (aisha-syed)
- Zarah Taufique (zarahtau)

## Prerequisites
- Python 3.x
- Docker (for running evaluations in containers)
- A Canvas account with API access
- A GitHub account (for GitHub-based assignments)
- Optional AI provider packages: `anthropic`, `openai`, `google-generativeai`

## Installation

```bash
# Install locally for development
pip install -e .

# Or install from PyPI
pip install assignment-codeval

# Install with AI provider support
pip install assignment-codeval[ai]
```

## Configuration

Create a `codeval.ini` file with your Canvas and run settings:

```ini
[SERVER]
url=<canvas API>
token=<canvas token>
[RUN]
precommand=
command=
```

For distributed assignments, add:
```ini
[RUN]
dist_command=
host_ip=
[MONGO]
url=
db=
```

For mobile (Flutter/Dart) assignments, add:
```ini
[RUN]
mobile_command=docker run -i --device /dev/kvm -v SUBMISSIONS:/submissions autograder-android bash -c "cd /submissions; EVALUATE"
```

For AI benchmarking, add:
```ini
[AI]
anthropic_key=sk-ant-...
openai_key=sk-...
google_key=...
```

AI keys can also be set via environment variables: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`.

Refer to a sample config file [here](samples/codeval.ini)

## Running the Application

```bash
# List all available commands
assignment-codeval --help

# Download submissions from Canvas/GitHub
assignment-codeval download-submissions <course_name>

# Evaluate downloaded submissions
assignment-codeval evaluate-submissions <course_name>

# Upload grading comments back to Canvas
assignment-codeval upload-submission-comments <course_name>

# Create an assignment on Canvas
assignment-codeval create-assignment <course_name> <specification_file>

# Check which submissions are missing grading
assignment-codeval check-grading <course_name>

# List recent codeval comments
assignment-codeval recent-comments <course_name>

# Test an assignment with AI models
assignment-codeval test-with-ai <codeval_file> [OPTIONS]
```

## Usage

### Codeval Specification Files

Codeval files (`.codeval` extension) define how to build, run, and test student submissions.

#### Assignment Description Tags
- `ASSIGNMENT START <Assignment_name>` — begins the Canvas assignment description in markdown (`CRT_HW START` also accepted, but deprecated)
- `ASSIGNMENT END` — ends the assignment description (`CRT_HW END` also accepted, but deprecated)

#### Specification Tags

| Tag | Meaning | Function |
|---|---|---|
| C | Compile Code | Specifies the command to compile the submission code |
| CTO | Compile Timeout | Timeout in seconds for the compile command to run |
| T/HT | Test Case | Command to run to test the submission (HT = hidden test) |
| I/IB/IF | Supply Input | Input for a test case. I adds a newline, IB does not, IF reads from a file |
| O/OB/OF | Check Output | Expected output. O adds a newline, OB does not, OF reads from a file |
| E/EB | Check Error | Expected error output. E adds a newline, EB does not |
| TO | Timeout | Time limit in seconds for a test case (default: 20) |
| X | Exit Code | Expected exit code for a test case (default: 0) |
| TEMP | Temp File | Registers a file to be deleted before and after the next test run |
| CF | Check Function | Checks that a function is used in the submission |
| NCF | Check Not Function | Checks that a function is **not** used |
| CMD/TCMD | Run Command | Runs a command. TCMD fails evaluation if the command errors |
| PRINT | Print Label | Prints a label/message to stdout |
| CMP | Compare | Compares two files |
| Z | Download Zip | Zip files to download from Canvas for test cases |
| SS | Start Server | Starts a server with timeout and kill-timeout settings |
| EMULATOR | Start Android Emulator | `EMULATOR <avd_name> <boot_timeout>` — starts an Android emulator AVD and waits up to `boot_timeout` seconds for it to fully boot. The AVD is auto-created (Pixel 2, API 28) if it does not exist. Sets `ANDROID_SERIAL` for all subsequent `adb` commands. The emulator is stopped after evaluation. Requires `ANDROID_HOME` set and `emulator`, `adb`, `avdmanager`, `sdkmanager` on `PATH`. |

#### Assignment Description Macros

| Macro | Replacement |
|---|---|
| `DISCSN_URL` | URL of the discussion created for the assignment |
| `EXMPLS <n>` | First `n` non-hidden test cases formatted for display |
| `FILE[file_name]` | Link to the specified file in the Codeval folder |
| `COMPILE` | Compile command from the `C` tag |
| `GITHUB_DIRECTORY` | GitHub directory for submission |

#### Example Specification File

```
ASSIGNMENT START Hello World
# Hello World

## Problem Statement
Write a C program that reads a name from stdin and prints `Hello, <name>!`

## Submission
Submit your code to GITHUB_DIRECTORY

## Sample Examples
EXMPLS 2

## Discussion
DISCSN_URL

ASSIGNMENT END

# Compile the submission
C gcc -o hello hello.c

# Test case 1: greet "World"
T ./hello
I World
O Hello, World!

# Test case 2: greet "Alice"
T ./hello
I Alice
O Hello, Alice!
```

### AI Benchmarking

Test assignments against multiple AI models:

```bash
# Test with all Anthropic models
assignment-codeval test-with-ai my_assignment.codeval -p anthropic

# Test with a specific model, 3 attempts
assignment-codeval test-with-ai my_assignment.codeval -m "Claude Sonnet 4" -n 3
```

| Option | Description |
|--------|-------------|
| `-o, --output-dir` | Directory to store solutions and results (default: `ai_test_results`) |
| `-n, --attempts` | Number of attempts per model (default: 1) |
| `-m, --models` | Specific models to test (repeatable) |
| `-p, --providers` | Filter by provider: `anthropic`, `openai`, `google` |

### Mobile (Flutter/Dart) Assignments

Evaluate Flutter/Dart app submissions developed in Android Studio using the `EMULATOR` tag. Students submit a standard Flutter project via GitHub — the autograder builds and tests it inside the `autograder-android` Docker container.

#### Student submission

Students submit a GitHub repository containing a Flutter project created in Android Studio:

```
my_flutter_app/
├── lib/
│   └── main.dart
├── integration_test/
│   └── app_test.dart
├── test/
│   └── widget_test.dart
├── pubspec.yaml
├── android/
└── ...
```

The student does **not** need to configure anything about the emulator — the `.codeval` file handles that entirely.

#### Host / container requirements

| Requirement | Detail |
|---|---|
| KVM | Hardware acceleration — Linux host with `/dev/kvm` available. Required inside Docker with `--device /dev/kvm`. |
| Android SDK | `ANDROID_HOME` env var set to the SDK root (e.g. `/opt/android-sdk`). |
| Flutter SDK | Installed at `/opt/flutter` in the `autograder-android` image; `flutter` and `dart` are on `PATH`. |
| Tools on PATH | `emulator`, `adb`, `avdmanager`, `sdkmanager`, `flutter`, `dart` must all be accessible. |
| `lsof` | Used to identify the emulator's adb serial when multiple devices are connected. |

#### Build the Docker image

A `Dockerfile.android` is included in the repo. Build it once on the grading machine:

```bash
# Build the base image first (if not already built)
docker build -t autograder-java .

# Build the Android + Flutter image (~2 GB download, takes several minutes)
docker build -f Dockerfile.android -t autograder-android .
```

The `autograder-android` image extends `autograder-java` with the Android SDK, Flutter SDK, emulator, and a pre-created AVD.

#### Example specification file (integration tests)

```
# Fetch Flutter dependencies
C flutter pub get

# Start the emulator (AVD auto-created as Pixel 2 / API 28 if missing)
EMULATOR Pixel_2_API_28 300

# Run Flutter integration tests against the running emulator
T flutter test integration_test/app_test.dart
X 0
```

#### Example specification file (unit/widget tests — no emulator needed)

If the assignment only requires unit or widget tests, skip the `EMULATOR` tag entirely and use a standard (non-mobile) Docker container:

```
C flutter pub get

T flutter test test/widget_test.dart
X 0
```

#### Notes
- The AVD (`Pixel_2_API_28`, Pixel 2, API 28 x86 system image) is created automatically if it does not exist.
- `ANDROID_SERIAL` is set automatically to the booted emulator's serial so all `adb` and `flutter` commands target the correct device.
- The emulator is shut down cleanly after evaluation completes.
- Boot typically takes 60–180 seconds; set `boot_timeout` accordingly (300 is a safe default).
- When a codeval file contains an `EMULATOR` tag, `evaluate-submissions` automatically uses `mobile_command` instead of `command`. If `mobile_command` is not set, it falls back to `command` with a warning.
- `flutter pub get` must run before any `flutter test` or `flutter build` command so dependencies are available inside the container.

## Project Structure

```
src/assignment_codeval/
├── cli.py              # Click CLI entry point, registers all subcommands
├── submissions.py      # Download/upload submissions, list assignments
├── evaluate.py         # Run evaluation on submissions (run-evaluation)
├── create_assignment.py # Create assignments on Canvas
├── github_connect.py   # GitHub repository setup and integration
├── canvas_utils.py     # Canvas API utilities
├── ai_benchmark.py     # AI model testing (test-with-ai)
├── install_assignment.py # Install codeval files to local/remote destinations
├── recent_comments.py  # List recent codeval comments on Canvas
├── check_grading.py    # Check which submissions are missing grading
├── convertMD2Html.py   # Markdown to HTML conversion
├── commons.py          # Shared utilities
├── file_utils.py       # File handling utilities
└── test_template.html  # HTML template for test results

tests/
├── unit/               # Unit tests and sample programs/codeval fixtures
├── integration/
│   └── test_codeval.py # Integration test suite
└── e2e/
    └── test_e2e.py     # End-to-end tests

samples/
├── codeval.ini               # Sample configuration file
└── assignment-name.codeval   # Sample specification file
```