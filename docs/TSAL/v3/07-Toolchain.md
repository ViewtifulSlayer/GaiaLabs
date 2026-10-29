# 7. Toolchain & ROM Generation

TSAL v3 provides an intelligent module discovery system and ROM generation pipeline that automatically handles dependencies and bank placement.

## Module Discovery System

The TSAL toolchain discovers modules by scanning the project for files that export a `layout` function and an optional `moduleInfo` object. This process does not require a special compiler plugin; it can be done with a simple script that imports all TypeScript files and inspects their exports.

```typescript
// ROM generator that discovers and processes TSAL modules
class TSALROMGenerator {
  private modules: TSALModule[] = [];
  
  // Type-safe module discovery
  discoverModules(projectPath: string): TSALModule[] {
    const discoveredModules: TSALModule[] = [];
    
    // Scan TypeScript files for TSAL modules
    const files = this.scanTypeScriptFiles(projectPath);
    
    for (const file of files) {
      const module = this.loadModule(file);
      if (isTSALModule(module)) {
        discoveredModules.push(module);
      }
    }
    
    return discoveredModules;
  }
  
  // Generate ROM from discovered modules
  generateROM(modules: TSALModule[]): Buffer {
    const romBuilder = new ROMBuilder();
    
    // Sort modules by dependencies
    const sortedModules = this.resolveDependencies(modules);
    
    // Generate assembly from each module by executing its layout(ctx)
    // function in a GenerationContext.
    for (const module of sortedModules) {
      // Create a new context for each module
      const context = new GenerationContext();
      
      // Execute layout function to generate assembly
      module.layout(context);
      
      // Collect generated assembly from the context
      const assembly = context.getGeneratedAssembly();
      romBuilder.addAssembly(assembly, module.moduleInfo);
    }
    
    return romBuilder.build();
  }
  
  private resolveDependencies(modules: TSALModule[]): TSALModule[] {
    // Topological sort based on moduleInfo.dependencies
    // Implementation details...
  }
}
```

## Execution Pipeline

The core of the toolchain is an execution pipeline that turns TypeScript calls into machine code.

1.  **Execute**: The `layout()` function of each module is executed. Instead of running on the CPU, TSAL functions like `LDA(imm(5))` generate an intermediate representation of the instruction.
2.  **Optimize**: The stream of generated instructions can be optimized. For example, unreachable code after a `JMP` or `RTL` can be eliminated.
3.  **Generate**: The optimized instruction stream is converted into its final binary machine code.
4.  **Link**: The machine code from all modules is placed into the correct banks, and labels are resolved to their final addresses to produce the ROM file.

### Execution-Time Optimization

Because code generation happens in an executable TypeScript environment, the toolchain can perform optimizations that a traditional assembler cannot.

#### Dead Code Elimination
```typescript
// TypeScript Code
const exampleLayout: LayoutFunction = (ctx) => {
  ctx.LDA(imm(42));
  ctx.RTS();
  // The following lines are still valid TS, but a smart context
  // can detect they are unreachable and generate no code for them.
  ctx.LDA(imm(100));
  ctx.STA(gameAddresses.directPage.temp_byte_1);
};

// ...so no code is generated for them.
// Generated Assembly:
// LDA #$2A
// RTS
```

#### Constant Folding
```typescript
// TypeScript Code
const PLAYER_SPEED = 3;
const JUMP_HEIGHT = 16;

const layout: LayoutFunction = (ctx) => {
    ctx.LDA(imm(PLAYER_SPEED + JUMP_HEIGHT));  // The addition is done at generation time
};

// Generated Assembly:
// LDA #$13    ; 3 + 16 = 19, which is 0x13
```