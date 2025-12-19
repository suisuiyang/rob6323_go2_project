# ROB 6323 — Unitree Go2 Locomotion (Isaac Lab / DirectRLEnv)

**Author:** Xinyang Zhang  
**Repository:** Fork of the provided baseline (ROB 6323 Go2 project)

This repo contains my modifications to the baseline Go2 locomotion task in Isaac Lab. The goal is to improve training stability and produce smooth, stable forward walking by adjusting **reward terms**, **controller logic**, **observations**, and **termination conditions**, while keeping the codebase clean and easy to reproduce.

---

## Quick start (Greene HPC)

> **Important constraints (required by the project):**
> - You must **clone your fork into `$HOME`** (not scratch/archive).
> - The directory name must be exactly **`rob6323_go2_project`**.
> - You must run **`./install.sh`** once before training.

### 1) Connect to Greene

- If you are off-campus / not on NYU Wi‑Fi, connect to **NYU VPN** first.
- Then SSH into Greene:
  ```bash
  ssh <netid>@greene.hpc.nyu.edu
  ```

### 2) Clone into `$HOME` with the correct directory name

From Greene:
```bash
cd $HOME
git clone <your-git-ssh-url> rob6323_go2_project
```

Authentication options (private repo):
- **Option A (recommended):** VS Code Remote SSH (credential forwarding)
- **Option B:** Generate an SSH key on Greene and add it to GitHub

### 3) Install the environment (required)

```bash
cd $HOME/rob6323_go2_project
./install.sh
```

This launches a setup job on **burst** (it can take a while). Check progress from Greene:

```bash
ssh burst "squeue -u $USER"
```

When the job disappears from `squeue`, installation is complete.

### 4) Launch training

```bash
cd "$HOME/rob6323_go2_project"
./train.sh
```

Monitor the job:

```bash
ssh burst "squeue -u $USER"
```

---

## What I edited (exact files)

Per the assignment, I only modified these two files:

- `source/rob6323_go2/rob6323_go2/tasks/direct/rob6323_go2/rob6323_go2_env.py`
- `source/rob6323_go2/rob6323_go2/tasks/direct/rob6323_go2/rob6323_go2_env_cfg.py`

> PPO hyperparameters live in:
> `source/rob6323_go2/rob6323_go2/tasks/direct/rob6323_go2/agents/rsl_rl_ppo_cfg.py`  
> (Not required/used for my submission unless explicitly stated otherwise.)

---

## Major changes (what + why)

Below are the major changes and why they were added. These correspond to the tutorial Parts 1–6.

### Part 1 — Action rate penalty (smoothness)
**What:** Track recent actions and penalize large changes between consecutive actions (1st/2nd order diffs).  
**Why:** Reduces jittery joint commands → smoother motion and more stable learning.

### Part 2 — Manual low-level PD torque controller
**What:** Implement explicit PD torque control and torque clipping:
\[\tau = K_p(q_{des} - q) - K_d \dot{q}\]  
**Why:** Gives direct control over gains/limits and improves stability/consistency versus implicit settings.

### Part 3 — Early termination by minimum base height
**What:** Terminate episodes when the robot base height drops below a threshold.  
**Why:** Avoid wasting compute on fallen states; speeds up training and encourages upright behavior.

### Part 4 — Gait shaping (Raibert heuristic) + observation expansion
**What:** Add gait clock / phase features + a Raibert-style foot placement shaping term; append clock inputs to observations.  
**Why:** Provides a structured stepping signal; improves periodic gait learning.

### Part 5 — Stability shaping (upright / less bouncing / less shaking)
**What:** Add rewards/penalties for:
- non-flat orientation (projected gravity XY)
- vertical velocity (z)
- large joint velocity
- roll/pitch angular velocity (XY)  
**Why:** Encourages stable torso, less bounce, and less unnecessary motion.

### Part 6 — Foot interaction shaping (clearance + contact)
**What:** Shape foot clearance during swing and contact forces according to stance/swing schedule.  
**Why:** Reduces dragging/scuffing and improves step quality and stability.

---

## Where to find results on Greene

After training, logs are written under your project clone:

```
$HOME/rob6323_go2_project/logs/[job_id]/rsl_rl/go2_flat_direct/[date_time]/
```

Each run directory typically contains:
- TensorBoard events file (`events.out.tfevents...`)
- checkpoints (`model_[epoch].pt`)
- YAML configs (exact env + PPO parameters used)
- rollout video under `videos/play/`

---

## Download logs to your computer (run locally, NOT on Greene)

From your **local machine** (Windows/macOS/Linux), use `rsync`:

```bash
rsync -avzP -e 'ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null' \
  <netid>@dtn.hpc.nyu.edu:/home/<netid>/rob6323_go2_project/logs ./
```

---

## Visualize training with TensorBoard (run locally)

1) Install TensorBoard locally:
```bash
pip install tensorboard
```

2) From the directory containing the downloaded `logs/` folder:
```bash
tensorboard --logdir ./logs
```

3) Open the shown URL (usually `http://localhost:6006/`).

---

## Submission checklist (how to turn in)

The deliverable is a **cleanly organized fork** of the baseline with modifications + a top-level `README.md` describing changes and reproduction.

Before submitting:
- [ ] Repo is a **fork** of the provided baseline.
- [ ] Repo contains this **top-level `README.md`** (major changes + how to reproduce).
- [ ] The two edited files are committed:
  - `.../rob6323_go2_env.py`
  - `.../rob6323_go2_env_cfg.py`
- [ ] Code has concise inline comments where logic differs from baseline.
- [ ] **Do not commit large artifacts** (`logs/`, checkpoints, videos) unless the course explicitly asks.
- [ ] Verify a clean run:
  - `./install.sh` completed (burst job finished)
  - `./train.sh` launches successfully
  - logs appear under `logs/...`

**How to submit (typical):**
- If the course portal asks for a **GitHub link**: submit the URL of your fork (optionally include the final commit hash).
- If it asks for a **zip**: create one without logs (example from repo root):
  ```bash
  git archive --format=zip -o rob6323_go2_submission.zip HEAD
  ```

---

## References (official guides used)
- NYU HPC access guide (VPN/SSH, dtn, gateway options)
- NYU HPC VS Code Remote SSH guide (recommended workflow)
