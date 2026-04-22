---
layout: default
title: About
---

## Dr. Matthias Kretz

![Matthias Kretz](/assets/images/me.jpg){:style="float: left; width: 250px; margin-right: 2rem; border-radius: 10%;"}

**Senior Software Engineer and SIMD/C++ Standardization Lead**  
*Husband — Father — Nerd — Athlete*

I am a Senior Software Engineer at [GSI Helmholtzzentrum für Schwerionenforschung](https://www.gsi.de/en/) in Darmstadt, 
Germany, and chair of WG21 SG6 (Numerics).
With over a decade of contribution to the C++ standardization process, my work spans from contributions to KDE to 
leading the development of `std::simd` for C++ and its implementation in GCC.

I aspire for efficiency by removing complexity—from making computers accessible through KDE to enabling scientists to 
write high-performance code without wrestling with vectorization details.
I recognize that every abstraction adds its own complexity, but believe in shifting it to where it matters most: 
compile-time verification rather than runtime surprises.

My research evolved from using SIMD for track reconstruction at CERN to SIMD types (including my PhD work), leading to 
my contribution of `std::experimental::simd` to the Parallelism TS and its eventual inclusion as `std::simd` into C++26.
I implement and maintain these in GCC, making my work accessible out-of-the box to most scientists.
I focus my language design efforts on `std::simd` and related numerics topics due to the specialized nature of this 
work.
You can find all my WG21 papers by searching for "Kretz" at 
[open-std.org](https://open-std.org/jtc1/sc22/wg21/docs/papers/) or explore the repository at 
[wg21-papers](https://github.com/mattkretz/wg21-papers/).
There is also my talk from
[CppCon 2023: std::simd — How to Express Inherent Parallelism Efficiently via Data-Parallel 
 Types](https://cppcon.programmingarchive.com/?mgi_13=7395/stdsimd-how-to-express-inherent-parallelism-efficiently-via-data-parallel-types-matthias-kretz).

### Research &amp; Implementation Interests

- **C++ Language Design** — Consistency, orthogonality, high performance, 
simplicity, and general applicability
- **Data Parallelism** — SIMD, ILP, memory access patterns, performance 
portability, GPU computing
- **Scientific Computing** — Numerics support for HPC and High-Energy Physics workloads
- **Compiler Optimization** — Practical implementation experience driving 
"missed optimization" reports

### Research Vision

Data-parallel types express data parallelism in general—they are not bound to CPUs with SIMD instructions. The same abstraction could translate to GPU execution or other parallel architectures. Achieving this requires novel compiler interactions to maintain the abstraction while exposing architecture-specific performance characteristics.

### Selected Publications

- **Kretz, M.** (2015). Extending C++ for explicit data-parallel programming via SIMD vector types. *Dissertation, Johann Wolfgang Goethe-Universität Frankfurt am Main*. [PDF](http://publikationen.ub.uni-frankfurt.de/frontdoor/index/index/docId/38415). [URN](urn:nbn:de:hebis:30:3-384155)
- **Heller, T., Adelstein Lelbach, B., Huck, K.A., et al. including Kretz, M.** (2019). Harnessing billions of tasks for a scalable portable hydrodynamic simulation of the merger of two stars. *The International Journal of High Performance Computing Applications*, 33(4), 699-715. [DOI](https://doi.org/10.1177/1094342018819744)
- **Kretz, M., & Lindenstruth, V.** (2011). Vc: A C++ library for explicit vectorization. *Software: Practice and Experience*. [DOI](https://doi.org/10.1002/spe.1149)
- **Rohr, D., Gorbunov, S., Kretz, M., et al.** (2012). ALICE HLT TPC Tracking of Pb-Pb Events on GPUs. *Journal of Physics: Conference Series*, 396(1). [DOI](https://doi.org/10.1088/1742-6596/396/1/012044)
- **Gorbunov, S., Rohr, D., Kretz, M., et al.** (2011). ALICE HLT High Speed Tracking on GPU. *IEEE Transactions on Nuclear Science*, 58(4). [DOI](https://doi.org/10.1109/TNS.2011.2157702)
- **Kretz, M.** (2009). Efficient Use of Multi- and Many-Core Systems with Vectorization and Multithreading. *Diplomarbeit, Universität Heidelberg*.

### Honors & Awards

- **Rainer-Kemp-Dissertations-Preis** (2016) — Best dissertation in Computer Science and Mathematics, Goethe University Frankfurt
- **Studienstiftung des deutschen Volkes** (2001–2008) — Scholarship from Germany's largest academic foundation
- **KDE Akademy Award** (2007) — "Best Non-Application" for Phonon multimedia framework
- **Bundeswettbewerb Informatik** (2001) — Winner of Germany's National Computer Science Competition; Special Award for Cooperative Behavior

### Software Projects

- [std::simd](https://github.com/GSI-HPC/simd) Staging area before 
contribution to GCC
- [vir-simd](https://github.com/mattkretz/vir-simd) Extensions for 
`std::experimental::simd`
- [C++ CI docker images](https://github.com/mattkretz/cplusplus-ci)
- [value-preserving literals](https://github.com/GSI-HPC/value-preserving-literals)
- [bit_int](https://github.com/mattkretz/bit_int) — Compile-time computation 
for bit-width integers
- Various WG21-related tools and patches
- See [GitHub profile](https://github.com/mattkretz) for complete list
- See [Sourceware Forge](https://forge.sourceware.org/mkretz) for my 
  GCC-related work

When I still had time for KDE:
- KDE Phonon library (KDE 4)
- KDE core library contributor for KDE 4
- aRts contributions (KDE 2/3 sound server)
- KView (KDE 2)

### Connect

- Mastodon: [@mkretz@floss.social](https://floss.social/@mkretz)
- GitHub: [mattkretz](https://github.com/mattkretz)
- ORCID: [0000-0002-0867-243X](https://orcid.org/0000-0002-0867-243X)
- Garmin Connect: [Profile](https://connect.garmin.com/app/profile/4827aa7d-deae-410e-82f6-98b73fe8e659)
- Strava: [Profile](https://www.strava.com/athletes/124318317)
