# sherpa3-conda  (demo)

Conda packaging for the **SHERPA 3** Monte-Carlo event generator
([gitlab.com/sherpa-team/sherpa](https://gitlab.com/sherpa-team/sherpa)) and its
one-loop matrix-element provider **OpenLoops**, with CI that builds both packages and
verifies the build by generating **W-boson production at LO and NLO**.

> **This repo is for packaging `sherpa3` (+ `openloops`) on conda** — a staging/demo
> ground for circulating the packages ahead of the official
> `conda-forge/staged-recipes` submission. (It's `sherpa3` because the conda-forge name
> `sherpa` already belongs to the unrelated X-ray-astronomy package.)

## Layout
```
openloops/   recipe.yaml + build.sh   ->  conda package  openloops 2.1.5  (GPL-3.0, SCons)
sherpa3/     recipe.yaml + build.sh   ->  conda package  sherpa3   3.0.4  (GPL-3.0, CMake)
test/        W_LO/ , W_NLO/           ->  W-production runcards to verify the build
.github/workflows/ci.yml             ->  build openloops -> build sherpa3 -> run W LO + NLO
```
`sherpa3` is built with LHAPDF, HepMC3, Rivet, Pythia8, **OpenLoops**, Python (pysherpa) and MPI.

## CI — `build-and-verify`
One Linux job, micromamba-based, three chained steps:
1. **Build OpenLoops** with `rattler-build` → `./output`.
2. **Build sherpa3 on top of OpenLoops** — `--channel ./output`, so the freshly built
   `openloops` satisfies sherpa3's dependency.
3. **Verify** — install `sherpa3` from `./output`, run **W → eν at LO** (full
   ME + shower + hadronisation), then have **OpenLoops compile a W one-loop library**
   (`openloops libinstall ppllj`) and **load it into sherpa3** — exercising the openloops
   package + the sherpa3↔OpenLoops interface end-to-end. (A full MC@NLO *event sample* on
   top additionally needs SHERPA's Amegic Born libraries compiled — a SHERPA run-config
   detail, not a packaging one.)

Built `.conda` files are uploaded as a CI artifact. To circulate them, add the repo
secret `ANACONDA_TOKEN` (and optional variable `ANACONDA_OWNER`) — the optional publish
step then uploads to `anaconda.org/<owner>` on pushes to `main`.

## Build / use locally
```bash
micromamba create -n build -c conda-forge rattler-build
rattler-build build --recipe openloops/recipe.yaml --output-dir ./output -c conda-forge
rattler-build build --recipe sherpa3/recipe.yaml   --output-dir ./output -c ./output -c conda-forge

micromamba create -n sherpa -c ./output -c conda-forge sherpa3
micromamba run -n sherpa lhapdf install NNPDF23_lo_as_0130_qed
cd test/W_LO && micromamba run -n sherpa Sherpa -f Sherpa.yaml
```

## Notes / TODO before conda-forge
- Linux-64 only for now (osx flag guards + the openmpi/mpich MPI variant are still TODO).
- OpenLoops process libraries for NLO are fetched at run time
  (`openloops libinstall <process>`), which needs a Fortran compiler — same model as
  LHAPDF PDF sets.
- conda-forge path: submit `openloops` to staged-recipes first; once it is live there,
  submit `sherpa3` (it can only build against an already-published `openloops`).
