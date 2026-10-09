# FATRUG3

Fully automatic timing ([FAT](https://en.wikipedia.org/wiki/Fully_automatic_time)) gates for sprint
training, version 3. This is a from-scratch redesign of [fatrug2](https://github.com/bardolf/fatrug2).

- Two identical ESP32 units (start/finish) linked over ESP-NOW Long Range.
- Runner detection with a single-point LiDAR (Benewake TFmini-S / TF-Luna) instead of ultrasound.
- Clock synchronization between the units, so radio latency does not affect the result.
- Phone web UI served by the unit (SoftAP): live time, run history, settings.

See [doc/design.md](doc/design.md) for requirements, decisions and the roadmap.

## Status

Design phase, see the [roadmap](doc/design.md#10-roadmap).
