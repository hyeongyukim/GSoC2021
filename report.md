
## Google Summer of Code 2021: Progression Report
---

#### Organisation: LLVM
#### Mentors: Juneyoung Lee, Nuno Lopes
#### Project Title: Fix miscompilation issues in LLVM IR using the ‘Freeze’ instruction
---

## Project Summary
Many of the bugs found are related to the notion of undef and poison values in LLVM.
The semantics of these values has been formalized recently, and existing optimizations are being fixed to match the semantics.
This project's goal is to fix such miscompilations with minimal-to-no performance regression.

Proposal link: <https://github.com/hyeongyukim/GSoC2021/blob/main/proposal.pdf>

## 1. Fix Miscompilation on InstCombine(GEP+icmp fold)
---
PR: <https://reviews.llvm.org/D99481>

This bug is one of the bugs in "Project Zero LLVM Bugs."
It is an optimization that converts a comparison based on the value of aggregate data structures to a comparison based on an index. 
```
if (some_array[index] == target_value)
=>
if (index == target_index)
```
And there is a problem because `index * ElementSize` can overflow.
When we assume ElementSize is 2, `some_array[index] == target_value` will be true not only when `index` is `target_index` but also `index` is `target_index + 0x80..00.`
And I solve this problem by adding 'and mask' to `index.`



## 2. Add an optimization to prevent poison from being propagated.
---
PR: <https://reviews.llvm.org/D105392>

It is a patch that prevents poison from propagating by changing the location of the freeze.
I observed that an additional assembly was inserted while solving the miscompilation problem of SimplyCFG(D104569).
This means that existing optimization is being disturbed by the newly added freeze.
To solve the compilation problem without performance degradation, I add an optimization pass that pushes the freeze until the below conditions are met.
- One-use
- Does not produce poison
- Has all but one guaranteed-non-poison operand




## 3. Fix SimplifyCFG optimization to be undef/poison safe
---
PR: <https://reviews.llvm.org/D104569>

`SimplyBranchOnICmpChain` performs optimization that changes ICmp chain to br and switch as follows.
```
	%A = icmp ne i32 %mode, 0
	%B = icmp ne i32 %mode, 51
	%C = select i1 %A, i1 %B, i1 false
	%D = select i1 %C, i1 %Cond, i1 false
	br i1 %D, label %T, label %F
=>
	br i1 %Cond, label %switch.early.test, label %F
switch.early.test:
	switch i32 %mode, label %T [
		i32 51, label %F
		i32 0, label %F
	]
```

However, this optimization has a problem: branching-on-poison (UB) occurs when `%mode` is 51 and the `%Cond` is poison.
Since the problem occurs when `%Cond` is poison, I can solve this problem by adding freeze instruction before br.

The incorrectness and correctness of this issue can be found in the link below.  
incorrectness: <https://alive2.llvm.org/ce/z/BWScX>  
correctness: <https://alive2.llvm.org/ce/z/x9x4oY>

## 4. [InstCombine] Add freezeAllUsesOfArgument to visitFreeze 
---
PR: <https://reviews.llvm.org/D106233>


## 5. Modify clang to turn on enable_noundef_analysis flag by default.
PR: <https://reviews.llvm.org/D105169>  
PR: <https://reviews.llvm.org/D108453>