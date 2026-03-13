# QEMU Virtual Atheros AR9285 – Meson build integration
#
# SPDX-License-Identifier: GPL-2.0-or-later
#
# This file is meant to be placed at hw/net/ath9k/meson.build inside
# the QEMU source tree.  The parent hw/net/meson.build must include it
# via: subdir('ath9k')
#
# Phase 3: Added chardev dependency for virtual medium socket.
# The ath9k_medium_hub is a standalone tool built separately:
#   gcc -Wall -O2 -o ath9k_medium_hub ath9k_medium_hub.c

system_ss.add(when: 'CONFIG_ATH9K_VIRT', if_true: files(
  'ath9k_pci.c',
))
