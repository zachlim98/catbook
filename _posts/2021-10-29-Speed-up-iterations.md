---
title: "A 300x speed boost when iterating data? Yes please!"
categories: [data, trading]
---

# Introduction

One of the challenges that one faces when backtesting is having to iterate through rows. Unlike other operations where vectorization is a possibility and brings significant speed gains, one is constrained when backtesting because one typically requires the values of the previous row in order to calculate the next row. This forces you to have to iterate through the data which, as we know, is incredibly slow. 

I recently needed to perform an iteration over a dataset with about 15 years of options data. Granted, this was not the largest dataset that I'd ever seen but it was still a sizable one. I hence decided to explore and compare three different ways to iterate through it 

### tl;dr

[Numba](https://numba.pydata.org) is the greatest invention on earth. If you can use arrays and numpy to perform your iteration, use numba. 

## Preparing the data

```python
import plotly.express as px
import pandas as pd
import numpy as np
import numba

long_data = pd.read_csv("long.csv")
long_data["Open"] = long_data["Open"].str.replace('$',"").astype(float)
long_data["Close"] = long_data["Close"].str.replace('$',"").astype(float)
```

Nothing much to see here really - just cleaning up the data a little by removing the dollar signs and converting everything to floats. 

The main thing that we would be testing is iterating over the dataframe and closing and open trades, calculating the PnL and resultant change in account size. The account size would then determine the lot size of the next trade - hence vectorization is not possible since each subsequent trade's PnL depends on the lot size which depends on the change in account size from the previous trade. 

### Method 1: `Itertuples`

A cursory search on StackExchange shows that `itertuples` is the method that's typically thrown up when one asks a question about iterating through a dataframe. This is faster than `iterrows` because under the hood, pandas converts each row into a tuple instead of a pandas series which makes accessing the data much faster. I also used a tip that I read somewhere which was to set `name=None` in order to create unnamed tuples (which has been shown to provide a speed boost).

```python
def via_iter(long_data):
    acc_size = 100000
    comm = 1.20
    pcr = 0.27
    tgt_return = 0.20
    size = [acc_size]
    pnl = [(long_data["Open"][0]+long_data["Close"][0])*100*(acc_size * tgt_return / pcr / 252 / 100 / long_data["Open"][0]) - (comm*2)]
    for row in long_data[1:].itertuples(name=None):
        size.append(size[row[0]-1]+pnl[row[0]-1])
        pnl.append((long_data.at[row[0]-1, 'Open']+long_data.at[row[0]-1, 'Close'])*100*(size[row[0]] * tgt_return / pcr / 252 / 100 / long_data.at[row[0]-1, 'Open']) - (comm*2))

    return pnl, size
```

### Method 2: Looping Arrays

Here, we get rid of the clunky pandas dataframe and instead use a numpy array. Unforunately, due to the way I needed to backtest this, vectorization was not an option and neither were there any numpy ufuncs that I could use. The good ol' for loop would have to do.

```python
def arr_no_numba(calc_data):
    acc_size = 100000
    comm = 1.20
    pcr = 0.27
    tgt_return = 0.20
    calc_data[0][2] = acc_size
    calc_data[0][3] = (calc_data[0][0]+calc_data[0][1])*100*(acc_size * tgt_return / pcr / 252 / 100 / calc_data[0][0]) - (comm*2)
    for idx in range(1, len(calc_data)):
        calc_data[idx][2] = calc_data[idx-1][2] + calc_data[idx-1][3]
        calc_data[idx][3] = (calc_data[idx-1][0]+calc_data[idx-1][1])*100*(calc_data[idx][2] * tgt_return / pcr / 252 / 100 / calc_data[idx-1][0]) - (comm*2)
    return calc_data
```

### Method 3: Numba

Finally, I decided to add on numba to the previous array for loop. Numba speeds up Python code by essentially translating certain Python and Numpy code into machine code(!!) at runtime. This allows it to achieve significantly higher speeds. As an added bonus, its super simple to use - just throw on the `@numba.jit` decorator and you're good to go!

```python
import numba

@numba.jit(nopython=True)
def via_numba(calc_data):
    acc_size = 100000
    comm = 1.20
    pcr = 0.27
    tgt_return = 0.20
    calc_data[0][2] = acc_size
    calc_data[0][3] = (calc_data[0][0]+calc_data[0][1])*100*(acc_size * tgt_return / pcr / 252 / 100 / calc_data[0][0]) - (comm*2)
    for idx in range(1, len(calc_data)):
        calc_data[idx][2] = calc_data[idx-1][2] + calc_data[idx-1][3]
        calc_data[idx][3] = (calc_data[idx-1][0]+calc_data[idx-1][1])*100*(calc_data[idx][2] * tgt_return / pcr / 252 / 100 / calc_data[idx-1][0]) - (comm*2)
    return calc_data
```

## Testing time

To use my array and Numba functions, I had to convert my dataframe to an array.

```python
arr = long_data[["Open","Close"]].astype(float).to_numpy()

input_arr = np.zeros((arr.shape[0],arr.shape[1]+2), dtype=float)
input_arr[:,:-2] = arr
```

I then ran a `timeit` on each of the functions. Note that I only ran 100 loops for the non-numba functions because 1000 loops just took far too long...


```python
iter_time = %timeit -o -n 100 via_iter(long_data)
arr_time = %timeit -o -n 100 arr_no_numba(input_arr)
numba_time = %timeit -o -n 1000 via_numba(input_arr)
```

    39.6 ms ± 937 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
    11 ms ± 213 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
    The slowest run took 4.44 times longer than the fastest. This could mean that an intermediate result is being cached.
    112 µs ± 83.7 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)

The results were pretty clear given that the units were completely different for the three tests. Here are some graphs to show the raw speeds (a very poor graph, I know)

```python
import plotly.express as px

px.bar(y=[iter_time.average*1000, arr_time.average*1000, numba_time.average*1000], x = ["Itertuples", "Numpy Array", "Numba"],
labels={"x":"Iteration Method", "y":" Time (in milliseconds)"})
```

![rawtime](https://user-images.githubusercontent.com/68678549/139377047-1fa55f88-7c1d-40f7-b8b4-ba875613472d.png)

And a much clearer graph showing the speed gain from using Numba


```python
px.bar(y=[iter_time.average/numba_time.average, arr_time.average/numba_time.average], x = ["Numba > Itertuples", "Numba > Numpy Array"],
labels={"x":"Iteration Method", "y":" Speed Gain"})
```

![spedgain](https://user-images.githubusercontent.com/68678549/139377045-ecf1b5aa-a69a-47b1-8046-02e91d34d4db.png)

## Conclusion

Numba appears to blow the competition out of the water - hence when needing to iterate through a dataframe in a backtesting context where you can't vectorize, use numpy arrays and then throw in that `@numba.jit` decorator and you'll be blasting through those iterations.  
