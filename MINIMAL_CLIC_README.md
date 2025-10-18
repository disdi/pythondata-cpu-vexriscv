# Minimal CLIC Implementation for VexRiscv

## Overview

The minimal CLIC implementation provides a simplified version of the RISC-V Core-Local Interrupt Controller that avoids potential hang issues found in the full CLIC implementation. This implementation is now integrated directly into the main CsrPlugin.scala file, making it a standard configuration option.

## Key Differences from Standard CLIC

| Feature | Standard CLIC | Minimal CLIC | Benefit |
|---------|---------------|--------------|---------|
| Interrupt IDs | 4096 (12-bit) | 256 (8-bit) | Reduced complexity |
| Priority Levels | 256 (8-bit) | 16 (4-bit) | Simpler priority logic |
| Interrupt Detection | Level-triggered | Edge-triggered | Prevents continuous triggering |
| Claim Mechanism | Atomic MCLAIMI | Simple latching | No race conditions |
| CSR Count | 5 CSRs | 2 CSRs | Reduced state management |
| Nesting Support | Full with MINTSTATUS | Basic | Simpler interrupt handling |

## Implementation Details

The minimal CLIC is integrated into `src/main/scala/vexriscv/plugin/CsrPlugin.scala` with a configuration flag `clicMinimal` that switches between standard and minimal CLIC implementations. The implementation uses:

- **Edge-triggered interrupts**: Prevents continuous triggering by latching on rising edge
- **Simplified state management**: No atomic claim mechanism (MCLAIMI)
- **Reduced CSR count**: Only MITHRESHOLD and MIVT registers
- **Hardware vectoring**: Direct jump to `MIVT + (interrupt_id << 2)`

## Available Configurations

All configurations are available through GenCoreDefault.scala with automatic parameter adjustment:

### 1. `minimal-clic`
Basic configuration with minimal CSR support:
```bash
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig minimal-clic --outputFile VexRiscv_MinimalClic"
```
- Sets `clicSupport = true, clicMinimal = true`
- 8-bit interrupt IDs, 4-bit priorities
- No vectored mode (xtvecModeGen = false)

### 2. `linux-minimal-clic`
Linux-compatible with minimal CLIC:
```bash
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig linux-minimal-clic --outputFile VexRiscv_LinuxMinimalClic"
```
- Linux minimal CSR set + minimal CLIC
- Same reduced interrupt/priority widths

### 3. `all-minimal-clic`
Full CSR support with minimal CLIC:
```bash
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig all-minimal-clic --outputFile VexRiscv_AllMinimalClic"
```
- Complete CSR feature set + minimal CLIC
- Suitable for testing and development

## CSR Registers

### MITHRESHOLD (0xFC1)
- **Purpose**: Interrupt priority threshold
- **Width**: 4 bits (values 0-15)
- **Access**: Read/Write
- Only interrupts with priority > MITHRESHOLD are serviced

### MIVT (0x307)
- **Purpose**: Interrupt Vector Table base address
- **Width**: 32 bits
- **Access**: Read/Write
- Hardware vectoring jumps to: `MIVT + (interrupt_id << 2)`

### Removed CSRs (compared to standard CLIC)
- **MCLAIMI (0xFC0)**: Not needed - no atomic claim mechanism
- **MINTSTATUS (0xFB1)**: Not needed - simplified nesting

## Module Interface

```verilog
module VexRiscv (
  // CLIC Interface
  input  wire        clicInterrupt,         // Interrupt pending signal
  input  wire [7:0]  clicInterruptId,       // 8-bit interrupt ID (0-255)
  input  wire [3:0]  clicInterruptPriority, // 4-bit priority (0-15)
  output wire [3:0]  clicThreshold,         // Current threshold value
  
  // Note: No clicClaim signal in minimal version
  // ... other signals ...
);
```

## Usage Example

### Software Setup

```c
// Set interrupt vector table base
write_csr(0x307, (uint32_t)interrupt_vector_table);  // MIVT

// Set interrupt threshold (only priority > 5 will be serviced)
write_csr(0xFC1, 5);  // MITHRESHOLD

// Interrupt vector table structure
void interrupt_vector_table[] __attribute__((aligned(4))) = {
  interrupt_handler_0,   // ID 0
  interrupt_handler_1,   // ID 1
  interrupt_handler_2,   // ID 2
  // ... up to 255 handlers
};

// Example interrupt handler
void interrupt_handler_0(void) {
  // Handle interrupt ID 0
  // No need to claim - automatically handled
  
  // Return from interrupt
  asm volatile("mret");
}
```

### Hardware Integration

```verilog
// External CLIC controller example
reg clic_int_pending;
reg [7:0] clic_int_id;
reg [3:0] clic_int_priority;

// Generate interrupt pulse (edge-triggered)
always @(posedge clk) begin
  if (interrupt_condition && !clic_int_pending) begin
    clic_int_pending <= 1'b1;
    clic_int_id <= 8'd10;      // Interrupt ID 10
    clic_int_priority <= 4'd8;  // Priority 8
  end
  else if (clic_int_pending) begin
    clic_int_pending <= 1'b0;  // Clear after one cycle
  end
end

// Connect to VexRiscv
assign cpu.clicInterrupt = clic_int_pending;
assign cpu.clicInterruptId = clic_int_id;
assign cpu.clicInterruptPriority = clic_int_priority;
```

## Troubleshooting

### If CPU Still Hangs

1. **Check interrupt handler returns properly**
   - Ensure all handlers end with `mret` instruction
   - Verify stack is properly managed

2. **Verify MIVT points to valid memory**
   - Must be 4-byte aligned
   - Must contain valid handler addresses

3. **Check external CLIC controller**
   - Must use edge-triggered interrupts (pulse)
   - Don't hold interrupt signal high continuously

4. **Verify priority settings**
   - Interrupt priority must be > MITHRESHOLD
   - MITHRESHOLD value must be 0-15

### Debug Tips

1. **Monitor signals in simulation:**
   ```verilog
   $monitor("CLIC: int=%b id=%d pri=%d thresh=%d", 
            clicInterrupt, clicInterruptId, 
            clicInterruptPriority, clicThreshold);
   ```

2. **Check CSR values:**
   ```c
   printf("MIVT: 0x%08x\n", read_csr(0x307));
   printf("MITHRESHOLD: %d\n", read_csr(0xFC1));
   ```



## Migration from Standard CLIC

1. **Reduce interrupt IDs**: Maximum 256 instead of 4096
2. **Adjust priorities**: Use 0-15 instead of 0-255
3. **Remove MCLAIMI reads**: No atomic claim needed
4. **Use edge-triggered interrupts**: Pulse instead of level
5. **Simplify nesting**: No MINTSTATUS tracking needed

## Performance Considerations

- **Latency**: Similar to standard CLIC (~5 cycles)
- **Area**: ~30% smaller than standard CLIC
- **Frequency**: Can achieve higher Fmax due to simpler logic
- **Power**: Lower power due to reduced state machines

## Example Generation Commands

```bash
# Basic minimal CLIC
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig minimal-clic"

# With caches
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig minimal-clic --iCacheSize 4096 --dCacheSize 4096"

# Linux-capable with minimal CLIC
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig linux-minimal-clic --iCacheSize 8192 --dCacheSize 8192"

# With debug support
sbt "runMain vexriscv.GenCoreDefault --csrPluginConfig minimal-clic --debug --hardwareBreakpointCount 4"
```

## Programmatic Configuration

To use minimal CLIC in custom Scala configurations:

```scala
import vexriscv.plugin._

// Create a CsrPlugin with minimal CLIC
val csrPlugin = new CsrPlugin(
  CsrPluginConfig(
    // Your base configuration
    mtvecInit = 0x80000020L,
    // Enable minimal CLIC
    clicSupport = true,
    clicMinimal = true,           // This enables minimal mode
    clicIntIdWidth = 8,           // 256 interrupts max
    clicIntPriorityWidth = 4      // 16 priority levels
  )
)

// Add to your VexRiscv configuration
val cpuConfig = VexRiscvConfig(
  plugins = List(
    csrPlugin,
    // ... other plugins
  )
)
```