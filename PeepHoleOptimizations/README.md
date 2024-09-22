
# Peephole Optimizations

- Peephole optimizations are a category of local code optimizations.
- The principal is very simple.
    - The optimizer analyzes sequence of instructions.
    - Only code thatis within a small windows of instructions
      is analyzed each time.
    - this window slides over the code.
    - once patterns are discoverd inside this window,
      optimizations are applied.

- Some memory access patterns are clearly redundent:

    load R0, m
    store m, RO

- Patterns like this can be easily elemenated by a peephole optimizer.

- Some branches can be rewritten into faster code. For isntance.

```asm
    if debug == 1 goto L1              if debug != 1 goto L2
    goto L2                            L1: ...
    L1: ...                            L2: ...
    L2: ...
```

```asm
goto L1                     goto L2             goto L2
....                        .....               .....

L1: goto L2                 L1: goto L2
```

- We can eliminate the second jump, provided that we know that there
  is not jump to L1 int the rest of the program, and L1 is preceded by
  an unconditional jump.
  - Why?
- Notice that this peephole optimization requires some previous info gathereing:
  we must know which instructions are targets of jumps or fall-throughs.

### Jumps to Jumps

- How could we optimize this code sequence?
    - under which assumptions is your optimization valid?

```asm
goto L1                         if a < b goto L2
.....                           goto L3
                                ....
L1 : if a<b goto L2             L3:
L3:
```

- In order to apply this optimization, we need to make sure that:
    - There is no other jump to L1
    - L1 is preceded by an unconditional goto.

- This optimization crosses the boundries of basic blocks,
  but it is still perfomed on a small sliding window.

### Strength Reduction

- Instead of performing reduction in strength at the DAG level, many
  compilers do it via a peephole optimizer.
    - This optimizer allow us to replace sequence such as 
        x*4 => x << 2


