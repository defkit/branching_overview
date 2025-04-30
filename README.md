# branching

Not all programs containing branching can be solved in ACIR. 

## Rules
There are several rules:
1) There can only be one block that is terminated with `Return`;
2) Every ssa block must be terminated;
3) If block terminated with `IfElse`, `then` branch must not be the end of the branch

Due to these rules the following programs are invalid:


```mermaid
graph TD;
    b0-->|Then| b1;
    b0-->|Else| b2;
```

Because it violates either rule 1 or rule 2


```mermaid
graph TD;
    b0-->|Then| b2;
    b0-->|Else| b1;
    b1-->|Jmp| b2;
```
Because it viloates rule 3.

And for some reason, the following program is invalid too.
```mermaid
graph TD;
    b0-->|Then| b1;
    b0-->|Else| b2;
    b1-->|Then| b2;
    b1-->|Else| b3;
    b2-->|Jmp| b3;
```
It does not viloate rule 3, but invalid. (I couldn't find an explanation for this in the documentation)

## How to fuzz
Suppose the fuzzer provides blocks of instructions e.g. [Block([Add, Sub, Sub]), Block([Mul, Mul])..], 

Let's introduce new instructions for the fuzzer
1) InsertSimpleInstructionBlock(idx) -- adds instructions to `current_block_context` from stored `InstructionBlocks`
2) MergeBlocks(idx0, idx1) -- adds combination of `InstructionBlocks` into stored array of `InstructionBlocks`
3) InsertIfBlock(idx_then, idx_else) -- terminates current SSA block with `jmp_if_else`. Creates two new SSA blocks from chosen `InstructionBlocks`. Switches `current_block_context` to `then_branch`. Adds `else_branch` to the stack. If current SSA block is already terminated, skip. 
4) InsertJmpBlock(idx) -- terminates current SSA block with `jmp`. Creates new SSA block from chosen `InstructionBlocks`. Switches `current_block_context` to new created branch. If current SSA block is already terminated, skip.
5) TerminateBlockWithReturn -- Terminates `current_block_context` with `jmp`. Tries to take block from stack, if stack is empty, finish.
6) InsertReturnBlock(idx) -- Must be at least 1. Creates new SSA block from chosen `InstructionBlocks`, and stores Id of this block. If such a block already exists, skip.

After all this, we termiate all blocks from the stack and current block with `jmp` to `ReturnBlock`


## Example
let instruction_blocks = [Block0, Block1])]

let branching_instructions = [InsertReturnBlock(0), InsertSimpleInstructionBlock(0), InsertIfBlock(0, 1), TerminateBlockWithReturn, InsertSimpleInstructionBlock(1), InsertJmpBlock(0), InsertIfBlock(0,0)]

after `InsertReturnBlcok(0)`

after `InsertSimpleInstructionBlock(0)`
```mermaid
graph TD;
    b0
    b6[Block0 id=6 RETURN BLOCK]
```

after `InsertIfBlock(0, 1)`, current block context = 1, stack = [2]
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 id=2]
    b6[Block0 id=6 RETURN BLOCK]
```

after `TerminateBlockWithReturn`, current block context = 2, stack = []
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 id=2]
    b1-->|Jmp| b6
    b6[Block0 id=6 RETURN BLOCK]
```


after `InsertSimpleInstructionBlock(1)`, current block context = 2, stack = []
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 + Block1 id=2]
    b1-->|Jmp| b6
    b6[Block0 id=6 RETURN BLOCK]
```

after `InsertJmpBlock(0)`, current block context = 3, stack = []
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 + Block1 id=2]
    b2-->|Jmp| b3[Block0 id=3]
    b1-->|Jmp| b6
    b6[Block0 id=6 RETURN BLOCK]
```

after `InsertIfBlock(0, 0)` current block context = 4, stack = [5]
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 + Block1 id=2]
    b2-->|Jmp| b3[Block0 id=3]
    b3-->|Then| b4[Block0 id=4]
    b3-->|Else| b5[Block0 id=5]
    b1-->|Jmp| b6
    b6[Block0 id=6 RETURN BLOCK]
```

After all the instructions have finished, we terminate all notterminated blocks.
The resulting program will look like this:
```mermaid
graph TD;
    b0[Block0 id=0]-->|Then| b1[Block0 id=1]
    b0[Block0 id=0]-->|Else| b2[Block1 + Block1 id=2]
    b2-->|Jmp| b3[Block0 id=3]
    b3-->|Then| b4[Block0 id=4]
    b3-->|Else| b5[Block0 id=5]
    b4-->|Jmp| b6
    b5-->|Jmp| b6
    b1-->|Jmp| b6
    b6[Block0 id=6 RETURN BLOCK]
```
