fprintd provider snap (Ubuntu Core Desktop, Phase 4)
====================================================

The fat base shipped fprintd + libpam-fprintd; the thin desktop variant
bakes only the host side (pam_fprintd.so, /etc/pam.d/gdm-fingerprint,
the polkit action net.reactivated.fprint.device.policy, /var/lib/fprint
seeding). This snap restores the daemon side: fprintd 1.94.5 +
libfprint 1.95.1+tod1 staged from the resolute archive (same builds the
fat base shipped), as a strictly-confined system dbus-activated daemon.

Design (follows the accountsservice provider-snap pattern):

- slots: `fprintd-dbus` (builtin dbus interface, system bus,
  net.reactivated.Fprint — activation + name ownership) and `fprintd`
  (custom fork interface carrying the daemon recipe).
- plugs: `polkit` (action-prefix net.reactivated.fprint; the action XML
  itself stays baked in the variant — do NOT ship a second copy via
  meta/polkit, it would double-seed) and `fprintd-client` for the
  bundled CLI tools.
- apps: `fprintd` daemon (daemon: dbus, daemon-scope: system,
  activates-on fprintd-dbus) + fprintd-list/enroll/verify/delete
  clients.
- base: plain core26 — the daemon execs no host helpers. No
  localdev_swap_base dance needed in go-build.
- layouts: /var/lib/fprint -> $SNAP_COMMON (state; the host fstab bind
  stays as inert factory seeding — same reasoning as the accountsservice
  snap's /var/lib/AccountsService layout) and /etc/fprintd.conf ->
  $SNAP/etc/fprintd.conf (core26 has no fprintd.conf; snap-update-ns
  creates file mountpoints).

Build: ./go-build (needs an apt-cacher/proxy env from go-build-lib.sh).

Known limitation (flagged, NOT silently worked around): real reader
access requires udev device tagging for the snap device cgroup (the
udisks2 `udev-file` slot-attribute pattern). Upstream's
70-libfprint-2.rules uses RUN+=/ENV{} rules that snapd's udev parser
cannot use for tagging, so the interface carries the AppArmor device
rules but hardware access needs a snapd-tagging-compatible rules file
before a reader will actually work. Device-less operation (ListDevices
-> empty, graceful PAM skip) is unaffected.
