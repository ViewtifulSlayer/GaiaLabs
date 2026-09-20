# 3. Addressing System & Memory Management

TSAL v3 provides a sophisticated addressing system that distinguishes between different SNES addressing modes and enables polymorphic instructions. **Crucially, platform addresses and game addresses are completely separate and never overlap.**

## Platform vs Game Address Separation

```typescript
// Platform addresses (65c816/platform-addresses.ts)
// - Defined by the CPU architecture
// - Never include Direct Page (DC) addresses
// - Fixed locations that don't change between games
export const platformAddresses = {
  // PPU registers (absolute addresses)
  ppu: {
    INIDISP: abs(0x2100),    // Display control
    BGMODE: abs(0x2105),     // Background mode
    BG1SC: abs(0x2107),      // BG1 tilemap
    OAMDATA: abs(0x2104),    // OAM data port
  },
  
  // APU registers (absolute addresses)
  apu: {
    APUIO0: abs(0x2140),     // APU I/O port 0
    APUIO1: abs(0x2141),     // APU I/O port 1
  },
  
  // CPU registers (absolute addresses)
  cpu: {
    RDNMI: abs(0x4210),      // NMI status
    TIMEUP: abs(0x4211),     // IRQ status
  }
} as const;

// Game addresses (iog_memory.ts)
// - Defined by the specific game
// - Include Direct Page (DC) addresses for game state
// - Can change between different ROM versions
export const gameAddresses = {
  // Direct page addresses (DC register offsets)
  directPage: {
    actor_flags: dc(0x10),       // Actor state flags
    temp_byte_1: dc(0x00),       // Temporary storage
    temp_byte_2: dc(0x01),       // Temporary storage
    dma_source: dc(0x04),        // DMA source address
  },
  
  // Absolute addresses (within game's memory space)
  absolute: {
    joypad_mask_std: abs(0x065A), // Joypad state
    player_health: abs(0x09A2),   // Player health
    current_scene: abs(0x0644),   // Current scene ID
  }
} as const;
```

## Address Types

```typescript
// Core address types for SNES
interface DirectPageAddress {
  type: 'direct';
  offset: number;  // 0x00-0xFF (DC register offset)
}

interface AbsoluteAddress {
  type: 'absolute';
  address: number;  // 0x0000-0xFFFF (within current bank)
}

interface LongAddress {
  type: 'long';
  bank: number;     // 0x00-0xFF
  address: number;  // 0x0000-0xFFFF
}

type Address = DirectPageAddress | AbsoluteAddress | LongAddress;

// Factory functions
export const dc = (offset: number): DirectPageAddress => ({
  type: 'direct',
  offset: offset & 0xFF
});

export const abs = (address: number): AbsoluteAddress => ({
  type: 'absolute', 
  address: address & 0xFFFF
});

export const long = (bank: number, address: number): LongAddress => ({
  type: 'long',
  bank: bank & 0xFF,
  address: address & 0xFFFF
});
```

## Platform Schema vs. Game Schema

A critical concept in TSAL is the separation between the **Platform Schema** and the **Game Schema**. The Platform Schema defines the hardware capabilities of the target system (like the SNES), while the Game Schema defines how a specific game organizes its data on that platform.

| Feature         | Platform Schema                                                               | Game Schema                                                                        |
|-----------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| **Scope**       | An entire hardware platform (e.g., SNES, NES).                                | A single game or ROM (e.g., *Illusion of Gaia*).                                  |
| **Purpose**     | Defines the **hardware's capabilities**, memory layout, and core data types.    | Defines how **game-specific data** is structured and organized within the ROM.       |
| **Defines**     | CPU instructions, PPU/APU registers, memory map, system types (`Byte`, `Word`). | Game-specific structs (`h_actor`), item tables, file locations, data blocks.     |
| **Analogy**     | The C++ language standard and compiler.                                       | The class definitions and data structures for a specific application.            |
| **Relationship**| Foundational. It is consumed by the Game Schema.                              | Builds upon the Platform Schema. Game structs use types defined at the platform level. |

---

## The Namespaced Instruction Set & `mem` Object

TSAL provides a type-safe and highly discoverable way to work with CPU instructions and memory. Instead of generic functions, each instruction has a **namespace**, and each supported addressing mode is a method on that namespace.

This is combined with a structured `mem` object for game-specific addresses, which prevents the use of ambiguous "magic numbers."

// Good: Readable, type-safe, and self-documenting.
import { mem, platform } from 'game'; // Game-specific memory and platform registers

const initializePPU = code(ctx => ctx.emit([
  LDA.imm(byte(0x0F)),
  STA.abs(platform.ppu.INIDISP),  // STA $2100
  LDA.imm(byte(0x09)),
  STA.abs(platform.ppu.BGMODE),   // STA $2105
]));

const updatePlayer = code(ctx => ctx.emit([
  LDA.dp(mem.dp.actor_flags),      // LDA $10 (direct page)
  AND.imm(byte(0xFE)),
  STA.dp(mem.dp.actor_flags),       // STA $10 (direct page)
  
  LDA.abs(mem.abs.player_health),   // LDA $09A2 (absolute)
  CMP.imm(byte(100)),
  BCC(playerLowHealth),
]));

## Evolution of Polymorphic Instructions

The namespaced approach is the modern, type-safe evolution of the original polymorphic instruction concept. Where previously a single function would inspect an `Address` object at runtime, the developer now makes the addressing mode explicit at compile time.

**Old approach (runtime polymorphism):**
`TRB(gameAddresses.directPage.actor_flags);`

**New approach (compile-time polymorphism):**
`TRB.dp(mem.dp.actor_flags);`

This shift provides two major advantages:
1.  **Compile-Time Safety**: It's impossible to use an addressing mode that an instruction doesn't support (e.g., `BRA.imm` would be a compile error).
2.  **IDE Discoverability**: Typing `TRB.` instantly shows a list of all valid addressing modes, making the system easier to learn and use. 