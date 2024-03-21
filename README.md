#### SIMD

**S**ingle **I**nstruction **M**ultiple **D**ata: a single instruction operates on multiple data elements. 

###### Autovectorization

- Compilers can automatically vectorize code using SIMD instructions, as shown in the previous examples.
- Autovectorization is an active area of research and development in compilers
  with many new papers every year.
- Compilers can inform the programmer about the vectorization process using flags like `-fopt-info-vec` in GCC, [LLVM offers similar diagnostics](https://llvm.org/docs/Vectorizers.html#diagnostics). 
- Specialized languages and compilers such as [ISPC](https://ispc.github.io/) and [Halide](https://halide-lang.org/) are designed to make it easier to write vectorized code.

###### Example: Copying an array

```
cat copy.c
void copy(long *restrict a, long *restrict b, unsigned long n) {
    for (unsigned long i = 0ul; i < n; i++) {
        a[i] = b[i];
    }
}
```


Without SIMD:
```
gcc -O3 -fno-tree-loop-distribute-patterns -fno-tree-vectorize -S  copy.c  -o /dev/stdout
# -fno-tree-loop-distribute-patterns: prevents replacing the loop with a call to memcpy
# -fno-tree-vectorize: prevents vectorization

# Loop body:
 # Each iteration of the loop copies 8 bytes from b[i] to a[i]
 .L3:
	movq	(%rsi,%rax,8), %rcx      # load 8 bytes from b[i] into rcx
	movq	%rcx, (%rdi,%rax,8)      # store 8 bytes from rcx into a[i]
	addq	$1, %rax
	cmpq	%rax, %rdx
	jne	.L3
```

With SIMD:
```
gcc -O3 -fno-tree-loop-distribute-patterns  -S -o /dev/stdout copy.c


# Loop body:
 # Each iteration of the loop copies 16 bytes from b[i] to a[i]
 .L4:
 	movdqu	(%rsi,%rax), %xmm0     # load 16 bytes from b[i] into xmm0
 	movups	%xmm0, (%rdi,%rax)     # store 16 bytes from xmm0 into a[i]
 	addq	$16, %rax             
 	cmpq	%rcx, %rax
 	jne	.L4
```

Wider SIMD registers can be used to copy more data per iteration:
```
gcc -O3 -fno-tree-loop-distribute-patterns -mavx512f  -S -o /dev/stdout copy.c
# -mavx512f: enable AVX-512 instructions
# Loop body:
 # Each iteration of the loop copies 64 bytes from b[i] to a[i]
 .L4:
 	vmovdqu64	(%rsi,%rax), %zmm0      # load 64 bytes from b[i] into zmm0
 	vmovdqu64	%zmm0, (%rdi,%rax)      # store 64 bytes from zmm0 into a[i]
 	addq	$64, %rax
 	cmpq	%rax, %rcx
 	jne	.L4
```

We can also vectorize manually using [intrinsics](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html):
```
cat copy_intrinsics.c

#include <immintrin.h>

void copy(long *restrict a, long *restrict b, unsigned long n) {

    // Only works if n is a multiple of 8
    for (unsigned long i = 0ul; i < n; i+=8ul) {
        __m512i v = _mm512_loadu_si512(&b[i]);
        _mm512_storeu_si512(&a[i], v);
    }
}
```


```
gcc -O3 -fno-tree-loop-distribute-patterns -mavx512f  -S -o /dev/stdout copy_intrinsics.c
Loop body:
 # Each iteration of the loop copies 64 bytes from b[i] to a[i]
 .L3:
 	vmovdqu64	(%rsi,%rax,8), %zmm0    # load 64 bytes from b[i] into zmm0
 	vmovdqu64	%zmm0, (%rdi,%rax,8)    # store 64 bytes from zmm0 into a[i]
 	addq	$8, %rax
 	cmpq	%rdx, %rax
 	jb	.L3
```

###### Example: Vectorized control flow

```
cat control_flow.c

void copy(long *restrict a, long *restrict b, unsigned long n) {
    for (unsigned long i = 0ul; i < n; i++) {
        if (a[i] > 0)
            b[i] = a[i];
        else
            b[i] = a[i] * 2;
    }
}
```


```
gcc -O3 -fno-tree-loop-distribute-patterns -mavx512f  -S -o /dev/stdout control_flow.c

# Loop body:
.L4:
	vmovdqu64	(%rdi,%rax), %zmm3     # load 64 bytes from a[i] into zmm3
	vpsllq	$1, (%rdi,%rax), %zmm0     # load 64 bytes from a[i] and shift left by 1 (multiply by 2) into zmm0
	vpcmpq	$6, %zmm1, %zmm3, %k1      # compare all elementf of zmm3 with 0 (stored in zmm1) and set mask k1 ($6 is the comparison type)
	vmovdqa64	%zmm3, %zmm0{%k1}      # copy elements from zmm3 to zmm0 using mask k1
	vmovdqu64	%zmm0, (%rsi,%rax)     # store 64 bytes from zmm0 into b[i]
	addq	$64, %rax
	cmpq	%rax, %rcx
	jne	.L4
```



#### The goal of this project

Autovectorization is __difficult__. 
Vectorizing a piece of code may require significant changes to the code (e.g., [All you need is superword-level parallelism: systematic control-flow vectorization with SLP](https://dl.acm.org/doi/10.1145/3519939.3523701).
The main goal of this project is not to develop an autovectorizer, but to develop a method for identifying missed opportunities for vectorization in existing code.  That is, given an existing binary, we want to identify loops (or even straighline) that could be vectorized but are not.


##### Approach

(This is a suggestion, feel free to deviate from it.)


The detection of missed opportunities for vectorization can be done in several ways. One approach is to use a dynamic analysis that traces an execution and identifies vectorizable code.

There are many dynamic analysis frameworks, for example:
- [Intel Pin](https://software.intel.com/sites/landingpage/pintool/docs/98830/Pin/doc/html/index.html)
- [DynamoRIO](https://dynamorio.org/)
- [angr](https://angr.io/)


###### A dataflow-based approach

Given a trace of execution (instructions and memory accesses), we can use a dataflow analysis to identify "parallel" computations.
For example in the following code (from the first example that copies an array)
```
 .L3:
	movq	(%rsi,%rax,8), %rcx      # load 8 bytes from b[i] into rcx
	movq	%rcx, (%rdi,%rax,8)      # store 8 bytes from rcx into a[i]
	addq	$1, %rax
	cmpq	%rax, %rdx
	jne	.L3
```

We can identify that the load and store instructions are independent and can be executed in parallel (i.e., they can be vectorized) by looking at the dataflow:
```
Iteration 1:
load a[i+0] -> rcx -> store a[i+0]

Iteration 2:
load a[i+1] -> rcx -> store a[i+1]

Iteration 3:
load a[i+2] -> rcx -> store a[i+2]
...
```
All the loads and stores are "parallel". Similarly, in a slightly more complex example:
```
 .L3:
	movq	(%rsi,%rax,8), %rcx      # load 8 bytes from array1[i] into rcx
	movq	(%rbx,%rax,8), %r10      # load 8 bytes from array2[i] into rdx
	addq    %r10, %rdx               # add the two values
	movq	%rdx, (%rdi,%rax,8)      # store 8 bytes from rdx into array3[i]
	addq	$1, %rax
	cmpq	%rax, %rdx
	jne	.L3
```
Dataflow:
```
Iteration 1:
load array1[i+0] -> rcx 
                       \    
                        -> add r10 and rcx -> store arrya3[i+0] 
                       / 
load array2[i+0] -> r10 

Iteration 2:
load array1[i+1] -> rcx 
                       \    
                        -> add r10 and rcx -> store arrya3[i+1] 
                       / 
load array2[i+1] -> r10 


Iteration 3:
load array1[i+2] -> rcx 
                       \    
                        -> add r10 and rcx -> store arrya3[i+2] 
                       / 
load array2[i+2] -> r10 
...
```

All the necessary data to build the dataflow graph can be obtained from the trace (generated by the dynamic analysis tool).


##### Basic idea

Identify parallel dataflow in the trace and match it to known vectorization instruction/patterns.
In the above examples it is easy to see that the dataflow can be vectorized using SIMD instructions, all the loads and stores are independent and on concecutive addresses.

Challenges:
- The dataflow graph can be very large. Hint: if this becomes a problem, you
  could first identify loops and only trace these.
- Control flow: some loops iterations might behave differently due to divergent
  control flow (like in the `control.flow.c` example ). Your approach should be
  able to handle this.
- The number of vector instructions and patterns is large. You might want to
  start with a small subset of instructions and patterns and then expand it.
- How do you match the dataflow to a vectorization pattern?  This is a key
  question and there are many possible approaches.  You might want to start
  with a simple pattern matching approach and then expand it (e.g., using a
  solver) to identify more complex patterns.
  




##### Test inputs

One popular autovectorization benchmark is [TSVC](https://github.com/UoB-HPC/TSVC_2) with both simple and complex loops that compilers (GCC and LLVM) fail to vectorize. This might be useful for testing your approach.[YARPGen](https://github.com/intel/yarpgen) is another potential source of test programs.


