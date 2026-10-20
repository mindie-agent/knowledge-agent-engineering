---
category: axiom
conditions:
  a3_dual_die_topology: 8_logical_npus_x_2_dies
  accelerator_family: Ascend A2/A3
  device_visibility_variable: ASCEND_RT_VISIBLE_DEVICES
  execution_surface: shared_container_cluster
  isolation_required:
  - git_worktree
  - tmp_state_directory
  rendezvous_variables:
  - MASTER_PORT
  - HCCL_IF_BASE_PORT
domain_path: engineering/npu-coordination
evidence_strength: software_contract
id: unit-multi-agent-npu-lease-and-container-isolation-axioms
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/agent-engineering-npu-provenance.json
  - cannbot-knowledge@7ea0efbeb79f2559f121fa8514a9fe5c82245807:knowledge/common/platforms/concepts/target_ascend910c.md
  - mindie-bench@c42493d6ff916debf2967b76adace62113b41340:AGENTS.md
  - msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/quantizer/msmodelslim-adapter-verification/references/verification_guide.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:model/model-infer-parallel-impl/references/standalone_parallel.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/collaborative-agent-kernel-evolution/skills/remote-cann-development/SKILL.md
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:plugins/mindie-agent/skills/mindie-agent/references/domain-skills.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/collaborative-agent-kernel-evolution/skills/git-version-management/SKILL.md
  source_digest: d36d173738cc47863d74fddd3772ae381c78503f1884822f02d16add39adda1b
  source_locator: docs/evidence/agent-engineering-npu-provenance.json
  source_type: derived_software_contract
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L1
title: Lease NPU devices, ports, worktrees, and state atomically across agents
triggers:
  error_signatures:
  - ASCEND_RT_VISIBLE_DEVICES
  - MASTER_PORT
  - HCCL_IF_BASE_PORT
  - address already in use
  - HcclCommInitRootInfoConfig
  - '507033'
  - ACL_ERROR_RT_DEV_PERMISSION_DENIED
  - device is occupied
  keywords:
  - multi-agent
  - npu lease
  - device isolation
  - container isolation
  - Phy-ID
  - logical NPU
  - A3 dual-die
  - git worktree
  - tmp state
  - port collision
  - rendezvous
  - exclusive timing lease
content_digest: sha256:8b3f9fbebbc31a8459a4c1730eb3eaf2d8f218a0b850d056925a43b7459f7b2e
---
# Axiom

Treat the tuple `(host, container, die set, rendezvous-port block, git worktree, /tmp state root)` as one lease. Acquire it atomically before launching any process that can touch an NPU, a collective port, or shared model state. Release it only after the process tree and its device/port residue are drained.

A check followed by a bind is not a lease. Another agent can pass the same check between the check and the bind. The result is two processes that each believe they own the same device, the same rendezvous port, or the same mutable worktree.

## A1. Device visibility is a reservation, not a free-card query

- `ASCEND_RT_VISIBLE_DEVICES` describes what a process may see. It does not prove that another process is not already using those devices.
- Before launch, reserve every die the run will use. Record the lease owner (`run_id`), the exact logical-device list, the corresponding physical die identity, the container, the expiry, and the coordinator or single-writer store that granted the lease.
- If any component cannot be reserved atomically, fail closed. Do not “pick another card” after a failed bind without re-running the full inventory and re-acquiring the lease.
- Keep the lease through warmup, graph capture, benchmark, and profiler collection. A performance measurement is valid only under an exclusive timing lease on the same physical NPU or the same A3 dual-die package.
- Release only after the process group is gone and the device inventory shows no residual process or allocated workspace. If release cannot be proven, mark the lease `unknown` and keep it out of the free pool.

A useful lease record is deliberately boring:

```yaml
lease:
  run_id: <run_id>
  owner: <agent-or-job-id>
  host: <host>
  container: <container>
  logical_npus: [0, 1, 2, 3]
  phy_ids: [0, 1, 2, 3]
  ports:
    master: 29500
    hccl_base: 23456
  worktree: <workspace>/<run_id>
  tmp_root: /tmp/<task>-<run_id>
  expires_at: <timestamp>
  release_evidence: <inventory-receipt>
```

## A2. Resolve A3 dual-die Phy-ID to logical NPU ID before container binding

An A3 package contains two dies. On an 8-package node the driver exposes 8 logical NPUs (logical IDs `0..7`) x 2 dies, which is 16 die-level `Phy-ID` values in the range `0..15`. The logical NPU namespace used by the framework, the die-level `Phy-ID` namespace reported by `npu-smi`, and container device nodes such as `/dev/davinci*` are related but not interchangeable.

Before binding a container:

- Inventory the host with a read-only `npu-smi info` or the platform equivalent. Record both the `Phy-ID` and the logical NPU ID that the framework will see.
- Pass the resolved logical list through `ASCEND_RT_VISIBLE_DEVICES`; record the resolved die identity in the lease.
- Re-resolve the mapping after a container restart, driver reload, host replacement, or a different container image. Never carry a mapping forward from another host or from a previous run.
- Never infer the mapping from `ASCEND_RT_VISIBLE_DEVICES=N` alone, from `/dev/davinciN`, or from an earlier run on a different container.
- If the mapping is ambiguous, stale, or inconsistent between host and container, stop before container device binding. A wrong die binding can look like a model bug, a corrupt collective, or a bad kernel.

The mapping is a property of the observed host. This axiom requires the mapping to be resolved; it does not assert one universal mapping for every A3 SKU.

## A3. Rendezvous ports belong to the same lease

`MASTER_PORT` and `HCCL_IF_BASE_PORT` are shared host/container resources. They are not harmless defaults.

- Reserve a contiguous port block per run and pass explicit values to the launcher.
- Check listeners in the same network namespace that the container will use. A host-side check does not prove that a container-side port is free.
- Treat `MASTER_PORT=29500` (torchrun default), framework ports such as `6038` and `6138`, and `HCCL_IF_BASE_PORT=23456` as one collision domain when co-locating runs.
- A rendezvous bind failure is an allocation failure. Do not classify it as a model defect or a kernel defect until the lease record and the listener table are reconciled.
- When a run ends, release the port block only after the process tree is drained. A `kill` that leaves a listener or a zombie bootstrap process is not a release.

## A4. Isolate worktrees and `/tmp` state per agent run

- Each agent run gets its own Git worktree and branch, created from a recorded base commit. Never let two agents edit the same worktree.
- Each run gets its own `/tmp/<task>-<run_id>/` for logs, receipts, profiling extracts, build staging, and scratch checkpoints.
- Shared build caches are read-only or per-slot. Use `MINDIE_SLOT_ID`, `ASCEND_CACHE_PATH`, `TORCHINDUCTOR_CACHE_DIR`, and `TRITON_CACHE_DIR` to separate writable cache state.
- Use `git -C <worktree>` with a realpath guard for every `add` and `commit`. Do not rely on shell cwd; a commit can land in the parent repository.
- Never clean another run's worktree, `/tmp` root, container, or process tree. The lease grants cleanup rights only over its own paths.
- An interrupted subagent's worktree is not evidence and is not deployable. Inspect it, record its diff digest, and leave it unmerged until the parent explicitly verifies it.

## A5. Release is a state transition, not a `kill`

A release is complete only when all of the following are recorded:

1. The process group received `SIGTERM`, a bounded drain elapsed, and surviving processes received `SIGKILL`.
2. The device inventory shows no residual process, memory allocation, or workspace from the run.
3. The port listeners and bootstrap processes from the run are gone.
4. The worktree and `/tmp` root are either retained as evidence or removed deliberately, never reused silently by another run.

If any item is unknown, the lease remains `unknown`. Do not return those resources to the free pool, and do not retry a mutating launch against them until they are reconciled.

## Failure signatures

- `507033` or `ACL_ERROR_RT_DEV_PERMISSION_DENIED` after a process looked healthy: suspect a device-lease collision or a stale die binding.
- `address already in use`, a rendezvous timeout, or `HcclCommInitRootInfoConfig` failure before model execution: suspect a `MASTER_PORT` / `HCCL_IF_BASE_PORT` collision or a rank/port mismatch.
- Two agents each report the same “free” device: the allocation was check-then-bind, not a lease.
- The host sees one device list and the container sees another: the logical-to-`Phy-ID` mapping was not resolved for that container.
- A run inherits stale metadata, logs, or cache entries from a previous `run_id`: the `/tmp` or cache root was not isolated.
- A commit appears on the parent branch instead of the variant branch: a `git -C` guard was missing or the worktree path was wrong.

## Scope and evidence boundary

This is an L1 software-contract axiom derived from the A3 dual-die platform contract, logical-device mapping documentation, rendezvous-port documentation, the remote-development contract, the exclusive timing-lease contract, and the worktree contract cited in the provenance record. It is reinforced by the recorded multi-agent session, where one agent's host was treated as reserved, an interrupted subagent left an unmerged worktree, and long-running service work was repeatedly affected by container and environment differences.

It does not establish a collision frequency, a universal A3 mapping, or a measured speedup. The practical rule is stronger than the evidence: if the lease cannot be acquired and released with evidence, do not start the run.
