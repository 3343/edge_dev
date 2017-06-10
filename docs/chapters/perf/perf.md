# Performance

## Instrumentation
EDGE's performance-relevant functions are manually instrumented. The manual instrumentation uses the libraries [Score-P](http://www.vi-hps.org/projects/score-p/) and [Scalasca](http://www.scalasca.org/software/scalasca-2.x).

### Installation of OTF2
OTF2 (Open Trace Format 2) is optional for installing Score-P and Scalasca.
* Download OTF2 from http://www.vi-hps.org/projects/score-p:
```
wget http://www.vi-hps.org/upload/packages/otf2/otf2-2.0.tar.gz -O otf2.tar.gz
```
* Extract OTF2 to the directory `otf2`:
```
mkdir otf2; tar -xzf otf2.tar.gz -C otf2 --strip-components=1
```
* Configure the installation and set `libs` as installation directory by running:
```
cd otf2; ./configure --prefix=$(pwd)/../libs
```
* Run `make` to build the library and `make install` to put it in the `libs` directory.

### Installation of OPARI2
OPARI2 (OpenMP Pragma And Region Instrumentor) is optional for Score-P's installation, but recommended.
* Download OPARI2 from http://www.vi-hps.org/projects/score-p:
```
wget http://www.vi-hps.org/upload/packages/opari2/opari2-2.0.2.tar.gz -O opari2.tar.gz
```
* Extract OPARI2 to the directory `opari2`:
```
mkdir opari2; tar -xzf opari2.tar.gz -C opari2 --strip-components=1
```
* Configure the installation and set `libs` as installation directory by running:
```
cd opari2; ./configure --prefix=$(pwd)/../libs
```
* Run `make` to build the library and `make install` to put it in the `libs` directory.

### Installation of Score-P
Score-P uses EDGE's high-level instrumentation to generate an executable, providing performance reports on completion.

* Download Score-P from http://www.vi-hps.org/projects/score-p/:
```
wget http://www.vi-hps.org/upload/packages/scorep/scorep-3.1.tar.gz -O scorep.tar.gz
```
* Extract Score-P to the directory `scorep`:
```
mkdir scorep; tar -xzf scorep.tar.gz -C scorep --strip-components=1
```
* Configure the installation and set `libs` as installation directory by running:
```
cd scorep; ./configure --with-otf2=$(pwd)/../libs --with-opari2=$(pwd)/../libs --prefix=$(pwd)/../libs
```
* Run `make` to build the library and `make install` to put it in the `libs` directory.

For the time being only GNU works as toolchain as Score-P throws error/segfaults in EDGE's OMP when using the Intel compilers.
You can enforce GNU compilers by instructing the configure-script accordingly, for example on Stampede 2 using Intel-MPI:
```
module load gcc
. /opt/intel/compilers_and_libraries_2017/linux/bin/compilervars.sh intel64
cd scorep; ./configure --with-otf2=$(pwd)/../libs --with-opari2=$(pwd)/../libs --prefix=$(pwd)/../libs --with-nocross-compiler-suite=gcc --with-mpi=intel3
make
make install
```

You can work around the error `configure: error: required option --interface-version not supported by cube-config.` in Score-P's (version 3.1) and Scalasca's (version 2.3, see below) configure-script by disabling cube: `--with-cube=no`.

### Installation of Scalasca
We use Scalasca to generate performance reports across multiple nodes.

* Download Scalasca from http://www.scalasca.org/:
```
wget http://apps.fz-juelich.de/scalasca/releases/scalasca/2.3/dist/scalasca-2.3.1.tar.gz -O scalasca.tar.gz
```
* Extract Scalasca to the directory `scalasca`:
```
mkdir scalasca; tar -xzf scalasca.tar.gz -C scalasca --strip-components=1
```
* Configure the installation and set `libs` as installation directory by running:
```
cd scalasca; ./configure --with-otf2=$(pwd)/../libs --prefix=$(pwd)/../libs
```
* Run `make` to build the library and `make install` to put it in the `libs` directory.


### Performance Reports
* Build EDGE with the flag `inst=yes`: `PATH=libs/bin:$PATH scons inst=yes`.
Do not build in parallel (don't use `-j` as SCons-flag).
* OMP: Run EDGE as usual
* MPI/MPI+OMP: Prepend program excecution with `scalasca -analyze`

_Remark:_ For some reason Score-P 3.0 shows different number of visits across thread w.r.t. step and cflow-id in the time manager. Here, 5 and 6 visits for `cflow_id=1`:

```
cube_stat -% -m visits scorep-20161012_2200_367901509819854/profile.cubex -r step_id=0
cube::Metric,Routine,Count,Sum,Mean,Variance,Minimum,Quartile 25,Median,Quartile 75,Maximum
visits,INCL(step_id=0),3,68,22.6667,1.33333,22.0000,,22.0000,,24.0000
visits,EXCL(step_id=0),3,34.0000,11.3333,0.333333,11.0000,,11.0000,,12.0000
visits,cflow_id=1,3,16.0000,5.33333,0.333333,5.00000,,5.00000,,6.00000
visits,cflow_id=0,3,18.0000,6.00000,0.00000,6.00000,,6.00000,,6.00000
```
This seems to be a bug.
