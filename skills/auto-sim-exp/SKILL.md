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
| SSH command | `ssh zgc5090` |
| Deploy path | `/home/chl/diffaero_isaac/source/diffaero/deploy` |
| Docker      | `python docker.py` |
| ROS ws      | `/diffaero-deploy/` (inside Docker) |

## Pipeline

Execute **every step below in order**. All steps run autonomously — never ask the user to run commands.

### Step 1 — Open SSH Session

Use the `run_command` tool:
- `CommandLine`: `ssh zgc5090`
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

### Step 8 — Launch arena Simulation

Use `send_command_input`:
- `CommandId`: `SSH_ID`
- `Input`: `DISPLAY=:1 roslaunch accel_control yopo_oa_arena.launch path:=/root/ws/train/2026-03-17/20-44-19/checkpoints/exported_20260317_212442\n`
- `WaitMs`: `500`

---

### Step 9 — Wait for Flight and Take Screenshot

To let the drone fly for ~30 seconds, use the `command_status` tool to pause your execution. **Do NOT send empty `\n` to the terminal during this time**, as it can interrupt the simulation shell causing it to fail.

1. **First Wait (20s)**: Use the `command_status` tool:
   - `CommandId`: `SSH_ID`
   - `WaitDurationSeconds`: `20`
   - `OutputCharacterCount`: `500`

2. **Screenshot Task (at 20 seconds)**: Use a NEW `run_command` tool call to take a screenshot and transfer it:
   - `CommandLine`: `ssh zgc5090 "DISPLAY=:1 scrot /tmp/sim_screen.png"; scp zgc5090:/tmp/sim_screen.png $env:USERPROFILE\Desktop\`
   - `WaitMsBeforeAsync`: `5000`

3. **Second Wait (10s)**: Use the `command_status` tool again to finish the flight time:
   - `CommandId`: `SSH_ID`
   - `WaitDurationSeconds`: `10`
   - `OutputCharacterCount`: `500`

---

### Step 10 — Stop Simulation

Send multiple interrupts (Ctrl+C) via `send_command_input` to forcefully stop the simulation (roslaunch and arena often ignore a single `\x03`):
- `CommandId`: `SSH_ID`
- `Input`: `\x03\x03\x03\n`
- `WaitMs`: `5000`

---

### Step 11 — Wait for Simulation to Close

Use `command_status` to wait for the simulation to cleanly exit and the bash prompt to return (e.g., `root@2x5090-1:~/ws/diffaero-deploy#`).
- `CommandId`: `SSH_ID`
- `WaitDurationSeconds`: `15`
- `OutputCharacterCount`: `2000`

*(Fallback: If the prompt does not return and processes are stuck, forcefully kill them by sending `killall -9 roslaunch rviz gzserver gzclient\n` via `send_command_input`, wait `2000`ms, and send `\n` to clear the prompt).*

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
