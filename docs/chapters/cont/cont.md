# Continuity
## Basic CI {#sec_basic_ci}
EDGE's basic Continuous Integration (CI) has low resource requirements and uses a collection of tools.
The basic CI and corresponding webhooks are used for:
  * Test of EDGE's installation process, including all tools and dependencies, from scratch through [Travis CI](https://travis-ci.org/3343/edge) (virtual machine).
  * Very basic sanity checks of different compile modes and executions through Travis CI.
  * Generation of code coverage reports through [Codecov](https://codecov.io/gh/3343/edge).
  * Test of EDGE's container support through execution of a [Singularity](http://singularity.lbl.gov) [bootstrap]({{book.edge_git}}/blob/develop/tools/singularity/debian.def) (Singularity requires root-permissions for the bootstrap) in Travis CI.
  * Test of EDGE's BSD 3-Clause license and the compliance of submodules through [FOSSA](https://app.fossa.io/projects/git%2Bhttps%3A%2F%2Fgithub.com%2F3343%2Fedge).

## Continuous Delivery
Continuous Delivery (CD) of EDGE with moderate resource requirements uses [GoCD](https://www.gocd.io).
Currently the GoCD web-interface and all computing resources are protected by a firewall.
If you want access to these, please get in touch.
In comparison to the Basic CI, the systems are preconfigured and only direct dependencies and EDGE itself are build.
EDGE's CD through GoCD is used for:
  * Test of EDGE's installation process, including libraries, using GNU, LLVM and Intel compilers.
  * Sanity checks of every commit covering a wide range of configurations, e.g., solved equations, used element types, structured/unstructured meshes, level of parallelization, vanilla kernels/runtime code generation, non-fused/fused runs, vector instruction sets. The sanity checks also include runs with the GNU and LLVM sanitizers, valgrind, and the Intel Inspector XE for detection of undefined behavior, memory leaks, etc.
  * Sanity checks of machine-specific optimizations. Currently, optimizations targeting Haswell (AVX2) and KnightsLanding (AVX512) are tested.
  * Automated convergence benchmarks of different configurations. Due to higher runtimes, the convergence benchmarks are executed on a time basis, currently once per week, and not after every commit.
  * Automated wave propagation benchmarks for seismic simulations. These are also executed once a week.
