# Humanoid VLA + Residual RL

Language-conditioned loco-manipulation on the **Unitree G1** humanoid, fully in simulation (MuJoCo).
A **SmolVLA** vision-language-action model decides *what* to do from camera images and a text instruction,
a **residual RL** policy refines *how* it is done, and a **PPO locomotion policy** keeps the robot balanced.

> 🚧 **Work in progress.** Currently in Phase 1 (G1 model in MuJoCo).
> Every step, decision and problem is documented in [`docs/`](docs/).

## Architecture

| Layer | Method | Role |
|---|---|---|
| High level | SmolVLA (imitation learning) | Camera + joint state + instruction → arm targets + walking command |
| Correction | Residual RL (PPO/SAC) | Learns small corrections on top of SmolVLA's actions from task reward |
| Low level | Deep RL locomotion (PPO) | Velocity command + proprioception → leg joint targets |

ROS 2 Jazzy connects the simulator and all policies as nodes.

## Stack

MuJoCo / MuJoCo Warp (via [mjlab](https://github.com/mujocolab/mjlab)) ·
[LeRobot](https://github.com/huggingface/lerobot) / SmolVLA · PyTorch · ROS 2 Jazzy · Ubuntu 24.04

## Roadmap

- [x] Phase 0: Environment setup
- [ ] Phase 1: G1 model in MuJoCo
- [ ] Phase 2: RL locomotion policy (PPO)
- [ ] Phase 3: Task scene + scripted expert demonstrations
- [ ] Phase 4: SmolVLA fine-tuning
- [ ] Phase 5: Hierarchical integration
- [ ] Phase 6: Residual RL on top of SmolVLA
- [ ] Phase 7: ROS 2 integration
- [ ] Phase 8: Evaluation and results

## Documentation

- [Environment setup](docs/00_environment_setup.md)

## License

[Apache-2.0](LICENSE)
