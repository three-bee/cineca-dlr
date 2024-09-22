# Installing Kaolin

## Load necessary modules & packages:


```bash
module load python/3.10.8--gcc--11.3.0
module load openblas
module load git-lfs
source $YOUR_ENV_PATH/bin/activate
```

Install Torch and Kaolin:
```
pip install torch==2.4.0 torchvision==0.19.0 torchaudio==2.4.0 --index-url https://download.pytorch.org/whl/cu121
pip install kaolin==0.13.0 -f https://nvidia-kaolin.s3.us-east-2.amazonaws.com/torch-2.4.0_cu121.html
```

The oldest Kaolin version that can run on Leonardo as of now (CUDA 12.1) requires GNU C Library (GLIBC) 2.29. `ldd --version` will give 2.28. 

2.29 is not located among modules (`modmap -all | grep glibc` should return empty) Hence, we need to build it locally.

```python
>> import kaolin
    ImportError: /lib64/libm.so.6: version GLIBC_2.29' not found (required by $YOUR_ENV_PATH/lib/python3.10/site-packages/kaolin/_C.so)
```

## Building `bison` and `glibc-2.29` from source locally

`bison` is a pre-requisite for `glibc-2.29` on Leonardo as of now. 

```bash
wget https://ftp.gnu.org/gnu/bison/bison-3.8.2.tar.gz
tar -xzf bison-3.8.2.tar.gz
cd bison-3.8.2
mkdir $HOME/bison
./configure --prefix=$HOME/bison
make
make install
echo $?
```
If successful, `echo` should give `0`. Then, append `export PATH=$HOME/bison/bin:$PATH` to `~/.bashrc`. Run `source ~/.bashrc` or open a new terminal. Check `bison --version`.

For `glibc`,

```bash
wget http://ftp.gnu.org/gnu/libc/glibc-2.29.tar.gz
tar -xvzf glibc-2.29.tar.gz
cd glibc-2.29
mkdir $HOME/glibc-2.29
../configure --prefix=$HOME/bison
make -j$(nproc)
make install
echo $?
```
If successful, `echo` should give `0`. Then, append `export PATH=$HOME/glibc-2.29/bin:$PATH` to `~/.bashrc`. Run `source ~/.bashrc` or open a new terminal. Check `ldd --version`.


## Dynamic link library path (`LD_LIBRARY_PATH`) issues

If you also try to append `export LD_LIBRARY_PATH=$HOME/glibc-2.29/lib:$HOME/glibc-2.29/lib64:/usr/lib64:/lib64:$LD_LIBRARY_PATH` to `~/.bashrc` (which should be the logical next step to use _C python packages), you will unfortunately get this:

```
>> ldd --version
    ld.so: dl-call-libc-early-init.c: 37: _dl_call_libc_early_init: Assertion sym != NULL' failed!
```

But linking is required to run Kaolin. Somehow the global 2.28 and local 2.29 is mixed up. 

So we get around by this: `deactivate` your env, open your `$YOUR_ENV_PATH/bin/activate`, and append:

```bash
export PATH=$HOME/bison/bin:$PATH
export PATH=$HOME/glibc-2.29/bin:$PATH

python() {
$HOME/glibc-2.29/lib/ld-linux-x86-64.so.2 --library-path $HOME/glibc-2.29/lib:$HOME/glibc-2.29/lib64:/usr/lib64:/lib64:$LD_LIBRARY_PATH $YOUR_ENV_PATH/bin/python "$@"
}
```

Then, `source $YOUR_ENV_PATH/bin/activate` and `python -c "import kaolin"`. Hopefully you will get no errors! Remove the modifications done on `PATH` in `.bashrc` too, since `activate` will handle it.

I believe that modifying `LD_LIBRARY_PATH` in `.bashrc` should have been enough for normal systems, but CINECA clusters are quite unique :)
