# e2HIL

**English** | [Français](README.fr.md)

Hardware-in-the-loop testing of embedded firmware **on the real boards**, by hand or in continuous integration.

> Status: specification (v0.1). Nothing to run yet. See the [specification](docs/SPEC.md) (French) and the
> [issues](https://github.com/laurentolive/e2HIL/issues).

- **Zero impact on the firmware**: the bench acts and measures from the outside (I/O box, probe, serial ports).
- **Same scenario shape as e2Sim**: `dut["BTN_POWER"].press(ms=300)`, `dut.run(s=1)`, `dut["PWM_MOTOR"].duty`,
  with the signal names of the e2Sim `board.yaml`. This is a shared convention, not a code dependency.
- **Ground truth from the bench**: frequency, duty cycle and latency are measured by an RP2040 I/O box, not by
  the firmware under test.
- **Built for CI**: `bench.yaml` (probes and ports by serial number), bench lock, cold power cycle, J-Link and
  Arduino flashing, serial logs attached to each test, pytest plugin, JUnit and Excel requirement reports.
- **Cheap**: a PC, off-the-shelf probes, a Raspberry Pi Pico, a USB hub with per-port power switching.

It completes the eSquareSoft tools: Polenta (requirements), e2Unit (unit tests), e2Sim (simulation),
eScope (observation). Reference project: [fan-demo](https://github.com/laurentolive/fan-demo).

## Layout (planned)

```
e2hil/      Python package: bench manager, flashing, scenario API, pytest plugin, reports, CLI
iobox/      firmware of the RP2040 I/O box (MIT)
docs/       specification, tutorials
```

## License

PC tool: PolyForm Noncommercial 1.0.0 ([LICENSE.md](LICENSE.md)), professional license for commercial use.
I/O box firmware: MIT ([iobox/LICENSE.md](iobox/LICENSE.md)).
