_(here's a Markdown-enabled version of manpage.txt which describes InvGN...)_  
##INVGN - Gauss-Newton inversion - version 1.0.  
_by Andrew Ganse, Applied Physics Laboratory, University of Washington, Seattle._  
_[andy@ganse.org](mailto:andy@ganse.org)_  
_Copyright (c) 2015, University of Washington, via 3-clause BSD license._  
_See LICENSE.txt for full license statement._  

-------------------------------------------------------------------------------

INVGN calculates Tikhonov-regularized, Gauss-Newton nonlinear iterated inversion
to solve the following damped nonlinear least squares problem:  

          minimize ||g(m)-d||^2_2 + lambda^2||Lm||^2_2  

For appropriate choices of regularization parameter lambda, this problem is
equivalent to:  

          minimize ||Lm||_2 subject to ||g(m)-d||_2<delta  
          (where delta is some statistically-determined noise threshold)  

and also to:  

          minimize ||g(m)-d||_2 subject to ||Lm||_2<epsilon  
          (where epsilon is some model-norm threshold)  
          
The function call is set up to allow use on both nonlinear and linear problems,
both regularized (inverse) and non-regularized (parameter estimation) problems,
and both frequentist and Bayesian problems.  See EXAMPLES below for various calls.
The dampled NLS regularization is accomplished with the L-curve method (see e.g.
Hansen 1987).

This code is based on material from:  

* Professor Ken Creager's ESS-523 inverse theory class, Univ of Washington, 2005  
* RC Aster, B Borchers, & CH Thurber, "Parameter Estimation and Inverse
    Problems," Elsevier Academic Press, 2004.  
* A Tarantola, "Inverse Problem Theory and Methods for Model Parameter
    Estimation", Society for Industrial and Applied Mathematics, 2005.  
* PC Hansen, "Rank-Deficient and Discrete Ill-Posed Problems: Numerical Aspects
    of Linear Inversion," Society for Industrial Mathematics, 1987.  
* PE Gill, W Murray, & MH Wright, "Practical Optimization," Academic Press, 1981.  

INVGN is compatible with [Octave](http://www.octave.org) (version >3), but to handle a few minor
remaining differences between Matlab and Octave, do note the "usingOctave"
flag among the options listed at top of the InvGN.m script.  Perhaps in future
versions INVGN will allow these options to be set via keyword/value pairs in the
input argument list, but not implemented yet.

<br>
###USAGE:

```
 [m,normdm,normmisfit,normrough,lambda,dpred,G,C,R,Ddiag,i_conv] = ...
 invGN(fwdfunc,derivfunc,epsr,dmeas,covd,lb,ub,m0,L,lambda,maxiters,useMpref,...
       verbose,fid,vargin);
```

<br>
###INPUTS:  (all vectors are always column vectors)  

  **`fwdfunc`** = Matlab/Octave function handle to forward problem function which
              is of form outvector=myfunction(invector,...).  (See example 1
              of "help function_handle" in Matlab for how to implement)  
**`derivfunc`** = Matlab/Octave function handle to derivatives function which
              is of form outvector=myfunction(invector,...).
              If derivfunc=[] then finitediffs are automatically used instead.  
     **`epsr`** = relative accuracy of forward problem computation (not the
              accuracy of the modeling of the physical process, but of the
              computation itself, see e.g. Gill,Murray,Wright 1981).  Affects
              convergence testing and is passed into finite diff function (if
              used), in both cases providing stepsize h (h is about sqrt(epsr)
              if the model params are all of order unity).  Epsr has a minimum
              at the tradeoff between degrading Talor series expansion accuracy
              if h too large, and increasing roundoff error in the numerator
              G(m+h)-G(m) of finite diffs if h too small.
              Epsr can be scalar (ie same for all elements of model vector) or 
              vector of same length as model vector for 1-to-1 correspondence
              to different variable "types" (eg layer bulk properties vs layer
              thicknesses).  
    **`dmeas`** = measured data vector (includes noise)  
     **`covd`** = if sqr matrix, then this is measurement noise covariance matrix.
              if col vector, then this is diag of meas noise covariance matrix.
              if neg, then = -covd^(-1/2), allowing e.g. use of zeros for a mask,
              so note in that case it's the inverse of the STDEVS (sqrt(cov)).  
       **`lb`** = lower model bounds vector (hard bnds, no tolerance)  
       **`ub`** = upper model bounds vector (hard bnds, no tolerance)
              These bounds are only implemented at the iteration steps after
              solving for the model perturbation without them; ie functions
              such as BVLS or LSQNONNEG are not used.  If the bounds are seen
              to be passed in a candidate model perturbation, the stepsize of
              the perturbation is reduced (but retaining its direction) until
              within bounds.  As long as we know the final solution must be
              within the bounds and not on them, this is merely like changing
              the starting point of inversion and is valid (perhaps somtimes
              not as efficient as one of those more complicated functions).
              But if the solution from this code ends up on a bound, note you
              should treat that solution with some caution and not consider it
              a true GN solution.  In my own inverse problems using this code
              the bounds only ever engage at the tails of my Lcurve where I'm
              not too worried about being rigorous, so I haven't found it
              necessary to complicate things with those other functions.  
       **`m0`** = initial model vector estimate to start iterating from; if this
              is instead a matrix (NxM), then it's a collection of M initial
              model vectors of length N each, corresponding to M lambda values
              (Typically this is done to continue iterations on previous runs
              without starting over.)
              Alternate use: if L (next input arg) = I (identity matrix), then
              this is also the "preferred model" whose distance from the soln
              model is minimized.  
        **`L`** = If this is a matrix, then this is the user-supplied finite
              difference operator for Tikhonov regularization (function
              finiteDiffOP.m can help construct it) in the frequentist
              framework.  If in the Bayesian framework and lambda is set to 1,
              then L can be supplied as the Cholesky decomposition of the
              inverse model prior covariance matrix.
              If L is scalar then it is the number of profile segments (for
              example if the model vector represents concatenated P & S wave
              speed profiles then Lin=2) and 2nd order Tikhonov (smoothness)
              frequentist regularization will be used.  
   **`lambda`** = Used for the frequentist inversion framework (if using Bayesian
              framework then set lambda=1 and see Bayesian note for L above).
              If lambda is a negative scalar, auto calculate Lcurve with num
              pts equal to magnitude of the negative scalar.  The auto-calc'd
              curve starts at lambda_mid = sqrt(max(eig(G'*G))/max(eig(L'*L)))
              and decreases logarithmically from there to 3 orders of mag
              less and more than lambda_mid by default.  If lambda >=0, use
              its value(s) directly as frequentist Tikhonov tradeoff param(s),
              in which case lambda can be scalar or vector.
              If lambda is a 3-element vector, lambda(1) is the pos or neg scalar
              just described; and lambda(2:3) are how many orders of magnitude
              greater (lambda(2)) or less (lambda(3)) than lambda_mid to sweep
              in the auto-calc'd curve.  (lambda(2:3) defaults are 3,-3)  
 **`useMpref`** = flag for one of three problem frameworks:
              useMpref=0 is for a frequentist formulation with higher-order
                Tikhonov regularization.  Here the regularization doesn't
                handle distance from preferred (initial) model m0, but no
                restriction on L (unlike useMpref=1).
              useMpref=1 is for using 0th-order-Tikh (ridge regr), ie
                L==eye(size(L)), in a frequentist formulation in which m0 is
                the "preferred model" the inversion minimizes distance to.
              useMpref=2 is for using Bayesian formulation, in which case:
                m0 is the prior model mean, lambda must =1, L=chol(inv(Cprior))
                (user must set that L externally, note ironically L there must
                be RIGHT triangular, the "L" notation is from frequentist case),
                output C is now the Bayesian posterior covariance matrix (which
                note is different from the frequentist C), outputs R and Ddiag
                are empty, and normrough won't have literal meaning anymore.  
 **`maxiters`** = Number of GN iterations to perform at each lambda on the Lcurve.
              So total number of derivative estimations will be
              numlambdas*maxiters.  
  **`verbose`** = 0-3, with 0 = run quietly, and 3 = lots of progress text
              Progress text (percent done) in findiffs is useful for slow
              forward problems and/or debugging - but note percent done text
              is unavailable in jacfwddist and jaccendist.  
      **`fid`** = File handle to which progress text is output in verbose option.
              Note fid=1 outputs to stdout, but an output file is useful for
              long batch runs when the forward problem is slow.  
    **`vargin`** = the extra arguments (if any) that must be passed to the fwdprob
              function are tacked onto the end here.  You may have none, in
              which case your last arg is the fid.)  
<br>
###OUTPUTS:  (all vectors are always column vectors)  

  **`m`** = matrix of maximum likelihood model solution vectors for each
              lambda.  So m is MxP for M model params and P points on Lcurve,
              ie each column of m is a model vector for a different lambda.  
   **`normdm`** = vectors of L2 norms of model perturbation steps "dm" for each
              lambda.  (so normdm is imax x P, with i steps to convergence
              for l'th lambda, imax=max(i(l)), and P points on Lcurve).  Unused
              elements in normdm are set to -1, since norms can't be negative.  
**`normmisfit`** = vector of L2 norms of data misfit between measured and 
              predicted data at solution point for each lambda.  
**`normrough`** = vector of L2 norms of roughness (for which units are 2nd deriv)
              at solution points for each lambda.  "Roughness" refers to case
              of 2nd order Tikhonov (smoothing constraint) but note really here
              this norm's meaning depends on what the supplied L matrix is
              (eg might be other model norms than roughness).  And in Bayesian
              case (useMpref=2) this doesn't have same meaning anymore although
              it's still part of the objective function as it relates to Cprior.  
   **`lambda`** = vector of tradeoff parameters used for Tikhonov regularization.
              (you might already have this if you passed in vector of lambdas
              as one of the input arguments, but this is useful in the case of
              specifying to auto-calculate N Lcurve points).  
    **`dpred`** = predicted data at solution models (so there are as many vectors
              in dpred as there are points on the Lcurve, so dpred is NxP, for
              N data pts and P points on Lcurve).  
        **`G`** = Jacobian matrices of partial derivatives at solution models
              (so there are as many jacobian matrices in G as there are points
              on the Lcurve, so G is NxMxP, for N data pts, M model params,
              and P points on Lcurve).  
        **`C`** = if useMpref<2, C contains the frequentist covariance matrices of
              solution models (so there are as many cov matrices in C as there
              are points on the Lcurve, so C is MxMxP, for M model params and
              P points on Lcurve).  If useMpref==2 (flagging Bayesian case),
              then C is instead the Bayesian posterior model covariance matrix.  
        **`R`** = resolution matrices of solution models (so there are as many cov
              matrices in C as there are points on the Lcurve, so C is MxMxP,
              for M model params and P points on Lcurve).
              (R is empty for Bayesian case when useMpref=2.)  
    **`Ddiag`** = column vector of diagonal of data res matrix (length Ndata)
              (Ddiag is empty for Bayesian case when useMpref=2.)  
    **`iconv`** = column vector of number of iterations before convergence at each
              lambda.  if didn't converge, the iconv value for that lambda
              is 0.  (length Ndata).  

For more clarity in the code, this function computes its model perturbations via
(regularized) normal equations, rather than generalized SVD or subiterations of
gradient descent, which may be more efficient and/or scalable.  The assumption
in this choice was that forward function code takes much longer to compute than
the model perturbation solution itself (which can otherwise be computed faster
via GSVD).  If that assumption is not true, then much improvement in compute
time of this function would result from solving for the perturbation using GSVD
methods such as those provided in Per Christian Hansen's toolkit (however note
that toolkit would take a lot of tweaking/paring to make it Octave compatible):
http://www2.imm.dtu.dk/~pch/Regutools  
Aside from computational efficiency/speed, another important consequence of
solving normal equations in this code is that it constrains the size of the
problem (number of model parameters) relative to amount of computer memory due
to calculation of a matrix inverse.  If useful to note I have successfully run
this code for over 10,000 model parameters on a computer with 16GB of RAM, and
for over 2500 model parameters on a computer with 8GB RAM (in both those cases
the matrix inverse was the memory constraint rather than forward problem).

<br>
###EXAMPLES:  
(Leaving off the output part `[var1,var2,etc]=` which is same format for all...)

**Calling invGN with forward problem function `fwdp` and derivatives function `derivp`:**  

    invGN(fwdp,derivp,epsr,datameas,...

**Calling invGN to calculate first finite differences of `fwdp` internally:**

    invGN(fwdp,[],epsr,datameas,...

**Regularized frequentist inversion with auto-chosen lambdas (numlambdas of them):**  

    invGN(fwdp,[],epsr,datameas,-invcovd,lb,ub,minit,L,-numlambdas,0,maxiters,3,1);

**Regularized frequentist inversion with prechosen lambdas:**  

    invGN(fwdp,[],epsr,datameas,-invcovd,lb,ub,minit,L,...
    [1.04193e+07; 10419.3; 10.4193; 0.0104193;],0,maxiters,3,1);

**Regu inv, picking up after previous run (using lambda vector from prev run):**  

    invGN(fwdp,[],epsr,datameas,-invcovd,lb,ub,minit,L,lambda,0,maxiters,3,1);

**Parameter estimation (no regularization):**  

    invGN(fwdp,[],epsr,datameas,-invcovd,lb,ub,minit,zeros(size(L)),1,0,maxiters,3,1);

**Frequentist nonlinear problem with preferred model:**  

    invGN(fwdp,[],epsr,datameas,covd,lb,ub,mpref,eye(M),-numlambdas,1,maxiters,0,1);

**Bayesian nonlinear problem:**  

    invGN(fwdp,[],epsr,datameas,covd,lb,ub,mprior,chol(inv(Cprior)),1,2,maxiters,0,1);

**Frequentist linear problem (should converge after 1st iter), w/ smoothing regu:**  

    deriv=@(x) A;  fwdp=@(x) A*x;  % define funcs using the A matrix inline  
    invGN(fwdp,deriv,epsr,datameas,covd,lb,ub,zeros(M,1),L,-numlambdas,0,2,0,1);

(Note `maxiters=2` there: actually the first "iter" in saved results is for the
initial value, which admittedly isn't relevant for purely linear problems, but
that's how this nonlinear script fits purely linear problems.  Yes absolutely
an inefficient kludge for the linear case.)

