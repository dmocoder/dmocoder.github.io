---
layout: post
title:  "Quick Sort #1"
date:   2020-07-07 21:00:00 +0000
categories: csharp front-end
---
To get to grips with Rust I decided to implement some basic sorting algorithms. First up is Quick sort.

Quick Sort is an in-place sorting algorithm with average time complexity `O(nlog(n))`. Its a fairly well known algorithm taught to CS Undergrads and there are much better explanations for how it works than this blog but in a nutshell:
1 - Select a Pivot value from the array. This varies because the choice of pivot can impact the sort time significantly but typically the value of the middle index of the array is used
2 - Sort the array so that values on the left are less than the pivot value and values on the right are greater (or equal)
3 - Partition the array in two using these sorted halves and repeat the above process again, choosing a new pivot within each partition
4 - Repeat recursively until the partition is just a single cell

### Sorting an Int32 Array

The easiest implementation of Quick Sort is sorting an integer array. In C# or any C-style language this is pretty straightforward and that's generally the same for Rust.

```
pub fn sort_int_array(arr: &mut [i32]) {
    quick_sort_range(arr, 0, arr.len() - 1);
}
```
Here's the public method we can pass a reference to an int32 array to be sorted. Nothing too crazy here but note the parameter definition; we need to pass in a mutable reference to the int32 array. We expect the quick sort method to sort the array and to allow this we need to declare the parameter as mutable. 
If we didn't need to mutate `arr` we could just pass in `& [i32]`. 

The ampersand `&` indicates that a reference is passed to the function, not the value. In C# Arrays are usually passed to a method by reference so this isn't unusual but its important to note that in Rust if we didn't pass it as a reference this function would then have ownership over the array. This would mean that the caller could no longer access the array they requested to be sorted. There wouldn't be much point to this function if that was the case.

```
fn quick_sort_range(arr: &mut[i32], low: usize, high: usize) {
    let index = partition(arr, low, high);

    if low < index - 1{
        quick_sort_range(arr, low, index - 1);
    }
    if index < high {
        quick_sort_range(arr, index, high);
    }
}
```
This is the recursive function. It calls the Partition formula which does the bulk of the work and then calls it again for each side of the partition if its greater than a single element. 
The function accepts the range of the array its sorting so it can recursively sort each half of the partition. 
I have assigned the range to a `usize` variable instead of `i32`. Depending on the architecture of the system executing the code usize may be a 32 bit integer, but to index an array you need to use a `usize` variable, not `i32`. `usize` is also the type returned from the `len()` function called above.

```
fn partition(arr: &mut [i32], low: usize, high: usize) -> usize {
    let pivot = low + (high - low)/2;
    let pivot_value = arr[pivot];

    let mut left: usize = low;
    let mut right: usize = high;

    while left < right {
        while arr[left] < pivot_value {
            left += 1;
        }
        while arr[right] > pivot_value {
            right -= 1;
        }

        if right < left {
            return left
        }
        else {
            swap_index(arr, left, right);
            left += 1;
            right -= 1;
        }
    }

    left
}
```
The partition function selects the pivot value and sorts the range of the array, moving values lower than the pivot to the left and higher than the pivot to the right. The index returned by the function returns the position of this partition. Nothing here is too gnarly although its worth noting that in C# we would increment `left` and `right` using the post-increment operator e.g. `left++;`. We can't do that in Rust because Rust doesn't have any post-increment operators. I'm not exactly sure why, outside of the understanding that it is apparently very difficult to implement a post-increment operator, but the workaround is to just use `left += 1;`.

```
fn swap_index(arr: &mut [i32], first: usize, second: usize) {
    let swap = arr[first];
    arr[first] = arr[second];
    arr[second] = swap;
}
```
Lastly this function is called by the Partition function to swap elements of the array. With int32 arrays this is simple but for a collection that can't be copied easily we'll see that its not so straightforward. Also this function is overkill as Rust provides a single `swap()` method on collections that would achieve the above in a single line. I'll add it in a later post.

Anyway that's it. This can be called by instantiating an array and passing it in, like so:

```
insert code here
```
