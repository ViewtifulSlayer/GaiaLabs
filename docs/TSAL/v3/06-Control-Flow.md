# 6. Control Flow

TSAL handles control flow in two primary ways:
1.  **Runtime Control Flow**: Standard assembly branching instructions (`BNE`, `JSR`, `BEQ`, etc.) used within the `layout()` function to control the logic of the final program.
2.  **Generation-Time Control Flow**: Standard TypeScript `if/else` statements and loops used to conditionally generate different assembly code during the build process.

## Runtime Control Flow (Branching & Jumps)

This is the standard control flow you would find in any assembly language. TSAL provides functions for all the 65c816's branching and jump instructions. These functions require a `Label` as a target. A powerful feature of the `code()` block is the ability to create locally-scoped labels that don't pollute the module's namespace.

```typescript
const my_looping_logic = code(ctx => ctx.emit(
  // local labels are created by passing a function to emit()
  (self, main_loop, do_something, done) => [
    main_loop,
    LDA.dp(mem.dp.some_flag),
    CMP.imm(byte(5)),
    BNE(do_something),   // Branch if not equal
    JMP(done),           // Unconditional jump

    do_something,
    // ... code ...
    JSR(some_subroutine), // Jump to an exported subroutine
    BRA(main_loop),       // Branch always back to the local label

    done,
    RTL(),
  ]
));
```

## Generation-Time Control Flow (Conditional Generation)

Because TSAL code is executed by the TypeScript runtime to produce assembly, you can use standard JavaScript/TypeScript control flow structures (`if/else`, `switch`, `for`) to change what code gets generated. This is a powerful feature for managing different versions, builds, or platforms.

### Feature Flags

A common use case is to include or exclude blocks of code based on feature flags defined in a configuration file.

```typescript
// build-config.ts
export const FEATURES = {
  DEBUG_MODE: true,
  ENHANCED_AUDIO: false,
};

// module.ts
import { FEATURES } from './build-config';

const getDebugCode = (messageAddress: Address) => {
  if (!FEATURES.DEBUG_MODE) {
    // If not in debug mode, return an empty code block that emits nothing.
    return code(ctx => ctx.emit([]));
  }
  // Otherwise, return the code that prints the message.
  return code(ctx => ctx.emit([
    LDA.abs(messageAddress), // Assuming absolute for simplicity
    JSR(debugPrint),
  ]));
}

export const layout = code(ctx => ctx.emit([
    // ...
    getDebugCode(my_debug_string), // The result of this function is included
    // ...
]));
```

### High-Level Loop Structures

You can also wrap low-level branching logic into higher-level, composable functions to create more readable control structures that prevent common branching errors.

```typescript
// A function that generates a standard if/else block.
const ifEqual = (thenBlock: SnesLayoutable, elseBlock?: SnesLayoutable) => {
    return code(ctx => ctx.emit(
        (self, elseLabel, endLabel) => {
            if (elseBlock) {
                return [
                    BNE(elseLabel),  // If Not Equal, branch to the 'else' part
                    thenBlock,
                    BRA(endLabel),   // Unconditionally jump *over* the 'else' part
                    elseLabel,
                    elseBlock,
                    endLabel
                ];
            } else {
                return [
                    BNE(endLabel),   // If Not Equal, branch to the end
                    thenBlock,
                    endLabel
                ];
            }
        }
    ));
}

// Usage creates clean, declarative code.
const someLogic = code(ctx => ctx.emit([
    CMP.imm(byte(10)), // Compare some value
    ifEqual(
        code(ctx => ctx.emit([ STA.dp(mem.dp.isTen) ])),
        code(ctx => ctx.emit([ STA.dp(mem.dp.isNotTen) ]))
    ),
]));
``` 