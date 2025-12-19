# ROB 6323 — Unitree Go2 Locomotion (Isaac Lab DirectRLEnv)

Author: Xinyang Zhang  


This repository contains my modifications to the provided ROB 6323 Go2 baseline environment.  
Main changes include: reward shaping (Parts 1,4,5,6), custom low-level PD torque control (Part 2), additional termination conditions (Part 3), and observation expansion (Part 4).

---

## 1. What I changed (Major changes + Why)

> All modifications are implemented by editing the original environment/config files (same module paths as baseline).
> No separate “new framework” was introduced — the goal is a clean, minimal diff that is easy to review and reproduce.

### Part 1 — Action Rate Penalty (smoothness)
**What:** Added action history buffer (length=3) and penalized 1st + 2nd order action differences.  
**Why:** Reduce jerky, high-frequency joint command oscillations → smoother gait & more stable learning.  
**Where:**
- `rob6323_go2_env.py`: `__init__` (history buffer + logging), `_get_rewards` (compute penalty), `_reset_idx` (clear buffer)

### Part 2 — Low-level PD Torque Controller (manual)
**What:** Disabled implicit PD in actuator config and implemented explicit torque control:
\[
\tau = K_p(q_{des} - q) - K_d \dot{q}
\]
with torque clipping.  
**Why:** Full control of gains/limits; more “robotics-style” control and consistent with tutorial.  
**Where:**
- `rob6323_go2_env_cfg.py`: set actuator stiffness/damping to 0; add `Kp`, `Kd`, `torque_limits`
- `rob6323_go2_env.py`: `_pre_physics_step` computes `desired_joint_pos`; `_apply_action` applies torques via `set_joint_effort_target`

### Part 3 — Early Termination (min base height)
**What:** Added termination when base height drops below `base_height_min`.  
**Why:** Speed up training and discourage collapsed/fallen states early.  
**Where:**
- `rob6323_go2_env_cfg.py`: `base_height_min`
- `rob6323_go2_env.py`: `_get_dones`

### Part 4 — Gait shaping (Raibert heuristic) + Observation Expansion
**What:** Implemented gait clock + desired contact schedule; added Raibert heuristic foot placement penalty; appended 4-dim clock inputs to observation.  
**Why:** Provides structured “teacher signal” for stepping & exposes gait phase to the policy.  
**Where:**
- `rob6323_go2_env.py`: `_step_contact_targets`, `_reward_raibert_heuristic`, `clock_inputs` into `_get_observations`
- `rob6323_go2_env_cfg.py`: `observation_space = 52` (48 + 4)

### Part 5 — Stability rewards (upright / less bouncing / less shaking)
**What:** Added penalties for:
- non-flat orientation (projected gravity XY)
- vertical velocity (z)
- large joint velocity
- roll/pitch angular velocity (XY)
**Why:** Encourage stable torso and reduce unnecessary motion.  
**Where:**
- `rob6323_go2_env.py`: `_get_rewards`
- `rob6323_go2_env_cfg.py`: reward scales for these terms

### Part 6 — Foot interaction rewards (clearance + contact force shaping)
**What:**  
- Foot clearance penalty during swing (avoid dragging).  
- Contact force shaping: reward contact in stance, penalize contact in swing (uses contact sensor indices).  
**Why:** Improve stepping quality and reduce scuffing/unstable contacts.  
**Where:**
- `rob6323_go2_env.py`: `_get_rewards` (feet clearance + shaped force), careful separation of `_feet_ids` (kinematics) vs `_feet_ids_sensor` (sensor indexing)

### Engineering fixes / robustness improvements (important for correctness)
- Registered the `ContactSensor` in the scene so it updates each physics step.
- More robust body index finding for base (`(base|trunk)`) and explicit foot name indexing.
- Fixed a potential dimension issue in contact-based termination logic by computing force magnitudes then max over history.

---

## 2. File overview (what to review)

Core modified files (same as baseline names/paths):
- `rob6323_go2_env.py`
- `rob6323_go2_env_cfg.py`

(If you keep “originalversion/editedversion” copies for your own record, make sure the autograder/import uses the baseline filenames above.)

---

## 3. How to run / reproduce

### 3.1 Install (baseline instructions)
Follow the provided baseline setup for Isaac Lab + dependencies (same as starter repo).  
My changes are isolated to the environment and config, so the original training pipeline should still work.

### 3.2 Train
Run the **same training entrypoint** as the baseline repo, only with this updated environment.

Common examples in Isaac Lab repos (pick the one that exists in your baseline):
- Option A (RSL-RL style):
  ```bash
  ./isaaclab.sh -p scripts/rsl_rl/train.py --task Rob6323Go2 --headless
