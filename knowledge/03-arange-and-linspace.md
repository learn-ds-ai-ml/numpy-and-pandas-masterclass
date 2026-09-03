# `arange()` and `linspace`

NumPy has its own equivalent to the built-in Python range function, and it actually provides two primary options depending on your needs.

## 1. `np.arange()`

This is the closest equivalent to Python's `range` and returns evenly spaced values within a given interval.

- **Syntax:** `np.arange(start, stop, step)`
- **Key Advantage:** Unlike Python's `range`, it supports floating-point (decimal) step sizes.
- **Behavior:** It includes the `start` value but excludes the `stop` value (half-open interval).

```python
import numpy as np

# Integer step size
arr1 = np.arange(1, 5)
# Output: array([1, 2, 3, 4])

# Floating-point step size
arr2 = np.arange(0, 1, 0.2)
# Output: array([0. , 0.2, 0.4, 0.6, 0.8])
```

---

### why the name `arange`

The name is a direct blend of the words **Array** and **Range**.

- **The Logic:** Python already has a built-in `range()` function that outputs a range object. NumPy's version performs a very similar task but outputs a **NumPy array**.
- **The Blueprint:** `Array` + `Range` = `arange`.

---

## 2. `np.linspace()`

If you are working with decimals, `np.arange` can sometimes be unpredictable due to floating-point precision limits. `np.linspace` is the safer, preferred alternative for floating-point intervals.

- **Syntax:** `np.linspace(start, stop, num)`
- **Key Advantage:** Instead of specifying the step size, you specify the exact **number** of samples you want.
- **Behavior:** It includes both the `start` and `stop` values by default (closed interval).

```python
# Get exactly 5 evenly spaced numbers between 0 and 1
arr3 = np.linspace(0, 1, 5)
# Output: array([0.  , 0.25, 0.5 , 0.75, 1.  ])
```

---

### why the name `linspace`

The name stands for **Linearly** spaced **Space**.

- **The Logic:** In mathematics and data science, a "space" refers to a continuous area or interval of numbers (like the interval between 0 and 1). "Linear" means that the steps between each number are completely equal and constant.
- **The Blueprint:** `Linear` + `Space` = `linspace`.

---

## Summary

- Use `arange` when you want an array built by specifying the _size_ of **each step** (e.g., "give me numbers from 0 to 10, stepping by 2").
- Use `linspace` when you want to divide a **mathematical space** by specifying the **total number of points** (e.g., "give me exactly 5 points between 0 and 10").
