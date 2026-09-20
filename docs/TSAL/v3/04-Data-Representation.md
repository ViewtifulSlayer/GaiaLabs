# 4. Data Representation

TSAL v3 provides sophisticated string and struct systems with database-driven validation and template literal support.

## Advanced Command-Embedded String Validation

String types in TSAL provide **sophisticated validation** for embedded commands within game text, going far beyond simple character map validation to ensure command syntax, parameter types, and control flow are all correct.

### Template Literal Approach

TSAL embraces **JavaScript template literals** (backticks) for natural multiline string support, eliminating the need for complex line continuation parsing:

```typescript
// Real game string using template literals - naturally supports multiline
const dialogue = WideString`
  [TPL:A][TPL:3]Erik: The sun is [LU2:95]bright. 
  I [LU1:BB]noticed [LU1:D7]before.
  [PAL:0][END]
`;
```

### The Challenge: Complex Embedded Commands

Game strings contain sophisticated mini-programs with typed parameters:

```typescript
// Each command has specific parameter requirements:
[TPL:A]     // TPL command with Byte parameter
[LU2:95]    // LU2 lookup with Byte index  
[LU1:BB]    // LU1 lookup with Byte index
[PAL:0]     // PAL palette change with Byte parameter
[END]       // END terminator with no parameters (halt: true)
```

## String Type System

```typescript
// String types are generated from database definitions
interface StringTypeDefinition {
  delimiter: string;
  terminator: number;
  characterMap: readonly (string | null)[];
  commands: CommandDefinitions;
  greedyTerminator?: boolean;
  maxLength?: number;
}
```

### Database-Driven Command Definitions

```typescript
// Generated from stringCommands.json
interface WideStringCommands {
  END: { params: []; halt: true };
  TPL: { params: [byte: number]; halt: false };
  PAL: { params: [byte: number]; halt: false };
  LU1: { params: [byte: number]; halt: false };
  // ... and so on
}
```

### TypeScript Template Literal Parser

TSAL uses **advanced TypeScript types** to parse and validate embedded commands at compile time.

For a detailed technical breakdown of the types involved, see **[Appendix A: String Parser Internals](./Appendix-A-String-Parser.md)**.

### Type-Safe String Creation

```typescript
// ✅ Valid - Template literals support natural multiline strings
const validDialogue = WideString`
  [TPL:A][TPL:3]Erik: The sun is [LU2:95]bright. 
  I [LU1:BB]noticed [LU1:D7]before.
  [PAL:0][END]
`;

// ❌ TypeScript Error: Unknown command
const invalidCommand = WideString`
  [INVALID:123]Hello World[END]
`;

// ❌ TypeScript Error: Wrong parameter count
const wrongParams = WideString`
  [TPL:A:B]Hello[END]  // TPL expects 1 param, got 2
`;
```

---

## Struct System

```typescript
import { LayoutFunction } from 'tsal';

// Struct constructors (generated from database) - execute to generate data
export const h_na4B_spirit = h_actor(imm(0x32), imm(0x00), imm(0x30));

// Struct usage in assembly
export const layout: LayoutFunction = (ctx) => {
  ctx(h_na4B_spirit);
  
  // Store struct data to memory
  ctx.LDA(imm(playerActor.id));
  ctx.STA(gameAddresses.directPage.player_data.id);
  
  ctx.LDA(imm(playerActor.type));  
  ctx.STA(gameAddresses.directPage.player_data.type);
  
  // Process meta commands
  const ppuSetup = meta.ppu(imm(0x0F));
  ctx.LDA(imm(ppuSetup.value));
  ctx.STA(platformAddresses.ppu.INIDISP);
};
```

### Polymorphic Structs with Discriminators

```typescript
// Meta command system (discriminator-based)
type MetaCommand = 
  | PPUCommand      // discriminator: 2
  | BitmapCommand   // discriminator: 3
  | PaletteCommand; // discriminator: 4

// Dynamic struct creation
meta(imm(2), imm(0x80)); // Auto-resolves to PPUCommand

// Specific constructors for type safety
meta.ppu(imm(0x80)); // Explicit PPU command
```

### Runtime Struct Matching

```typescript
// Pattern matching for dynamic struct processing
const processMetaCommand = (ctx: ExecutionContext, commandAddr: Address) => {
  ctx.LDA(commandAddr);  // Load discriminator
  
  // TypeScript switch with type narrowing
  const discriminator = loadByte(ctx, commandAddr);
  switch (discriminator.value) {
    case 2: {
      // TypeScript knows this is PPUCommand
      const cmd = meta.ppu(loadByte(ctx, commandAddr.offset(1)));
      return processPPU(ctx, cmd);
    }
    case 3: {
      // TypeScript knows this is BitmapCommand
      const cmd = meta.bitmap(/* ... */);
      return processBitmap(ctx, cmd);
    }
  }
};
``` 