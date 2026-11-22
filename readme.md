ThundeaL TD80Pro and be30a3/txhd2 clones: HDMI input lag from ~150 ms down to ~20–40 ms, plus real 60fps. No flashing, ADB only

[readme ru](readme_ru.md)
Tested on firmware build 20251112154138.

Platform: Amlogic txhd2, board be30a3, Android 13 (SDK 33). The firmware is userdebug/test-keys — root via adb root, no bootloader unlock needed. Should apply to every clone of this ODM (Stellamore STM930/T950S family: ThundeaL, Tandil, WZATCO, etc.). Verify yours with adb shell getprop ro.board.platform (txhd2) and ro.product.board (be30a3).

⚠️ Read this first

Digital zoom and keystone cancel the whole result. Reset zoom to 100% (maximum picture) and keystone to 0, turn auto-keystone/camera auto-focus off. Why: without geometry correction the HDMI-in frame goes straight out via a hardware plane (sideband, GPU not involved). Keystone/zoom is arbitrary geometry the hardware video plane cannot do, so the vendor reroutes the picture through GPU composition: on a Mali-450 with 1 GB of RAM that adds 1–2 frames of lag, and the vendor deliberately halves this path to 30fps. Level the projector physically — feet/wedge.

How I measured

Phone camera at 240 fps, a millisecond stopwatch on the PC with the screen duplicated to the projector, both screens in one frame. A series of 10–15 shots, median. Stock on my unit: ~150 ms (a third-party review measured ~110 ms on this model).

Why stock is this slow — what I found inside

1. ~73 ms is a deliberate AV-sync video delay. The nodes hdmin_delay_duration/last_required_total_delay are one and the same variable. They are zero after reboot; the audio HAL writes "73" every time the HDMI input is opened.
2. The vendor flag /sys/class/dummy_venc/encp_projector_fps = 1 halves the output to 30fps — on a physically 60 Hz panel (LVDS 2040×1130 @ 138.312 MHz; the panel driver itself reports an honest 60.00 Hz). Half of the input frames were simply never shown.
3. Heavy processing (AI-PQ) is already off in stock, the input is 1080p60 RGB without HDCP — no conversions. What remains is vdin capture and scanout.

I couldn't find a ready recipe for this board anywhere — not on XDA, not on Reddit, not on GitHub. I had to dig it out myself via kernel sysfs counters and Amlogic GPL sources.

What you need before you start

On the projector, enable Developer options, network debugging and allow ADB. On the computer you'll run commands from (Windows in my case), unpack platform-tools, open a console in that folder, type adb connect <projector-ip>:5555, then adb devices — the projector should show up. If things go sideways, Google is your friend.

Quick stock diagnostics

adb shell "cat /sys/class/video/vframe_states | grep delayed"   → 68–72 ms
adb shell "cat /sys/class/dummy_venc/encp_projector_fps"        → 1
adb shell "cd /sys/module/aml_media/parameters; a=$(cat vsync_count); sleep 10; c=$(cat vsync_count); echo vsync_per_s=$(( (c-a)/10 ))"  → 30

One-shot run (for testing; a reboot reverts everything)

@echo off
adb root
adb shell "echo 0 > /sys/class/dummy_venc/encp_projector_fps"
adb shell "echo 0 > /sys/module/aml_media/parameters/use_low_latency"
adb shell "echo 1 > /sys/module/aml_media/parameters/pc_mode_en"
adb shell "echo 1 > /sys/module/aml_media/parameters/vdin_pc_mode"
adb shell "echo 1 > /sys/class/videoqueue/force_game_mode"
adb shell "echo 0 > /sys/class/video/hdmin_delay_duration"
adb shell "echo 0 > /sys/class/video/last_required_total_delay"
adb shell "echo 0 > /sys/class/video/enable_hdmi_delay_normal_check"
adb shell "echo 1 > /sys/module/aml_media/parameters/freerun_en"
adb shell "echo 1 > /sys/class/video/freerun_mode"
adb shell "echo 0 > /sys/class/dummy_venc/encp_projector_fps"
adb shell "cat /sys/class/video/vframe_states | grep delayed"

After it: delayed ≈ 44, flag 0, vsync_per_s=60.

Profiles

Profile	encp_projector_fps	use_low_latency	Lag	Note
Recommended	0	0	~40 ms (fill in your series median)	60fps, clean picture
Sport	0	1	~20 ms	horizontal tear mid-screen at 60fps
Stock	1	0	~150 ms	30fps, out of the box

Why "Sport" tears: use_low_latency pushes the frame out mid-scan without waiting for VSync. In the vendor's 30fps mode the flip line sat near the top edge and was invisible; at 60fps it lands mid-screen.

About audio. We remove the AV-sync delay, so if your sound comes from the projector it will run ahead of the picture: for movies put echo 73 back into both delay nodes, or reboot. My audio goes straight from the PC to the speakers, so it doesn't affect me.

Autostart (survives reboots)

Everything above is sysfs: a reboot resets it, and tvserver re-writes its 73 ms every time the HDMI input opens. So I installed a service: the heavy knobs are applied once, a loop holds the rest.

lowlat.sh:

#!/system/bin/sh
echo 1 > /sys/module/aml_media/parameters/pc_mode_en
echo 1 > /sys/module/aml_media/parameters/vdin_pc_mode
echo 1 > /sys/class/videoqueue/force_game_mode
echo 1 > /sys/module/aml_media/parameters/freerun_en
echo 1 > /sys/class/video/freerun_mode
echo 0 > /sys/class/dummy_venc/encp_projector_fps
while true; do
  echo 0 > /sys/class/dummy_venc/encp_projector_fps
  echo 0 > /sys/module/aml_media/parameters/use_low_latency
  echo 0 > /sys/class/video/hdmin_delay_duration
  echo 0 > /sys/class/video/last_required_total_delay
  echo 0 > /sys/class/video/enable_hdmi_delay_normal_check
  sleep 1
done

lowlat.rc:

service lowlat /system/bin/sh /data/local/tmp/lowlat.sh
    user root
    group root
    disabled

on property:sys.boot_completed=1
    start lowlat

Install:

adb root
adb shell "mount -o remount,rw /vendor"
adb push lowlat.sh /data/local/tmp/lowlat.sh
adb push lowlat.rc /vendor/etc/init/lowlat.rc
adb shell "sed -i 's/\r$//' /data/local/tmp/lowlat.sh /vendor/etc/init/lowlat.rc"
adb shell "chmod 755 /data/local/tmp/lowlat.sh; chmod 644 /vendor/etc/init/lowlat.rc"
adb reboot

Verify a minute after boot (HDMI input open): adb shell "pgrep -f lowlat" returns a PID; flag 0; delayed ≈ 44; vsync_per_s=60.

Pitfalls I hit (so you don't have to):

1. Files saved on Windows carry CRLF line endings — init silently fails to parse such an .rc. sed -i 's/\r$//' fixes it (already in the install block).
2. adb remount writes into an overlayfs that does not survive reboot — your file vanishes. Use a direct mount -o remount,rw /vendor instead (verity is disabled on this firmware).
3. Check the service with pgrep -f lowlat, not ps -A | grep lowlat — the latter shows the process name sh, not the script path.
4. After an OTA (A/B slot switch) the service is gone in the new slot — reinstall.

Undo the autostart: delete /vendor/etc/init/lowlat.rc and reboot.

Do not touch (tested — useless or dangerous)

• /sys/class/display/mode accepts only dummy_p/dummy_panel; anything else is silently ignored.
• hdmin_delay_min_ms/max_ms are statistics, rewritten on their own.
• pts_latency (18000) is DVB stream latency, unrelated to HDMI-in.
• /dev/mem, /dev/kmem, /proc/kcore are cut out of this kernel — no soft memory patching.
• Firmware from other be3xx boards will brick the device; no custom ROMs exist for be30a3/txhd2.

Risks

Nothing is flashed and nothing is uninstalled: all tuning is sysfs writes — a reboot returns full stock; the autostart is one added file in /vendor, removed by the undo step. The only irreversible risk stays where it always was: trying to flash other people's images.
