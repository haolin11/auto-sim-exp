---
name: auto-sim-exp
description: Run a UAV simulation experiment on a remote server via SSH. Launches the Docker environment, sets up ROS and PX4, starts the rviz simulation, then cleanly terminates the session.
---

# Auto Simulation Experiment

Autonomously execute a full UAV simulation experiment on a remote host over SSH.

## When to Activate

Activate this skill when the user asks to:
- Run a simulation experiment
- Test a flight in simulation
- Execute a uav experiment in remote server
- Run `/run_simulation_experiment`

## Connection Details

| Parameter   | Value                       |
|-------------|-----------------------------|
| SSH command | `ssh -p 6000 chl@101.201.76.56` |
| Deploy path | `/home/chl/diffaero_isaac/source/diffaero/deploy` |
| Docker      | `python docker.py` |
| ROS ws      | `/diffaero-deploy/` (inside Docker) |

## Pipeline

Execute **every step below in order**. All steps run autonomously — never ask the user to run commands.

### Step 1 — Open SSH Session

Use the `run_command` tool:
- `CommandLine`: `ssh -p 6000 chl@101.201.76.56`
- `WaitMsBeforeAsync`: `5000`

Save the returned `CommandId` as `SSH_ID`. It is used in every `send_command_input` call below.

---

### Step 2 — Navigate to Deploy Directory

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `cd /home/chl/diffaero_isaac/source/diffaero/deploy\n`
- `WaitMs`: `2000`

---

### Step 3 — Start Docker Container

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `python docker.py start\n`
- `WaitMs`: `2000`

---

### Step 4 — Enter Docker Container

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `python docker.py enter\n`
- `WaitMs`: `2000`

---

### Step 5 — Enter ROS Workspace

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `cd diffaero-deploy/\n`
- `WaitMs`: `2000`

---

### Step 6 — Source ROS Setup

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `source devel/setup.bash\n`
- `WaitMs`: `2000`

---

### Step 7 — Source PX4 Setup

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `source px4_setup.bash\n`
- `WaitMs`: `2000`

---

### Step 8 — Launch Gazebo Simulation

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `DISPLAY=:1 roslaunch accel_control yopo_oa_arena.launch path:=/root/ws/train/2026-03-17/20-44-19/checkpoints/exported_20260317_212442\n`
- `WaitMs`: `500`

---

### Step 9 — Wait for Flight

To ensure a strict ~30-second delay before exit, send two consecutive empty inputs with maximum `WaitMs` delays.

First wait:
- `CommandId`: `SSH_ID`
- `Input`: `\n`
- `WaitMs`: `10000`

Second wait:
- `CommandId`: `SSH_ID`
- `Input`: `\n`
- `WaitMs`: `10000`

Third wait:
- `CommandId`: `SSH_ID`
- `Input`: `\n`
- `WaitMs`: `10000`

---

### Step 10 — Stop Simulation

Send an interrupt (Ctrl+C) to gracefully stop the simulation:
- `CommandId`: `SSH_ID`
- `Input`: `\x03`
- `WaitMs`: `500`

---

### Step 11 — Wait for Simulation to Close

Use `command_status` to wait for the simulation to cleanly exit, indicated by the word `done` and the return of the bash prompt (e.g., `root@2x5090-1:~/ws/diffaero-deploy#`). You may need to loop this step until the prompt appears.
- `CommandId`: `SSH_ID`
- `WaitDurationSeconds`: `15`
- `OutputCharacterCount`: `2000`

---

### Step 12 — Exit Docker Container

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `exit\n`
- `WaitMs`: `2000`

---

### Step 13 — Exit SSH Session

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `exit\n`
- `WaitMs`: `2000`

---

### Step 14 — Verify SSH Exit

Use `command_status` to ensure the SSH session cleanly logs out:
- `CommandId`: `SSH_ID`
- `WaitDurationSeconds`: `5`
- `OutputCharacterCount`: `1000`

---

### Step 15 — Terminate Local Command

Explicitly terminate the local run_command task object:
- `CommandId`: `SSH_ID`
- `Terminate`: `true`

---

## Operating Principles

- **Run everything yourself** — never ask the user to execute any command.
- **Never send commands during waits** — when waiting between steps, only use `command_status` to pause yourself. Do not send `sleep` or any other commands to the remote terminal.
- **Each `send_command_input` call sends exactly one command** to the remote terminal, then waits the specified `WaitMs`.
- **Report progress** — briefly inform the user at key milestones (simulation launched, fly, exit).
- **Handle errors** — if a step fails (e.g., docker not running, roslaunch fails), read the terminal output via `command_status`, report the error to the user, and terminate the session cleanly.
