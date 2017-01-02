# runlmc

Do you like to apply Bayesian nonparameteric methods to your regressions? Are you frequently tempted by the flexibility that kernel-based learning provides? Do you have trouble getting structured kernel interpolation or various training conditional inducing point approaches to work in a non-stationary multi-output setting?

If so, this package is for you.

**runlmc** is a Python 3 package designed to extend structural efficiencies from _Scalable inference for structured Gaussian process models_ (Staaçi 2012) and _Thoughts on Massively Scalable Gaussian Processes_ (Wilson et al 2015) to the non-stationary setting of linearly coregionalized multiple-output regressions. For the single output setting, MATLAB implementations are available [here](http://www.gaussianprocess.org/gpml/code/matlab/doc/).

In other words, this provides a matrix-free implementation of multi-output GPs for certain covariances. As far as I know, this is also the only matrix-free implementation for single-output GPs in python.

## Usage Notes

* Currently, I'm only supporting 1 input dimension.

## A note on GPy

[GPy](https://github.com/SheffieldML/GPy) is a way more general GP library that was a strong influence in the development of this one. I've tried to stay as faithful as possible to its structure.

I've re-used a lot of the GPy code. The main issue with simply adding my methods to GPy is that the API used to interact between GPy's `kern`, `likelihood`, and `inference` packages centers around the `dL_dK` object, a matrix derivative of the likelihood with respect to covariance. The materialization of this matrix is the very thing my algorithm tries to avoid for performance.

If there is some quantifiable success with this approach then integration with GPy would be a reasonable next-step.

## Example

    def noisify(x, sd): return x + np.rand.normal(0, sd)
    Xs = [np.arange(-10, 5, 0.5), np.arange(-5.25, 10, .5)]
    Xs = [[noisify(x, 0.1) for x in ls] for i, ls in enumerate(Xs, 1)]
    Ks = [runlmc.kern.StdPeriodic() for _ in range(2)]
    Ys = runlmc.util.Sample(Xs, Ks, added_noise=[1,0.3])
    m = runlmc.models.LMC(Xs, Ys,ranks=[1,1],kernels_list=Ks)
    m.optimize()
    
    xticks = np.arange(-10, 10, 0.01)
    mus, _ = m.predict([xticks, xticks])
    los, his = m.predict_quantiles([xticks, xticks])
    
    %matplotlib inline
    import matplotlib.pyplot as plt
    for i in range(2)
        print("Output", i)
        for outs in [lo, mu, hi]: plt.plot(xticks, outs[i])
        plt.plot(Xs[i], Ys[i])
        plt.show()
        
TODO: show resulting image
        
## Benchmarks

Make sure that the directory root is in the `PYTHONPATH` when running the benchmarks. E.g., from the directory root:

    PYTHONPATH=. python benchmark/inversion.py
        
## Dev Stuff

Required packages for development (Python 3 versions): `pylint nose paramz gpy sphinx contexttimer`.

All below invocations should be done from the repo root.
 
| Command           | Purpose  |
| ----------------- | -------- |
| `./style.sh`      | Check style with pylint, ignoring TODOs and locally-disabled warnings. |
| `./docbuild.sh`   | Regenerate docs (index will be in `doc/_generated/_build/runlmc.html`) |
| `nosetests -l DEBUG`       | Run unit tests |

### Roadmap

0. Put image into this README (link to ipynb in `examples/`)
0. Review docs in general
0. Rank > 1 coregionalization
0. create an LMC test for checking a no-covariance multioutput case detected (requires rank-2 kernel to learn the identity matrix as its coregionalization). Similarly for a with-covariance. Both cases should be 2-output, single kernel, non-noisy.
0. Add StdPeriodic, Matern kernel (add a PSD test)
0. np.linalg.eigvalsh -> scipy.linalg.eigvalsh
0. Figure out eigenvalue issues - ; log
0. Benchmark/evaluate reconstruction error for K (on various example kernels)
0. Benchmark/evaluate reconstruction error for log likelihood
0. Write up the current algorithm (PDF)
0. Model learning
    * derivative-free opt first -> common bottlenecks?
    * numerical derivative opt -> numerical derivation class (use MAT321 method)
    * derivatives -> hopefully it doesn't come to this, lot more work.
0. Add tests to verify gradient (for a particular model, with and without prior, use the MAT321 method)
0. Beat GPy; iterate for speed.
0. TODO(PAPER) - add references to paper (in README, too)

### Considerations 

* SEED=3333617092 nosetests runlmc.models.test_lmc breaks Toeplitz PSD strictly (currently has large negative eigval cutoff)
* SLFM approach -> can we take determeinant in this representation?
   0. SLFM approach work for computing deriv of log det / log det exactly (pg. 16 in vector-valued-lmc.pdf)
   0. How to take determinant? Derivatives?
   0. Re-prove (legitimately); start by showing wilson SKI m^(-3) conv (in multioutput case), then prove SLFM for 1 input dim, rank 1
   0. Rank >1 reduction to rank 1 (use constant kq terms)
   0. multidimensional proof; requires cubic interpol from SKI (again, for multioutput)
   0. SLFM code up; GP code up; do K, dK/dL reconstruction experiments.
* TODO(MSGP) - fast toeplitz eig
* MINRES or LCG?
* How can we add good preconditioners? How much do they help?
* What are condition numbers in practice?
* Why are sparse eigensolvers poor? Can we use them as an accurate general-purpose solution if all else fails?
* Consider other approximate inverse algorithms: see Thm 2.4 of [Agarwal, Allen-Zhu, Bullins, Hazan, Ma 2016](https://arxiv.org/abs/1611.01146)
0. New logdet algo? [Chebyshev-Hutchinson](https://arxiv.org/abs/1503.06394) [Code](https://sites.google.com/site/mijirim/logdet)

### Low-priority Tasks

0. rename `rand_psd` -> `rand_pd`
0. Continuous integration for unit tests
0. Drop gpy dep (in non-tests) - requires exact kernel cholesky impl
0. TODO(priors) - Incorporating priors (e.g., three-parameter beta) - add tests for priored versions of classes, some tests in parameterization/ (priors should be value-cached, try to use an external package)
0. multidimensional inputs and ARD.
0. product kernels (multiple factors) and active dimensions

### Thesis Plan

0. Intro
0. Related Work - see detailed-plan.md; proposal; close loose ends here
0. SKI - prove O(m^-3) formalism (one-input case)
0. runlmc theoretical kernel re-creation error (can we re-prove? what are the bounds) (one-input case) (SLFM approach, if viable)
0. experimental proof of above
0. algorithm for runlmc kernel; explain/prove fast structural runtimes
0. experimental proof for above; comparison to gpy exact/ssgp
0. test lmc._autogrid for edge cases.
0. test lmc kernel reconstruction, quadratic with ExactAnalogue with different gaussian noise (not all 1s)
0. test `LMC._raw_predict` unit testing, by using K_SKI() and anlogous math
