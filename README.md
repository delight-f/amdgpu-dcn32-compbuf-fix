# DCN 3.2 `dcn32_program_compbuf_size` REG_WAIT timeout — fix

## Summary

A one-file, low-risk kernel patch for the amdgpu display driver that fixes (and
silences) the recurring message:

```
amdgpu 0000:2b:00.0: [drm] REG_WAIT timeout 1us * 100 tries - dcn32_program_compbuf_size line:147
```

Hardware affected: **DCN 3.2** (Radeon RX 7000-series, Navi 31/32/33).
Patch file: `0001-drm-amd-display-fix-compbuf-wait-timeout-on-DCN-3-2.patch`

## Root cause

`dcn32_program_compbuf_size()` in
`drivers/gpu/drm/amd/display/dc/hubbub/dcn32/dcn32_hubbub.c` resizes the display
hub's compressed buffer. Before programming the new size it waits for the four
`DCHUBBUB_DETx_CTRL` size registers to latch, using four open-coded calls:

```c
REG_WAIT(DCHUBBUB_DET0_CTRL, DET0_SIZE_CURRENT, hubbub2->det0_size, 1, 100);
...
```

`REG_WAIT(reg, field, val, delay_us, max_attempts)` gives the poll a **100 µs**
budget (1 µs × 100). On DCN 3.2 the DET size update can take longer than that,
so the poll expires and the driver logs a `REG_WAIT timeout`.

The equivalent DCN 3.1 routine was already fixed upstream by replacing those
four calls with `dcn31_wait_for_det_apply()` (1000 µs × 30 ≈ **30 ms**), in the
patch *"drm/amd/display: fix compressed buffer config routine waiting time"*.
That change touched **only `dcn31_hubbub.c`** — the DCN 3.2 twin was left with
the original, too-short timeout.

This is the DCN 3.2 counterpart of the DCN 3.1 fix.

## What the patch changes

`drivers/gpu/drm/amd/display/dc/hubbub/dcn32/dcn32_hubbub.c`, +27 / −4:

1. Adds `dcn32_wait_for_det_apply()`, mirroring the DCN 3.1 helper, using the
   same `REG_WAIT(..., 1000, 30)` timing.
2. Replaces the four open-coded `REG_WAIT(..., 1, 100)` calls in
   `dcn32_program_compbuf_size()` with calls to the new helper.
3. Registers `.wait_for_det_apply = dcn32_wait_for_det_apply` in
   `hubbub32_funcs`, matching the DCN 3.1 function table.

No functional change beyond the corrected timeout.

## Apply and build

The patch is generated against upstream `master`, and the hunk context matches
kernel 7.0.0-31 line-for-line (`line:147`/`line:148` in your dmesg), so it
applies to both.

```sh
# in a kernel source tree for your running kernel (or a mainline/drm-next tree)
git apply --check /path/to/0001-drm-amd-display-fix-compbuf-wait-timeout-on-DCN-3-2.patch
git apply        /path/to/0001-drm-amd-display-fix-compbuf-wait-timeout-on-DCN-3-2.patch

# build and install using your tree's usual flow, e.g. for a distro tree:
make olddefconfig
make -j"$(nproc)" && sudo make modules_install && sudo make install
sudo update-initramfs -u -k all
sudo reboot
```

If you would rather not rebuild your distro kernel, applying the patch on top of
a mainline build you already run is the quickest way to test it.

## Verify

After rebooting into the patched kernel, run for a while under the same display
workloads and re-check:

```sh
dmesg | grep -c 'dcn32_program_compbuf_size'
```

Expected result: no new occurrences. Before the patch this system logged 11 in
~38 h of uptime.

## Submitting upstream

The patch is formatted with `git format-patch` and is authored and signed off as
`Faraaz de Belder <faraaz@debelder.com>`.

1. Subscribe to amd-gfx, then apply the patch to the current DRM tree
   (`drm-next` / `amd-staging-drm-next`).
2. Send to the amd-gfx list with the DRM maintainers in copy:
   ```sh
   git send-email --to=amd-gfx@lists.freedesktop.org \
       --cc=harry.wentland@amd.com --cc=sunpeng.li@amd.com \
       --cc=Rodrigo.Siqueira@amd.com --cc=airlied@gmail.com \
       0001-dcn32-compbuf.patch
   ```

`scripts/checkpatch.pl` should be clean; the patch was checked for whitespace
errors and applies cleanly to the pristine upstream file.

## Related, but not covered here

The other message in your log,
`workqueue: dm_handle_vmin_vmax_update [amdgpu] hogged CPU for >10000us`, is a
separate issue. An upstream patch already addresses it —
*"drm/amd/display: Use unbound workqueues for deferred DM work"* — by moving the
deferred low-context IRQ work to a dedicated unbound workqueue. Nothing to write
here; wait for it to land in a kernel you can install.

Neither message indicates hardware failure. Both are log noise that disappears
once the respective patches reach your kernel.

## References

- CachyOS issue #1013 — DCN 3.2 counterpart report:
  https://github.com/CachyOS/linux-cachyos/issues/1013
- DCN 3.1 fix (upstream), *"drm/amd/display: fix compressed buffer config routine waiting time"*:
  https://lists.freedesktop.org/archives/amd-gfx/2026-June/146074.html
- *"drm/amd/display: Use unbound workqueues for deferred DM work"*:
  https://lists.freedesktop.org/archives/amd-gfx/2026-June/147787.html

## License

The patch modifies GPL-2.0-only kernel source and is provided under the same
terms as the Linux kernel (`GPL-2.0-only`).
