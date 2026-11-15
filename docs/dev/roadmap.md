# forcefield roadmap

20261115 (First draft AI assisted, then human edited)

## Principles

- **Every milestone ends with a working tool.** Nothing is left broken or half-isolated between milestones.
- **Escape tests define done.** Each milestone adds tests, and every earlier test keeps passing.
- **Fail closed.** If any layer fails to apply, the agent is never exec'd.
- **One policy model.** Mounts and Landlock rules are both derived from the same in-memory policy.
- **Unsafe lives in `sys`.** Every block carries a `// SAFETY:` comment, and `unsafe_op_in_unsafe_fn` is denied.
- **Single-threaded.** Assert one thread before every fork; concurrency comes from processes.
- **Kernel floor: Linux 7.1** (Landlock ABI 9). UDP rules are used only when ABI is 10 or higher.

## Phase 1: Foundations

Phase 1 builds the policy model in safe Rust, then the `sys` layer and the self re-exec; nothing is isolated yet.

### M0: Policy model and dry run

Goal: turn a config into a complete sandbox plan and print it, with zero `unsafe`.

Build:

- Multi-call binary: `argv[0]` to switch the role.
  `forcefield` is the user CLI (`run`, `explain`, arguments parsed by hand);
   `forcefield-init`, `forcefield-fwd` and `forcefield-proxy` are internal roles with no subcommands.
  Any other `argv[0]` (renamed binary, alias, symlink) falls back to the user CLI.
- Policy loaded through a small config crate (replaced later).
- Git-root resolution; refuse `$HOME` and `/`.
- `forcefield explain` prints the derived mounts, Landlock rules and limits.

Learn: ownership, enums, `Result` and `?`, your own error type, module layout.

Done when:

- [ ] The crate contains no `unsafe`
- [ ] Unit tests cover policy derivation and the `$HOME` refusal

### M1: Sys layer and self re-exec

Goal: `forcefield run -- cmd` re-execs itself as `forcefield-init` and execs `cmd`, with no isolation yet.

Build:

- `sys` module: safe wrappers returning `io::Result`, file descriptors as `OwnedFd` and `BorrowedFd`.
- Open `/proc/self/exe` at startup; fork, then `fexecve` with `argv[0]` set to `forcefield-init`. Internal roles take inputs only as inherited fds, check them first, and set `PR_SET_NAME` for readable `ps`.
- Single-thread assertion from `/proc/self/status` before every fork.
- Exit-status relay, and an `O_CLOEXEC` pipe that reports exec failures.
- Lint gates: `unsafe_op_in_unsafe_fn`, `clippy::undocumented_unsafe_blocks`.

Learn: FFI basics, RAII handles, errno to `io::Error`.

Done when:

- [ ] Exit codes and deaths by signal are relayed faithfully
- [ ] A missing binary reports exec's errno instead of hanging
- [ ] An unknown `argv[0]` runs the user CLI; an internal role started without its fds exits with an error

## Phase 2: Process-level confinement

After M3, forcefield confines the agent with Landlock and seccomp and no namespaces, similar to Codex's Landlock-only mode. Enough for basic daily usage.

### M2: no\_new\_privs and Landlock

Goal: kernel-enforced rules for the filesystem, IPC and ports, with no namespaces yet.

Build:

- `PR_SET_NO_NEW_PRIVS`.
- Landlock through raw syscalls with ABI detection; refuse to run below ABI 9.
- Filesystem rules from the policy: RW git root, RO system paths, `$HOME` denied except an allowlist.
- Scopes for signals and abstract unix sockets; `RESOLVE_UNIX` denies every pathname socket.
- TCP connect to 443 only; UDP 53 only when ABI is 10 or higher.

Learn: the Landlock ABI model, `#[repr(C)]` structs, irreversible restrictions.

Done when:

- [ ] `sudo true` fails
- [ ] Reading `~/.ssh` fails
- [ ] Connecting to the session bus and the docker socket fails
- [ ] Writing outside the git root fails
- [ ] TCP to port 22 fails while HTTPS works

Known gap until M5: `.git/hooks`, `.git/config` and in-tree secret files stay writable or readable.

### M3: Hand-written seccomp

Goal: a cBPF denylist assembled by hand and shown to behave the same as a libseccomp oracle.

Build:

- Architecture check, and rejection of x32 syscall numbers on x86\_64.
- Deny `io_uring_*`, `bpf`, `userfaultfd`, `keyctl`, `add_key`, `perf_event_open`.
- Deny namespace creation: check `clone` and `unshare` flags. `clone3` returns `ENOSYS`, because seccomp can't read the flags behind its pointer and libc then falls back to `clone`.
- Deny the `TIOCSTI` and `TIOCLINUX` ioctls.

Learn: cBPF, the syscall ABI, why seccomp can't inspect pointer arguments.

Done when:

- [ ] Same verdict as the libseccomp oracle for every listed syscall, its neighbours, a wrong architecture and x32
- [ ] A `TIOCSTI` injection attempt fails
- [ ] `unshare -U` fails inside the sandbox

## Phase 3: Namespaces

Phase 3 moves the M2–M3 sandbox inside user, mount and PID namespaces. It closes M2's filesystem gaps and makes teardown automatic.

### M4: User namespace and sync protocol

Goal: the supervisor/init handshake, with the sandbox running inside a new user and mount namespace.

Build:

- `clone3` with `CLONE_NEWUSER | CLONE_NEWNS | CLONE_PIDFD`.
- Supervisor writes `setgroups=deny` and the uid/gid maps, your uid to itself.
- Socketpair protocol: `MAPPED`, `READY`, `ERROR{stage, errno}`.
- Ambient capabilities carry `CAP_SYS_ADMIN` across `fexecve`. Verify this first; the fallback is to unshare after the exec.
- Recursive RO bind of `/` plus the RW git root, `pivot_root`, then drop all capabilities.

Learn: user namespace semantics, capability sets across exec, runc's sync design.

Done when:

- [ ] `id` inside shows your uid and no capabilities
- [ ] A setup failure reports stage and errno, and the agent never runs
- [ ] Everything outside the git root is read-only

### M5: Constructed filesystem view

Goal: replace the RO copy of `/` with a view built from the policy, closing M2's known gap.

Build:

- tmpfs root via `fsopen`/`fsmount`; grafts via `open_tree` plus `mount_setattr` (ro, nosuid, nodev).
- Skeleton `$HOME` with allowlisted entries, a private `/tmp`, per-sandbox caches.
- RO `.git/hooks` and `.git/config`; masks over secret files.
- Minimal `/dev` that includes your pty.
- Consistency check: every Landlock path exists in the mount plan.

Learn: the new mount API, mount propagation, optionally idmapped mounts.

Done when:

- [ ] Writing `.git/hooks/pre-commit` fails
- [ ] `.env` reads as empty or missing
- [ ] `/tmp` is not the host's
- [ ] Cache writes never reach the host's `~/.cargo`
- [ ] Overwriting the forcefield binary fails

### M6: PID namespace, init and signals

Goal: init runs as PID 1, reaps and forwards signals; the kernel cleans up on exit.

Build:

- `CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS`; a fresh `/proc`.
- Reaper loop with SIGTERM and SIGHUP forwarding.
- Agent in its own process group, made foreground with `tcsetpgrp`.
- Supervisor `poll` over a signalfd and pidfds.
- Second Landlock layer applied to the agent only.

Learn: PID 1 semantics, job control, pidfd.

Done when:

- [ ] No zombies remain after a harness spawns and abandons children
- [ ] Ctrl-C reaches only the agent
- [ ] Agent exit kills every leftover process
- [ ] `/proc` lists only sandbox processes

## Phase 4: Resources (v1 complete)

M7 bounds memory, processes and CPU and completes v1: the full sandbox core, still on the host network.

### M7: cgroup v2 delegation

Goal: per-sandbox resource limits, and teardown through `cgroup.kill`.

Build:

- Re-exec under `systemd-run --user --scope -p Delegate=yes` when not already delegated.
- Leaves `supervisor/` and `sandbox/`; `CLONE_INTO_CGROUP` and `CLONE_NEWCGROUP`.
- `memory.max`, `pids.max` and `cpu.max` from the policy.
- Teardown writes `cgroup.kill`, then removes the leaves.

Learn: cgroup v2 delegation, the no-internal-process rule, the systemd cgroup driver kubelet also uses.

Done when:

- [ ] A fork bomb hits `pids.max` and the host stays responsive
- [ ] An OOM kills the sandbox, not your session
- [ ] No cgroup or temp directory is left after exit

## Phase 5: Egress and credentials

Phase 5 closes HTTPS exfiltration and takes the API key out of the sandbox. It touches none of the v1 core.

### M8: Network namespace, forwarder and CONNECT proxy

Goal: the agent reaches only allowlisted domains, through a proxy outside the sandbox.

Build:

- `CLONE_NEWNET`, with only `lo` brought up.
- `forcefield-fwd`: `127.0.0.1:N` to the proxy's unix socket, one fork per connection.
- `forcefield-proxy`: CONNECT parsing, domain allowlist, refusal of private and metadata IPs, one pump process per direction.
- The proxy hardens itself with Landlock, seccomp and no\_new\_privs; `SIGCHLD` ignored for automatic reaping.
- Agent env sets `HTTPS_PROXY`; its Landlock layer allows TCP to port N only.

Learn: network namespaces, HTTP CONNECT, inetd-style servers built from forks.

Done when:

- [ ] A non-allowlisted domain fails at the proxy
- [ ] Direct connections fail with no route
- [ ] An allowlisted name that resolves to `169.254.169.254` is refused
- [ ] Proxy death tears the sandbox down

### M9: Credential injection with libtls

Goal: the harness works with no API key anywhere in its view.

Build:

- libtls FFI (about 10 functions) behind a cargo feature, with optional `dlopen`.
- Injection route: plain HTTP to the proxy with `Connection: close`, the auth header rewritten, TLS upstream.
- Handle `Expect: 100-continue`; stream responses back.
- Secrets passed to the proxy by fd and wiped from supervisor memory.

Learn: C library FFI with opaque handles, HTTP/1.1 framing, TLS verification.

Done when:

- [ ] The harness works with an empty key in its environment
- [ ] The key appears in no `/proc/*/environ` and no file inside the sandbox
- [ ] A certificate for the wrong host is rejected

## Stretch goals (to be planned)

- **seccomp user-notify supervisor** for approval-gated access expansion (about 600–1,200 lines).
- **overlayfs snapshot and rollback** of the project tree (about 300–500 lines).
- **Landlock audit**: surface denials to the user through the ABI 7 logging flags.
- **Own config parser** to replace the temporary crate.
- **Upstream**: a Landlock proof of concept in youki for the open OCI runtime-spec PR.
- **Track the RFCs** for Landlock supervision and namespace/capability permissions.
