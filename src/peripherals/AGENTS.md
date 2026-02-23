# AGENTS.md — src/peripherals

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Hardware peripheral support — STM32, RPi GPIO, Arduino, serial communication.

## STRUCTURE
```
src/peripherals/
├── traits.rs           # Peripheral trait
├── mod.rs              # Factory + registration
├── serial.rs           # Serial communication
├── rpi.rs              # Raspberry Pi GPIO
├── arduino_flash.rs    # Arduino flashing
├── nucleo_flash.rs     # STM32 Nucleo flashing
├── uno_q_bridge.rs    # Arduino Uno bridge
├── uno_q_setup.rs      # Uno-Q setup tool
└── capabilities_tool.rs # Hardware capabilities tool
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add board | Implement `Peripheral` trait → register in `mod.rs` |
| Serial I/O | `serial.rs` — `SerialPort` wrapper |
| RPi GPIO | `rpi.rs` — GPIO pin control |
| Firmware flash | `*_flash.rs` — board-specific flashing |

## CONVENTIONS
- Peripheral impl: `struct FooPeripheral; impl Peripheral for FooPeripheral { ... }`
- Expose tools: `fn tools(&self) -> Vec<Box<dyn Tool>>`
- Error handling: Return `PeripheralError` with board context

## ANTI-PATTERNS
- **NEVER** flash without confirmation
- **NEVER** skip baud rate validation
- **NEVER** assume serial port exists
- **NEVER** block on serial reads without timeout

## RISK: MEDIUM
Hardware interaction. Changes require:
- Physical device testing
- Timeout validation
- Error propagation tests
