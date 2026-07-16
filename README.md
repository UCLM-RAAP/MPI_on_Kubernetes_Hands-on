# Hands-on: Running the HPL (Linpack) Benchmark on a Kubernetes MPI Cluster

In this Hands-on you will compile and run the **HPL (High Performance Linpack)** benchmark on a Kubernetes cluster using the Kubeflow **MPIJob** operator. Along the way you will:

- See how MPI processes are distributed across pods on Kubernetes
- Compare **UCX** vs plain **TCP** transports on a high-performance interconnect

> **Before you start:** cluster access, resource reservation, connecting to your launcher node, and the `mpi-devel.yaml`. This guide assumes you already have a working launcher pod and can `mpirun` against your assigned workers.

> **Instructor note:** the cluster admins have confirmed that `spec.slotsPerWorker` in the `MPIJob` manifest is a supported, adjustable field — but CPU/memory `resources` are managed via a namespace-level default (a `LimitRange`, most likely) and should **not** be edited in the manifest. This Hands-on therefore uses the base `mpi-devel.yaml` exactly as provided, with **`slotsPerWorker: 1`** left untouched throughout. No YAML edits are required at any point in this guide.

---

## Table of Contents

1. [Deploying the MPI Environment](#1-deploying-the-mpi-environment)
2. [Compiling HPL](#2-compiling-hpl)
3. [A Small `HPL.dat` for Fast Iteration](#3-a-small-hpldat-for-fast-iteration)
4. [Running HPL](#4-running-hpl)
5. [UCX vs TCP](#5-ucx-vs-tcp)
6. [Cleaning Up](#6-cleaning-up)

---

## 1. Deploying the MPI Environment

Use the `MPIJob` manifest already provided on the course page (`mpi-devel.yaml`) exactly as given — no edits needed. It deploys one **Launcher** pod and **2 Worker** pods, with `slotsPerWorker: 1` (1 MPI rank per worker).

```bash
kubectl create -f mpi-devel.yaml 
kubectl get pods 
```

Wait until `mpi-devel-launcher`, `mpi-devel-worker-0`, and `mpi-devel-worker-1` are `Running`, SSH the launcher:

```bash
ssh mpi-devel-launcher@ngt.cern.ch
```

The MPI Operator automatically provides a ready-to-use hostfile at `/etc/mpi/hostfile` and configures OpenMPI to launch remote processes via `kubectl exec` — no SSH needed.

---

## 2. Compiling HPL

We use **HPL 2.3** (2018), the official standalone Linpack benchmark from Netlib. https://www.netlib.org/benchmark/hpl/

### 2.1 Download

```bash
mkdir -p $HOME/src && cd $HOME/src
wget https://netlib.org/benchmark/hpl/hpl-2.3.tar.gz
tar xzf hpl-2.3.tar.gz
```

### 2.2 Compile in `/tmp` (not on EOS)

**Always compile in local `/tmp`, never directly on the shared/network filesystem (EOS/CVMFS/AFS).** These are network filesystems and can cause unpredictable build failures under the file-locking and I/O patterns `make`/`ar` use.

```bash
mkdir -p /tmp/hpl-build
cp -r $HOME/src/hpl-2.3 /tmp/hpl-build/
cd /tmp/hpl-build/hpl-2.3
```

### 2.3 The Makefile

Copy this ready-to-use `Make.Linux_PII_CBLAS` into `/tmp/hpl-build/hpl-2.3/` — it's already configured for this cluster's OpenMPI and OpenBLAS installation:

```makefile
SHELL        = /bin/sh
CD           = cd
CP           = cp
LN_S         = ln -s
MKDIR        = mkdir
RM           = /bin/rm -f
TOUCH        = touch

ARCH         = $(arch)

TOPdir       = /tmp/hpl-build/hpl-2.3
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin/$(ARCH)
LIBdir       = $(TOPdir)/lib/$(ARCH)

HPLlib       = $(LIBdir)/libhpl.a

MPdir        = /usr/local/mpi
MPinc        = -I$(MPdir)/include
MPlib        = -Wl,-rpath -Wl,$(MPdir)/lib -L$(MPdir)/lib -lmpi

LAdir        = /usr/lib64
LAinc        =
LAlib        = /usr/lib64/libopenblas.so.0

F2CDEFS      =

HPL_INCLUDES = -I$(INCdir) -I$(INCdir)/$(ARCH) $(LAinc) $(MPinc)
HPL_LIBS     = $(HPLlib) $(LAlib) $(MPlib)

HPL_OPTS     = -DHPL_CALL_CBLAS

HPL_DEFS     = $(F2CDEFS) $(HPL_OPTS) $(HPL_INCLUDES)

CC           = mpicc
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -fomit-frame-pointer -O3 -funroll-loops

LINKER       = mpicc
LINKFLAGS    = $(CCFLAGS)

ARCHIVER     = ar
ARFLAGS      = r
RANLIB       = echo
```

### 2.4 Build

```bash
make arch=Linux_PII_CBLAS
```

The binary appears at `bin/Linux_PII_CBLAS/xhpl`.

### 2.5 Copy the result to shared storage

`/tmp` is **local to the launcher pod only** — the workers won't see it. `mpirun` needs the binary at the same path on every node, so copy it to your shared home (EOS):

```bash
mkdir -p $HOME/src/hpl-2.3/bin/Linux_PII_CBLAS
cp bin/Linux_PII_CBLAS/xhpl    $HOME/src/hpl-2.3/bin/Linux_PII_CBLAS/
cp bin/Linux_PII_CBLAS/HPL.dat $HOME/src/hpl-2.3/bin/Linux_PII_CBLAS/
```

Verify the binary resolves its libraries correctly outside `/tmp`:

```bash
ldd $HOME/src/hpl-2.3/bin/Linux_PII_CBLAS/xhpl
```

All entries should point to system paths (`/usr/local/mpi/lib/...`, `/usr/lib64/...`) — none to `/tmp`.

From now on, always work from:

```bash
cd $HOME/src/hpl-2.3/bin/Linux_PII_CBLAS
```

---

## 3. A Small `HPL.dat` for Fast Iteration

The file to edit is:

``` bash
$HOME/src/hpl-2.3/bin/Linux_PII_CBLAS/HPL.dat
```

This is the copy you placed on shared storage in step 2.5 — edit **this** one, not the one still sitting in `/tmp`. It needs to live on the shared filesystem (EOS) because only MPI rank 0 actually reads it (the contents are then broadcast internally to the other ranks), and you don't control in advance which node ends up running rank 0 — so every node needs the file at the same path.

For a Hands-on, we want each run to finish in **seconds, not hours**, with a single process grid matching our 2 workers. Replace the entire contents of `HPL.dat` with exactly this (including the first two header lines — HPL's parser expects them to be present, even though their text isn't otherwise used):

``` txt
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
8000         Ns
1            # of NBs
192          NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1            Ps
2            Qs
16.0         threshold
1            # of panel fact
2            PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
4            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
2            RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

The fields that matter most for this Hands-on:

| Line | Field | Value | Why |
|---|---|---|---|
| `Ns` | Problem size `N` | `8000` | Small enough to finish in seconds |
| `NBs` | Block size `NB` | `192` | Reasonable default, keep it fixed |
| `Ps` / `Qs` | Process grid | `1` / `2` | Must satisfy `P × Q = -np` — matches our 2 workers, 1 rank each |

> Once you understand the full workflow, feel free to try a larger `N` and let a "real" benchmark run in the background — but for the comparisons below, keep it small so each test completes quickly.

---

## 4. Running HPL

```bash
export OPENBLAS_NUM_THREADS=1
mpirun -np 2 \
  -x OPENBLAS_NUM_THREADS \
  --mca pml ucx \
  ./xhpl
```

Take note of the **Gflops** value at the end of the output — this is your baseline for the network comparison in the next section. To obtain a more reliable result, the benchmark should be excecuted at least 3 times and calculate the average of the results.

---

## 5. UCX vs TCP

This is a high-performance interconnection networks Hands-on, so let's make the interconnect choice visible. OpenMPI can move data between nodes over several transports; on this cluster the RDMA-capable path is exposed to **UCX**, while a naive configuration falls back to plain **TCP** — much slower for MPI collective operations at scale.

> **What's actually available in this container:** check with `ompi_info | grep -i "MCA pml"` and `ompi_info | grep -i "MCA btl"`. On this image you'll find PML components `ucx` and `ob1`, and BTL components `tcp`, `vader`, `self`, `smcuda` — **notably no `openib`**. That's expected: this build handles RDMA through UCX directly rather than the older `openib` BTL, which OpenMPI has been phasing out.
>
> **Important:** UCX is selected **automatically by default** here (highest PML priority) — if you run `mpirun` without specifying `--mca pml`, you're already using UCX.

### 5.1 Run with UCX (RDMA)

This is what you already ran in Section 4.

```bash
mpirun -np 2 \
  -x OPENBLAS_NUM_THREADS \
  --mca pml ucx \
  ./xhpl
```

### 5.2 Run forcing TCP only

First, find the real network interface to use. These pods share the node's network namespace (`hostNetwork`), so a plain `ip addr` shows dozens of unrelated `cali*` interfaces belonging to *other* pods on the same node — filter those out along with Calico's internal overlay interfaces:

```bash
mpirun -np 2 sh -c "ip -o -4 addr show scope global \
            | grep -vE 'cali|docker0|tunl0'"
```

You should get one clean line per worker, showing the real physical NIC (e.g. `enp194s0f0np0`) with a routable IP and typically a large MTU (9000, jumbo frames) — that's the one to use. Interfaces with no `inet` line (`NO-CARRIER`) or link-local addresses (filtered out by `scope global`) are not candidates.

Now force OpenMPI to ignore UCX/RDMA entirely and use only that interface over TCP:

```bash
mpirun -np 2 -x OPENBLAS_NUM_THREADS \
  --mca pml ob1 \
  --mca btl tcp,self \
  --mca btl_tcp_if_include enp194s0f0np0 \
  ./xhpl
```

- `--mca pml ob1 --mca btl tcp,self` disables UCX and forces the older point-to-point layer over plain TCP sockets.
- `--mca btl_tcp_if_include enp194s0f0np0` restricts TCP traffic to the real physical interface you just identified — **use `if_include` rather than `if_exclude`** here: with so many unrelated `cali*` interfaces visible due to `hostNetwork`, explicitly whitelisting the one you want is far safer than trying to exclude everything you don't.

> Replace `enp194s0f0np0` with whatever interface name your own discovery command returned — it can differ between nodes.

**Compare Gflops to the UCX run.** For communication-heavy problem sizes you should see TCP noticeably slower — the gap grows with problem size and process count, since HPL's broadcast/reduce steps depend heavily on interconnect latency and bandwidth.

---

## 6. Cleaning Up

**Always delete the `MPIJob` resource itself — never delete individual pods directly.** The MPI Operator owns the pods; deleting pods without deleting the parent object can make the operator immediately recreate them, or leave pods stuck in `Terminating`.

```bash
kubectl delete mpijob mpi-devel 
```

---
