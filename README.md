<p align="center">
  <a href="https://github.com/nschloe/optimesh"><img alt="optimesh" src="figs/optimesh-logo.svg" width="60%"></a>
  <p align="center">Triangular mesh optimization.</p>
</p>

[![PyPi Version](https://img.shields.io/pypi/v/optimesh.svg?style=flat-square)](https://pypi.org/project/optimesh/)
[![PyPI pyversions](https://img.shields.io/pypi/pyversions/optimesh.svg?style=flat-square)](https://pypi.org/project/optimesh/)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.4728056.svg?style=flat-square)](https://doi.org/10.5281/zenodo.4728056)
[![GitHub stars](https://img.shields.io/github/stars/nschloe/optimesh.svg?style=flat-square&logo=github&label=Stars&logoColor=white)](https://github.com/nschloe/optimesh)
[![PyPi downloads](https://img.shields.io/pypi/dm/optimesh.svg?style=flat-square)](https://pypistats.org/packages/optimesh)

[![Discord](https://img.shields.io/static/v1?logo=discord&logoColor=white&label=chat&message=on%20discord&color=7289da&style=flat-square)](https://discord.gg/PBCCvwHqpv)

[![gh-actions](https://img.shields.io/github/workflow/status/nschloe/optimesh/ci?style=flat-square)](https://github.com/nschloe/optimesh/actions?query=workflow%3Aci)
[![codecov](https://img.shields.io/codecov/c/github/nschloe/optimesh.svg?style=flat-square)](https://app.codecov.io/gh/nschloe/optimesh)
[![LGTM](https://img.shields.io/lgtm/grade/python/github/nschloe/optimesh.svg?style=flat-square)](https://lgtm.com/projects/g/nschloe/optimesh)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square)](https://github.com/psf/black)

Several mesh smoothing/optimization methods with one simple interface. optimesh

- is fast,
- preserves submeshes,
- only works for triangular meshes, flat and on a surface, (for now; upvote [this
  issue](https://github.com/nschloe/optimesh/issues/17) if you're interested in
  tetrahedral mesh smoothing), and
- supports all mesh formats that [meshio](https://github.com/nschloe/meshio) can
  handle.

Install with

```
pip install optimesh
```

Example call:

```
optimesh in.e out.vtk
```

Output:
![terminal-screenshot](figs/term-screenshot.png)

The left hand-side graph shows the distribution of angles (the grid line is at the
optimal 60 degrees). The right hand-side graph shows the distribution of simplex
quality, where quality is twice the ratio of circumcircle and incircle radius.

All command-line options are documented at

```
optimesh -h
```

### Showcase

![disk-step0](figs/disk-step0.png)

The following examples show the various algorithms at work, all starting from the same
randomly generated disk mesh above. The cell coloring indicates quality; dark green is
bad, yellow is good.

#### CVT (centroidal Voronoi tessellation)

|                             ![cvt-uniform-lloyd2](figs/lloyd2.webp)                             |          ![cvt-uniform-qnb](figs/cvt-uniform-qnb.webp)           |        ![cvt-uniform-qnf](figs/cvt-uniform-qnf.webp)         |
| :---------------------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------: | :----------------------------------------------------------------------------------------: |
| uniform-density relaxed [Lloyd's algorithm](https://en.wikipedia.org/wiki/Lloyd%27s_algorithm) (`--method lloyd --omega 2.0`) | uniform-density quasi-Newton iteration (block-diagonal Hessian, `--method cvt-block-diagonal`) | uniform-density quasi-Newton iteration (default method, full Hessian, `--method cvt-full`) |

Centroidal Voronoi tessellation smoothing ([Du et al.](#relevant-publications)) is one
of the oldest and most reliable approaches. optimesh provides classical Lloyd smoothing
as well as several variants that result in better meshes.

#### CPT (centroidal patch tessellation)

|                                        ![cpt-cp](figs/cpt-dp.png)                                         | ![cpt-uniform-fp](figs/cpt-uniform-fp.webp) | ![cpt-uniform-qn](figs/cpt-uniform-qn.webp) |
| :-------------------------------------------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------: | :-----------------------------------------------------------------------: |
| density-preserving linear solve ([Laplacian smoothing](https://en.wikipedia.org/wiki/Laplacian_smoothing), `--method cpt-linear-solve`) |    uniform-density fixed-point iteration (`--method cpt-fixed-point`)     |        uniform-density quasi-Newton (`--method cpt-quasi-newton`)         |

A smoothing method suggested by [Chen and Holst](#relevant-publications), mimicking CVT
but much more easily implemented. The density-preserving variant leads to the exact same
equation system as Laplacian smoothing, so CPT smoothing can be thought of as a
generalization.

The uniform-density variants are implemented classically as a fixed-point iteration and
as a quasi-Newton method. The latter typically converges faster.

#### ODT (optimal Delaunay tessellation)

| ![odt-dp-fp](figs/odt-dp-fp.webp) | ![odt-uniform-fp](figs/odt-uniform-fp.webp) | ![odt-uniform-bfgs](figs/odt-uniform-bfgs.webp) |
| :-------------------------------------------------------------: | :-----------------------------------------------------------------------: | :---------------------------------------------------------------------------: |
| density-preserving fixed-point iteration (`--method odt-dp-fp`) |    uniform-density fixed-point iteration (`--method odt-fixed-point`)     |                  uniform-density BFGS (`--method odt-bfgs`)                   |

Optimal Delaunay Triangulation (ODT) as suggested by [Chen and
Holst](#relevant-publications). Typically superior to CPT, but also more expensive to
compute.

Implemented once classically as a fixed-point iteration, once as a nonlinear
optimization method. The latter typically leads to better results.

### Using optimesh from Python

You can also use optimesh in a Python program. Try

<!--pytest-codeblocks:skip-->

```python
import optimesh

# [...] create points, cells [...]

points, cells = optimesh.optimize_points_cells(
    points, cells, "CVT (block-diagonal)", 1.0e-5, 100
)

# or create a meshplex Mesh
import meshplex

mesh = meshplex.MeshTri(points, cells)
optimesh.optimize(mesh, "CVT (block-diagonal)", 1.0e-5, 100)
# mesh.points, mesh.cells, ...
```

If you only want to do one optimization step, do

<!--pytest-codeblocks:skip-->

```python
points = optimesh.get_new_points(mesh, "CVT (block-diagonal)")
```

### Surface mesh smoothing

optimesh also supports optimization of triangular meshes on surfaces which are defined
implicitly by a level set function (e.g., spheres). You'll need to specify the function
and its gradient, so you'll have to do it in Python:

```python
import meshzoo
import optimesh

points, cells = meshzoo.tetra_sphere(20)


class Sphere:
    def f(self, x):
        return 1.0 - (x[0] ** 2 + x[1] ** 2 + x[2] ** 2)

    def grad(self, x):
        return -2 * x


# You can use all methods in optimesh:
points, cells = optimesh.optimize_points_cells(
    points,
    cells,
    "CVT (full)",
    1.0e-2,
    100,
    verbose=False,
    implicit_surface=Sphere(),
    # step_filename_format="out{:03d}.vtk"
)
```

This code first generates a mediocre mesh on a sphere using
[meshzoo](https://github.com/nschloe/meshzoo/),

<img src="figs/tetra-sphere.png" width="20%">

and then optimizes. Some results:

| ![odt-dp-fp](figs/sphere-cpt.webp) | ![odt-uniform-fp](figs/sphere-odt.webp) | ![odt-uniform-bfgs](figs/sphere-cvt.webp) |
| :--------------------------------------------------------------: | :-------------------------------------------------------------------: | :---------------------------------------------------------------------: |
|                               CPT                                |                                  ODT                                  |                           CVT (full Hessian)                            |

### Which method is best?

From practical experiments, it seems that the CVT smoothing variants, e.g.,

```
optimesh in.vtk out.vtk -m cvt-uniform-qnf
```

give very satisfactory results. (This is also the default method, so you don't need to
specify it explicitly.) Here is a comparison of all uniform-density methods applied to
the random circle mesh seen above:

<img src="figs/comparison.svg" width="90%">

(Mesh quality is twice the ratio of incircle and circumcircle radius, with the maximum
being 1.)

### Why optimize?

| <img src="figs/gmsh.png" width="80%"> | <img src="figs/gmsh-optimesh.png" width="80%"> | <img src="figs/dmsh.png" width="80%"> |
| :-----------------------------------------------------------------: | :--------------------------------------------------------------------------: | :-----------------------------------------------------------------: |
|                              Gmsh mesh                              |                           Gmsh mesh after optimesh                           |               [dmsh](//github.com/nschloe/dmsh) mesh                |

Let us compare the properties of the Poisson problem (_Δu = f_ with Dirichlet boundary
conditions) when solved on different meshes of the unit circle. The first mesh is the
on generated by [Gmsh](http://gmsh.info/), the second the same mesh but optimized with
optimesh, the third a very high-quality [dmsh](https://github.com/nschloe/dmsh) mesh.

We consider meshings of the circle with an increasing number of points:

| ![gmsh-quality](figs/gmsh-quality.svg) | ![gmsh-cond](figs/gmsh-cond.svg) | ![gmsh-cg](figs/gmsh-cg.svg) |
| :------------------------------------------------------------------: | :------------------------------------------------------------: | :--------------------------------------------------------: |
|                         average cell quality                         |             condition number of the Poisson matrix             |           number of CG steps for Poisson problem           |

Quite clearly, the dmsh generator produces the highest-quality meshes (left).
The condition number of the corresponding Poisson matrices is lowest for the high
quality meshes (middle); one would hence suspect faster convergence with Krylov methods.
Indeed, most CG iterations are necessary on the Gmsh mesh (right). After optimesh, one saves
between 10 and 20 percent of iterations/computing time. The dmsh mesh cuts the number of
iterations in _half_.

### Access from Python

All optimesh functions can also be accessed from Python directly, for example:

<!--pytest-codeblocks:skip-->

```python
import optimesh

X, cells = optimesh.odt.fixed_point(X, cells, 1.0e-2, 100, verbose=False)
```

### Installation

optimesh is [available from the Python Package
Index](https://pypi.org/project/optimesh/), so simply do

```
pip install optimesh
```

to install.

### Relevant publications

- [Qiang Du, Vance Faber, and Max Gunzburger, _Centroidal Voronoi Tessellations: Applications and Algorithms_,
  SIAM Rev., 41(4), 1999, 637–676.](https://doi.org/10.1137/S0036144599352836)

- [Yang Liu et al., _On centroidal Voronoi tessellation—Energy smoothness and fast computation_,
  ACM Transactions on Graphics, Volume 28, Issue 4, August 2009.](https://dl.acm.org/doi/10.1145/1559755.1559758)

- [Long Chen, Michael Holst, _Efficient mesh optimization schemes based on Optimal Delaunay Triangulations_,
  Comput. Methods Appl. Mech. Engrg. 200 (2011) 967–984.](https://doi.org/10.1016/j.cma.2010.11.007)

### Testing

To run the optimesh unit tests, check out this repository and type

```
pytest
```

### License

This software is published under the [GPLv3 license](https://www.gnu.org/licenses/gpl-3.0.en.html).
