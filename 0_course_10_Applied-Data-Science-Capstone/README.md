# Jarrow-Rudd binomial tree

## Introduction
The Jarrow-Rudd binomial tree is a method used for option pricing, similar to the Cox-Ross-Rubinstein (CRR) binomial tree, but with some differences in the way up and down movements of the underlying asset are calculated. The Jarrow-Rudd model uses the actual probability measure (as opposed to the risk-neutral measure in CRR) and is used for both European and American options.

Input Parameters:
- `S0`: Initial stock price.
- `K`: Strike price.
- `T`: Time to maturity in years.
- `r`: Risk-free interest rate.
- `sigma`: Volatility.
- `N`: Number of time steps.
- `option_type`: 'call' for a call option, 'put' for a put option.

Function Parameters:
- `dt`: Time step size.
- `u`, `d`: Up and down factors.
- `p`: Probability of an up movement.
- `asset_prices`: Matrix to store the underlying asset prices at each node.
- `option_values`: Matrix to store the option values at each node.

Algorithm:
- Asset Price Calculation: Populate the asset_prices matrix.
- Option Value Calculation: Populate the option_values matrix at maturity and work backward to the present value.

## Original Profiling (`Orig`)
### Terminal
Initial build
```sh
module load python
python assignment2.py
```

Original result:
```sh
7.977846506434156
10.458192318299641
13.284100601456002
16.433817883712898
19.87406305755777
15.022460609801088
12.502809796584085
10.328721454657945
8.478442111832365
6.918690660594717
Elapsed time: 10.68949 seconds
```

### cProfile 
Profiling using `cProfile`
```sh
python -m cProfile -s cumulative assignment2.py
```

Detailed profiling using `cProfile`:
```sh
[chuanyin@midway3-login4 Assignment2_chuanyin]$ python -m cProfile -s cumulative assignment2.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 7.977846506434156
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 10.458192318299641
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 13.284100601456002
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 16.433817883712898
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 19.87406305755777
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 15.022460609801088
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 12.502809796584085
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 10.328721454657945
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 8.478442111832365
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 6.918690660594717
Elapsed time: 10.40711 seconds
         86343 function calls (84378 primitive calls) in 10.618 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    111/1    0.000    0.000   10.618   10.618 {built-in method builtins.exec}
        1    0.003    0.003   10.617   10.617 assignment2.py:1(<module>)
       10    0.000    0.000   10.403    1.040 assignment2.py:55(jarrow_rudd_binomial_tree)
       10    6.957    0.696    6.957    0.696 assignment2.py:38(backtrack_option_values)
       10    3.424    0.342    3.426    0.343 assignment2.py:4(populate_asset_prices)
       14    0.001    0.000    0.560    0.040 __init__.py:1(<module>)
       10    0.014    0.001    0.020    0.002 assignment2.py:21(initialize_option_values)
```

Report shows:
1. total time and total number of function calls
1. most of the time spent in `backtrack_option_values()`
1. some time spent in `populate_asset_prices()`
1. least time spent in `initialize_option_values()`

### Line profiler 
Profiling using `line_profiler`. In the code, use profile decorator to indicate the function you want to profile:
```sh
pip install line_profiler
~/.local/bin/kernprof -l -v assignment2.py 
```

Line Profiler shows the following expensive lines:
1. 
```sh
Elapsed time: 21.70014 seconds
Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     4                                           @profile
     5                                           def populate_asset_prices(S0, u, d, N):
    17   5025000     662897.2      0.1     12.7          for j in range(i + 1):
    18   5015000    4555666.4      0.9     87.2              asset_prices[j, i] = S0 * (u ** (i - j)) * (d ** j)
Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    22                                           @profile
    23                                           def initialize_option_values(asset_prices, K, N, option_type='call'):
    33     10010       1530.5      0.2      6.0          if option_type == 'call':
    34      5005      11047.3      2.2     43.3              option_values[j, N] = max(0, asset_prices[j, N] - K)
    35                                                   else:
    36      5005      10537.0      2.1     41.3              option_values[j, N] = max(0, K - asset_prices[j, N])
Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    40                                           @profile
    41                                           def backtrack_option_values(option_values, r, dt, p, N):
    52   5015000     787410.5      0.2      7.2          for j in range(i + 1):
    53  10010000    7960330.1      0.8     73.2              option_values[j, i] = np.exp(-r * dt) * (p * option_values[j, i + 1] 
    54   5005000    2131466.8      0.4     19.6                                                       + (1 - p) * option_values[j + 1, i + 1])    
```

## Optimization 1 (`Cython1`)
Changes
- Python to Cython
- use static C types such as float, unsigned int...

Cython file structure:
- `assignment2.pyx`: Cython file containing functions to be optimized
- `setup.py`: build script, similar to a Makefile
- `test_binomial_tree.py`: Python file calling the functions

To compile Cython:
```sh
python setup.py build_ext --inplace
```

To run:
```sh
python test_binomial_tree.py
```

### Terminal
```sh
Result:
[chuanyin@midway3-login4 Cython]$ python test_binomial_tree.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 7.978293921101031
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 10.45874511451189
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 13.284765352868641
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 16.43457260029976
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 19.874937525823952
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 15.021904426547158
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 12.502302070822417
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 10.328270503682935
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 8.478025457998397
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 6.918337361920475
Elapsed time: 0.18826 seconds
```

## Optimization 2 (`Cython2`)
Improvements:
- Python function simplification.
- precalculate factor
- simplify expensive computations such as exponentition, max into if/else

### Terminal
N = 10000 result to benchmark against:
```sh
[chuanyin@midway3-login3 Cython2]$ python test_binomial_tree.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 7.979965945930158
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 10.460386800895566
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 13.289558888601151
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 16.440004668890083
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price: 19.88111687526132
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 15.015491788135076
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 12.495411036201373
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 10.32408218654753
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 8.474025488280555
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price: 6.9146358421462635
Elapsed time: 26.17402 seconds
```


```sh
[chuanyin@midway3-login4 Cython2]$ python test_binomial_tree.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  7.981543198341405
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  15.01845962293404
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  10.462454312262912
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  12.497880773205276
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  13.292185590136386
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  10.32612275710044
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  16.44325406082741
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  8.47570039231207
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  19.885046408262117
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  6.916002530441519
Elapsed time: 20.96592 seconds
```


## Optimization 3 (`Cython3`)
Parallel work
- use parallel range (prange) operation is used to run code in parallel.
- use numpy arrays which are already optimized
- double for loops seem to have problems, but optimize on N

### Terminal
results from parallelizing populate_asset_prices()
```sh
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  7.981543198341405
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  15.01845962293404
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  10.462454312262912
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  12.497880773205276
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  13.292185590136386
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  10.32612275710044
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  16.44325406082741
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  8.47570039231207
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  19.885046408262117
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  6.916002530441519
Elapsed time: 8.37642 seconds
```

results from also parallelizing initialize_option_values()

```sh
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  7.981543198341405
S = 90, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  15.01845962293404
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  10.462454312262912
S = 95, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  12.497880773205276
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  13.292185590136386
S = 100, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  10.32612275710044
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  16.44325406082741
S = 105, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  8.47570039231207
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 call price:  19.885046408262117
S = 110, r = 0.03, v = 0.3, T = 1, N = 10000 put price:  6.916002530441519
Elapsed time: 8.16753 seconds
```

```sh
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  7.978440845097185
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  15.022181061909295
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  10.45893771717133
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  12.502532306534398
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  13.285009998006178
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  10.328460703591706
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  16.434875250603874
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  8.478181584783407
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  19.87530353204955
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  6.918464766323219
Elapsed time: 1.02993 seconds
```

- schedule static for above functions

```sh
[chuanyin@midway3-login4 Cython3]$ python test_binomial_tree.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  7.978440845097185
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  15.022181061909295
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  10.45893771717133
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  12.502532306534398
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  13.285009998006178
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  10.328460703591706
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  16.434875250603874
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  8.478181584783407
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  19.87530353204955
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  6.918464766323219
Elapsed time: 0.30585 seconds
```

results from vectorizing backtrack_option_values()
- didn't implement parallelization due to data dependency
```sh
[chuanyin@midway3-login4 Cython3]$ python test_binomial_tree.py 
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  7.978440845097185
S = 90, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  15.022181061909295
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  10.45893771717133
S = 95, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  12.502532306534398
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  13.285009998006178
S = 100, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  10.328460703591706
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  16.434875250603874
S = 105, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  8.478181584783407
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 call price:  19.87530353204955
S = 110, r = 0.03, v = 0.3, T = 1, N = 1000 put price:  6.918464766323219
Elapsed time: 0.66481 seconds
```
