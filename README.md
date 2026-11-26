# git_new_features


wenke  1111

## Intel Hardware PGO (HWPGO)

Hardware Profile-Guided Optimization (HWPGO) is a feature of the Intel® oneAPI DPC++/C++ Compiler (`icx`/`icpx`), available since the 2024.0 release. Traditional PGO builds an instrumented binary to collect a profile. HWPGO instead uses the CPU's Performance Monitoring Unit (PMU) to sample a normally optimized binary, then turns those samples into a source-level profile that guides recompilation. It is a form of sample-based PGO (SPGO).

**Why use it**
- Lower overhead: the profiled binary is not instrumented, so it runs close to production speed.
- Richer feedback: PMU events such as taken branches and branch mispredictions can guide decisions like code layout and branch-to-cmov conversion.
- Profiles can be collected on representative or production workloads.

**Workflow (Linux)**
1. Build with profiling support (the LLD linker is required):
   ```sh
   icx -O3 -fprofile-sample-generate app.c -o app
   ```
   The `-fprofile-sample-generate[=keep-all-opt|med-fidelity|max-fidelity]` option trades optimization level for profile accuracy. The default is `keep-all-opt`.
2. Collect PMU samples with `perf`, using LBR via `-b`:
   ```sh
   perf record -o app.perf.data -b -c 1000003 \
     -e BR_INST_RETIRED.NEAR_TAKEN:uppp,BR_MISP_RETIRED.ALL_BRANCHES:upp -- ./app
   ```
3. Convert the samples into profiles with the oneAPI `llvm-profgen` (run `icx --print-prog-name=llvm-profgen` to find it):
   ```sh
   llvm-profgen --format text --binary=app --perfdata=app.perf.data \
     --sample-period=1000003 --perf-event=BR_INST_RETIRED.NEAR_TAKEN:uppp \
     --output=app.freq.prof
   llvm-profgen --format text --binary=app --perfdata=app.perf.data \
     --sample-period=1000003 --perf-event=BR_MISP_RETIRED.ALL_BRANCHES:upp \
     --leading-ip-only --output=app.misp.prof
   ```
4. Rebuild using the profile:
   ```sh
   icx -O3 -fprofile-sample-use=app.freq.prof app.c -o app
   ```

On Windows, use `/fprofile-sample-generate` and `/fprofile-sample-use`. Collect samples with the SEP tool from Intel® VTune™ Profiler, and pass `llvm-profgen --perfscript=...` with the `:pdir` event modifier.

**References**
- [Intel: Hardware Profile-Guided Optimization (compiler guide)](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2024-2/hardware-profile-guided-optimization.html)
- [Intel: Hardware-based PGO technical article](https://www.intel.com/content/www/us/en/developer/articles/technical/hwpgo.html)
- [EuroLLVM 2024: Enabling HW-based PGO](https://llvm.org/devmtg/2024-04/slides/TechnicalTalks/Xiao-EnablingHW-BasedPGO.pdf)
