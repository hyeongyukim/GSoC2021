
## Google Summer of Code 2021: Final Report

#### Organisation: The LLVM Compiler Infrastructure
#### Mentors: Juneyoung Lee, Nuno Lopes
#### Project Title: Fix miscompilation issues in LLVM IR using the ‘Freeze’ instruction

---

## Project Summary
Many of the bugs found are related to the notion of undef and poison values in LLVM.
The semantics of these values has been formalized recently, and existing optimizations are being fixed to match the semantics.
This project's goal is to fix such miscompilations with minimal-to-no performance regression.

Proposal link: <https://github.com/hyeongyukim/GSoC2021/blob/main/proposal.pdf>

## 1. Fix Miscompilation on InstCombine(GEP+icmp fold)
Status: Landed
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
Status: Landed
PR: <https://reviews.llvm.org/D105392>

It is a patch that prevents poison from propagating by changing the location of the freeze.
I observed that an additional assembly was inserted while solving the miscompilation problem of SimplyCFG(D104569).
This means that existing optimization is being disturbed by the newly added freeze.
To solve the compilation problem without performance regression, I add an optimization pass that pushes the freeze until the below conditions are met.
- One-use
- Does not produce poison
- Has all but one guaranteed-non-poison operand



## 3. Fix SimplifyCFG optimization to be undef/poison safe
Status: Landed
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


## 4. Fix introduction of UB when hoisted condition may be undef or poison
Status: Working in progress
PR: <https://reviews.llvm.org/D106041>

Recently the semantics of LLVM IR's value was formalized recently, and existing optimizations were being fixed to match the semantics.
One representative example is loop unswitch.

```
while (C1) {
  if (C2) { A }
  else   { B }
}
Into:

if (C2) {
  while (C1) { A }
} else {
  while (C1) { B }
}
```

LoopUnswitch transforms the code if `C2` is a loop-invariant expression and hoisting the conditional branch is beneficial.
However, this transform isn't safe due to existence of UB.
If C1 is false and C2 is poison, the original probram is well defined. 
But after the transformation  branching on poison is executed regardless of the value of `C1` which raise UB.

And we can solve this kind of branch-on-undef problem by adding a freeze instruction to `C2`.
The freeze corrects LoopUnswitch, but it causes big regressions in some benchmarks.
So currently LoopUnswitch is still unfixed, and new real bugs related to this are still being reported(<https://bugs.llvm.org/show_bug.cgi?id=51043>)

In this patch, I am fixing LoopUnswitch with freeze with minimal-to-no performance regression.
LoopUnswitch itself needs improvement, but other optimizations like InstCombine and SimplifyCFG also need to improve.
This section deals only with LoopUnswitch and other optimizations will be covered in other sections.

While checking the performance regression caused by newly introudced freeze, I found the following issue:
```
  Cond.fr = freeze(Cond)
  br Cond.fr, BB_True, BB_False
BB_True:
  use(Cond.fr)			; Cond.fr is not replaced to true
  ...
BB_False:
  use(Cond.fr)			; Cond.fr is not repalced to false
  ...
```

In Original LoopUnswitch, we check the use of invariants and replace them if they are dominated by LoopPH or ClonedPH.
But while fixing LoopUnswitch, I inserted freeze to Cond(which is invariant) and replaced all original uses of Cond to Cond.fr.
Not only Cond but also Cond.fr is invariant, the current implementation of LoopUnswich doesn't know that Cond.fr is also invariant.
Therefore, in this patch, I added code to deduce Cond.fr as a constant.

Several patches have been made, but performance regression still exists.
After `enable_noundef_analysis` flag turn on by default, performance is likely to improve significantly.
Therefore, further modifications will be made after #6 is landed.

## 5. Add `freezeDominatedUses` function at InstCombine
Status: Landed
PR: <https://reviews.llvm.org/D106233>

While fixing the miscompilation of LoopUnswitch(<https://reviews.llvm.org/D106041>), I introduced a freeze instruction to a branch condition.
But newly added freeze disturbed other optimizations such as SimplifyCFG.
Freeze is a no-op that does not cause performance regression, so I can reduce performance regression by changing other optimization passes freeze-aware.
A good example is this patch, which solves the problem of newly added freeze is blocking SimplyCFG.

```
...
BB1:
  cond.fr = freeze(cond)  	; newly added freeze while fixing LoopUnswitch
  br cond.fr, TrueBB, FalseBB

TrueBB:
  use(cond)					; If TrueBB is dominated by BB1, cond is always true.
							; After freeze is added, cond and cond.fr are considered as
							; different values, so the value of cond cannot be deduced to true.
...
```

Because it is a problem that occurs when using `cond` and `cond.fr` simultaneously, this can be solved by changing all uses of `cond` to `cond.fr`.
I implemented the function `freezeDominatedUses` in InstCombine, which changes all dominated uses to frozen values.

The correctness of the patch can be checked by Alive2.  
Links:  
<https://alive2.llvm.org/ce/z/WKSW66>  
<https://alive2.llvm.org/ce/z/vqiQLT>



## 6. Modify clang to turn on `enable_noundef_analysis` flag by default.
Status: Waiting for review
PR: <https://reviews.llvm.org/D105169>  
PR: <https://reviews.llvm.org/D108453>


Like above, this patch is intended to solve the performance regression that occurred while fixing the LoopUnswitch.
Some of the freezes were applied to function arguments, and it disturbed some optimizations like inlining.
If the argument has a `noundef` attribute, the freeze will not be added because we only add freeze if the value can be undef or poison.
We can add the noundef attribute to the function arguments by adding the `enable_noundef_args` flag to the compiler.
It is cumbersome to add this flag all-time, so I created a patch that turns `enable_noundef_analysis` by default.
More than 1000 test codes were affected by this patch that changed Clang's default flag.


## Future Work

Many improvements have been made, but LoopUnswitch has yet to be fixed.
To fix the LoopUnswitch, I will first try to land the patch that turn on `enable_noundef_analysis` as defualt and then re-analyze the performance regression caused by LoopUnswitch patch.


## Acknowledgements

I would like to appreciate all LLVM community member for their help during this project. 
Thanks for my mentors Juneyoung Lee and Nuno Lopes for patient guide through the entire process.
And I also thanks to all reviewers of my code.
