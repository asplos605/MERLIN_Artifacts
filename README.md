This is the artifact repository for Asplos submission #605.

Verifier log can be found in /verifier_log/. These examples are selected from the original files.

**Constant propagation and dead code elimination:**
This example comes from Facebook's packet counter.
```
Before optimization:
0: R1=ctx(off=0,imm=0) R10=fp0
; 
0: (b7) r1 = 0                        ; R1_w=0
; 
1: (63) *(u32 *)(r10 -4) = r1
last_idx 1 first_idx 0
regs=2 stack=0 before 0: (b7) r1 = 0
2: R1_w=P0 R10=fp0 fp-8=0000????
; 
2: (63) *(u32 *)(r10 -8) = r1         ; R1_w=P0 R10=fp0 fp-8=00000000
```

```
After optimization:
0: R1=ctx(off=0,imm=0) R10=fp0
; 
0: (05) goto pc+0
; 
1: (62) *(u32 *)(r10 -4) = 0          ; R10=fp0 fp-8=mmmm????
; 
2: (62) *(u32 *)(r10 -8) = 0          ; R10=fp0 fp-8=mmmmmmmm
```
Originally, the fp-8, which means frame pointer with offset -8 is evaluated to 0. After optimization, fp-8 is evaluated to 'mmmmmmmm', meaning the verifier did not evaluate it to a specific value. This shifting indicate the verifier is being more 'strict' instead of 'vulnerable'. While it may lead to slightly slower verification time (i.e. few usecs on our testbench) when verifying related operations, it is trival.

Meanwhile, this constant propagation is adopted in gcc ebpf backend, meaning it does not pose security issues. 

**Superword Level Merging:**
This example shares the same original program as previous optimization. Superword Level Merging targets at constants, so it always work with constant propagation.
```
After optimization:
0: R1=ctx(off=0,imm=0) R10=fp0
; 
0: (05) goto pc+0
; 
1: (05) goto pc+0
; 
2: (7a) *(u64 *)(r10 -8) = 0          ; R10=fp0 fp-8_w=mmmmmmmm
```
The overall result is almost the same. The only difference is the '_w' attribute indicating it is written in form of double word. In the same time, the verifier allows accessing it by word (e.g. reading only first 4 bytes), so there won't be any differences for the rest of the program. Therefore, the code after optimization is always correctly verified and accepted.

**Code Compaction:**
This example comes from Katran's load-balancer.
```
Before optimization:
2033: (85) call bpf_xdp_adjust_head#44        ; R0=scalar()
2034: (67) r0 <<= 32                  ; R0_w=scalar(smax=9223372032559808512,umax=18446744069414584320,var_off=(0x0; 0xffffffff00000000),s32_min=0,s32_max=0,u32_max=0)
2035: (77) r0 >>= 32                  ; R0_w=scalar(umax=4294967295,var_off=(0x0; 0xffffffff))
```
```
After optimization;
2033: (85) call bpf_xdp_adjust_head#44        ; R0=scalar()
2034: (bc) w0 = w0                    ; R0_w=scalar(umax=4294967295,var_off=(0x0; 0xffffffff))
2035: (05) goto pc+0
```
In this case, r0 is the return value of a bpf helper. The exact value is unknown. Instruction #2034 and #2035 use shifts to limit r0 to 32-bit, and the verifier records that as 'umax=4294967295'. After optimization, the verifier also recognizes that r0 is a 32-bit uint, with exactly same information 'umax=4294967295'. Therefore, this optimization is also always correctly verified and accepted.

**Peephole optimization:**
This example also comes from Katran's load-balancer.
```
Before optimization:
1747: (18) r4 = 0xffe00000            ; R4_w=4292870144
1749: (bf) r0 = r2                    ; R0_w=scalar(id=62) R2_w=scalar(id=62)
1750: (5f) r0 &= r4                   ; R0_w=scalar(umax=4292870144,var_off=(0x0; 0xffe00000),s32_max=2145386496) R4_w=4292870144
; return (word << shift) | (word >> ((-shift) & 31));
1751: (77) r0 >>= 21                  ; R0_w=scalar(umax=2047,var_off=(0x0; 0x7ff))
```
```
After optimization;
1747: (05) goto pc+0
1748: (05) goto pc+0
1749: (bf) r0 = r2                    ; R0_w=scalar(id=31) R2=scalar(id=31)
1750: (67) r0 <<= 32                  ; R0_w=scalar(smax=9223372032559808512,umax=18446744069414584320,var_off=(0x0; 0xffffffff00000000),s32_min=0,s32_max=0,u32_max=0)
; return (word << shift) | (word >> ((-shift) & 31));
1751: (77) r0 >>= 53                  ; R0_w=scalar(umax=2047,var_off=(0x0; 0x7ff))
```
Same as previous optimization, the verifier is able to identify the effective bits of r0, and both code ends to be in the same status. The status of optimized code is always correctly updated.