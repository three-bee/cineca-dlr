# Issues with `gcc`, `GLIBC` and `GLIBCXX` and installing `pymeshlab`

**NOTE:** If you did not upgrade `glibc`, `pip install $YOUR_PYMESHLAB_WHEEL` and `python -c "import pymeshlab"` will work just fine. 


However; if you have built `glibc` locally to upgrade (for instance, Kaolin for `glibc=2.29`), the local linker may not see the correct `GLIBCXX` files for other packages and fail to load (for instance, `pymeshlab`).

```python
>>> import pymeshlab
    from .pmeshlab import *
    ImportError: /usr/lib64/libstdc++.so.6: version GLIBCXX_3.4.29' not found (required by /leonardo_work/EUHPC_A02_031/3d-gan-diff/PyMeshLab/pymeshlab/pmeshlab.cpython-310-x86_64-linux-gnu.so)
```

We will update the `python` override in `$YOUR_ENV/bin/activate` for `gcc`, and build `pymeshlab` from source by specifying the locally updated `glibc` linkers in `cmake` configurations.


## Updating `gcc` and `GLIBCXX`
| Feature           | **GCC** (GNU Compiler Collection)        | **glibc** (GNU C Library)             | **GLIBCXX** (GNU C++ Standard Library) |
|-------------------|------------------------------------------|---------------------------------------|----------------------------------------|
| **Purpose**        | Compiler for C, C++, and other languages | C standard library providing system functions | C++ standard library for C++ language features |
| **Role**           | Compiles code into machine-executable format | Provides core system functions (e.g., memory, I/O) | Provides C++ functionality (e.g., `std::vector`, `std::cout`) |
| **Components**     | `gcc`, `g++`, `libstdc++` (C++ library)  | Core system library (`libc.so`, `libm.so`) | C++ standard library (`libstdc++.so`) |
| **Used For**       | Compilation of C, C++, and other languages | Execution of C and C++ programs on Linux | Execution of C++ programs that use standard C++ features |
| **Versioning**     | Versions like **GCC 9.3**, **GCC 12.2**  | Versions like **glibc 2.28**, **glibc 2.29** | Versions like **GLIBCXX_3.4.29** (from `libstdc++.so`) |
| **Examples**       | `gcc -o output program.c`                | `printf`, `malloc`, `fopen`, `pthread_create` | `std::vector`, `std::cout`, `std::thread` |

* GLIBCXX is provided by GCC (through `libstdc++.so`). If a higher version of `gcc` or `GLIBCXX` is required, the default `gcc-8.5.0` in CINECA may cause issues.

* To upgrade `glibc` locally, see Kaolin installation.
* To upgrade `gcc` and consequently `GLIBCXX`, perform the following:
    1. Comment out the following anaconda block in `~\.bashrc`:
    ```bash
        # >>> conda initialize >>>
        # !! Contents within this block are managed by 'conda init' !!
        __conda_setup="$('/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/anaconda3-2023.09-0-zcre7pfofz45c3btxpdk5zvcicdq5evx/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
        if [ $? -eq 0 ]; then
            eval "$__conda_setup"
        else
            if [ -f "/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/anaconda3-2023.09-0-zcre7pfofz45c3btxpdk5zvcicdq5evx/etc/profile.d/conda.sh" ]; then
                . "/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/anaconda3-2023.09-0-zcre7pfofz45c3btxpdk5zvcicdq5evx/etc/profile.d/conda.sh"
            else
                export PATH="/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/anaconda3-2023.09-0-zcre7pfofz45c3btxpdk5zvcicdq5evx/bin:$PATH"
            fi
        fi
        unset __conda_setup
        # <<< conda initialize <<<
    ```
    2. Load the desired gcc module: `module load gcc` (as of now loads 12.2.1)
    3. Load a suitable python version: `module load python/3.10.8--gcc--11.3.0`
    4. If you are using a custom `glibc` version (default is 2.28) and specified the local linker in `python` like this in `$YOUR_ENV/bin/activate`: 
        ```bash
        python() {
        $HOME/glibc-2.29/lib/ld-linux-x86-64.so.2 --library-path $HOME/glibc-2.29/lib:$HOME/glibc-2.29/lib64:/usr/lib64:/lib64 $YOUR_ENV_PATH/bin/python "$@"
        }
        ```
        add the following `/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/gcc-12.2.0-gmhym3kmbzqlpwkzhgab2xsoygsdwxcl/lib64`to `--library-path`:
        ```bash
        python() {
        $HOME/glibc-2.29/lib/ld-linux-x86-64.so.2 --library-path $HOME/glibc-2.29/lib:$HOME/glibc-2.29/lib64:/leonardo/prod/spack/5.2/install/0.21/linux-rhel8-icelake/gcc-8.5.0/gcc-12.2.0-gmhym3kmbzqlpwkzhgab2xsoygsdwxcl/lib64:/lib64:$LD_LIBRARY_PATH /leonardo_work/EUHPC_A02_031/3d-gan-diff/3dgandiff_env/bin/python "$@"
        }
        ```
        so that the latest `libstdc++.so` can be used in dynamic linking process while importing python packages.

## Building `pymeshlab`

```bash
git clone --recursive https://github.com/cnr-isti-vclab/PyMeshLab.git
cd PyMeshLab ## This is $YOUR_PYMESHLAB_PATH
mkdir build
cd build

cmake -GNinja \ 
-DCMAKE_BUILD_TYPE=Release \ 
-DCMAKE_C_COMPILER=gcc \ 
-pymDCMAKE_CXX_COMPILER=g++ \ 
-DCMAKE_C_FLAGS="-Wl,--dynamic-linker=$HOME/glibc-2.29/lib/ld-linux-x86-64.so.2 -L$HOME/glibc-2.29/lib -L$HOME/glibc-2.29/lib64" \ 
-DCMAKE_CXX_FLAGS="-Wl,--dynamic-linker=$HOME/glibc-2.29/lib/ld-linux-x86-64.so.2 -L$HOME/glibc-2.29/lib -L$HOME/glibc-2.29/lib64"
..

ninja
ninja install
```

After building PyMeshLab, add the following line to your `$YOUR_ENV/bin/activate`:
```bash
export PYTHONPATH=$YOUR_PYMESHLAB_PATH:$PYTHONPATH
```