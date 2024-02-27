Usage:

'alignbpf' for libAlignBPF.so
'atomicbpf' for libAtomicBPF.so

All passes are built with LLVM 17. Please pass '--load-pass-plugin /path/to/libAlignBPF.so --load-pass-plugin /path/to/libAtomicBPF.so -passes=alignbpf,atomicbpf' to opt to apply these passes.
