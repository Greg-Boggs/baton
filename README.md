# Baton

Baton makes GNOME logout instant and keeps the screen from ever going black. The greeter takes over the screen right away, and your session shuts down in the background.

## The problem

On stock Ubuntu 26.04 (GNOME 50, Wayland, NVIDIA), logging out takes tens of seconds. During that time the screen freezes or goes black and gives no sign that the computer is doing anything.

The cause is how logout is built, not one bug. Logout runs as a chain of steps, one after another:

1. Stop every process in the user session.
2. Only then give up the display.
3. Only then cold-start a new greeter (the GDM login screen).

Nothing owns the screen between those steps, and every slow step adds to what the user sees. Two bugs make it worse:

- **gnome-shell doesn't exit on SIGTERM.** Its GJS and GL teardown hangs, so systemd waits out the 5s stop timeout and then SIGKILLs it. Upstream: [gnome-shell#5560](https://gitlab.gnome.org/GNOME/gnome-shell/-/work_items/5560).
- **Apps ignore SIGTERM.** Discord, ChatGPT, codex and localsearch-3 keep `user@UID.service` waiting for another timeout, and then get SIGKILLed anyway.

Journal from one logout on 2026-09-27:

| Time | Event |
|---|---|
| 08:37:18 | gnome-shell SIGKILLed after stop timeout |
| 08:37:27 | logind removes the user session |
| 08:37:36 | greeter session created |
| 08:37:42 | `user@1000.service` times out, apps SIGKILLed |
| 08:37:49 | greeter ready |

## The design

Baton swaps the order: **hand over the screen first, clean up afterwards.**

1. **A greeter that stays running.** GDM keeps its greeter alive after login instead of killing it. At logout the greeter takes over `seat0` at once, the same way Switch User already works. The screen always has an owner.
2. **Background teardown.** Once the greeter has the seat, the old session can't be seen or reached, so it shuts down with nobody waiting on it.
3. **Fast shell exit.** gnome-shell gives up the display and exits right away, skipping the teardown that hangs. The kernel frees GPU memory on exit anyway.
4. **Bounded, parallel app shutdown.** Unsaved-work prompts happen *before* the handoff. After it, all apps get SIGTERM at once, and after a hard 2s deadline anything left in the session cgroup is killed.
5. **Visible progress.** The greeter shows a status line such as "Signing out of Greg… closing Discord" until teardown finishes.

Targets: login screen visible within 1s of clicking Log Out, never a black screen, old session fully gone within 5s.

## Work tracking

Everything is tracked in [GitHub issues](https://github.com/Greg-Boggs/baton/issues). Start with the measurement issue: it gives us the baseline every other change gets checked against. The issue that keeps the greeter running is the one that matters most for how logout feels.

## What we patch

We own this stack. Baton carries patches to Ubuntu's source packages and builds local `.deb` files, pinned in apt.

| Package | Change |
|---|---|
| `gdm3` | keep the greeter running, hand it the seat at logout, don't wait for teardown |
| `gnome-shell` / `mutter` | fast exit path on logout, sign-out status in the greeter UI (`js/gdm/`) |
| `gnome-session` | unsaved-work prompts before handoff, parallel app stop with a deadline |
| systemd drop-ins | stop timeouts for session units |

Everything that works gets sent upstream to GNOME, so we can eventually drop our patches.

## Target machine

- Ubuntu 26.04.1 LTS, kernel 7.0
- GNOME Shell 50.1, gdm3 50.1, Wayland only (26.04 has no X11 GNOME session)
- NVIDIA GTX 1660 SUPER, `nvidia-driver-610`, `nvidia_drm modeset=1`
- Local patched `.deb` files follow the layout of `~/clicklock/`

## Notes for agents

- **Testing a real logout ends your own session, including you.** Test with a second user account, a VM, or a nested/devkit gnome-shell. Only log out the main session when the user asks you to.
- **Read the journal.** `journalctl -b -o short-iso` around a logout shows every phase. Look for `stop-sigterm timed out`, `Removed session`, and `New session ... gdm-greeter`.
- **Stop timeouts are 5s** for both `org.gnome.Shell@ubuntu.service` and `user@1000.service` (`systemctl show -p TimeoutStopUSec`).
- **Lock is not logout.** Locking keeps the session running behind a curtain and is already instant. Logout really ends the session: it frees the GPU, drops the session bus, and hands the seat back to GDM. Baton makes logout feel like lock without giving up that separation.
- **Never edit vendor files in `/usr` or `/lib`.** Changes go in patches, drop-ins, or config under `/etc`, and must be reversible.

## License

GPL-3.0. See [LICENSE](LICENSE).
