---
layout: post
title: Why Rust Ownership?
subtitle: why do we really need ownership in rust?
tags: [rust]
---

**NOTE: This is an old post that I've transferred here, the original post is [here in medium](https://medium.com/@hariom.2711/rust-ownership-from-scratch-7cbb1369aebe). The blog post date is the date when I originally posted it, although I moved it here in Oct. 2024**  

Memory ownership in Rust sounds like a very alien concept to someone like me who has been coding in garbage collected languages for a long time.  
The first thing that always comes to mind is “why even do this?”. I’ll try to provide the motivation behind it in this blog post.  
Before that, it would be good if you are at least familiar with what the concept is. This [chapter](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html) does a good job teaching that.  
We’ll write a simple merge sort program in C. I will iteratively try to reduce memory leaks in the program.  
By taking a hard look at our memory usage, we’ll end up with a basic ownership+borrow memory management strategy.  
Take a look at the theory behind merge sort [here](https://www.geeksforgeeks.org/merge-sort/) for a refresher.  

We’ll write a simple merge sort program in C. I will iteratively try to reduce memory leaks in the program.  
By taking a hard look at our memory usage, we’ll end up with a basic ownership+borrow memory management strategy.  

Take a look at the theory behind merge sort [here](https://www.geeksforgeeks.org/merge-sort/) for a refresher.

---

This is the first draft of the code. I have commented the code and hope that it is enough for an explanation

```c
/**
 * Merge sort
 *    Take an array, split it in half
 *    recursively sort both halves
 *    go through the sorted halves, pick the first element of each and push the smaller element in result
 *    loop through both halves until all their elements are pushed to result
 */
#include <stdio.h>
#include <stdlib.h>

/**
 * convenience data type.
 * Simply store both halves and their lengths for a given array
 */
typedef struct
{
    float *left;
    size_t len_left;
    float *right;
    size_t len_right;
} halves_t;

/**
 * method to get halves from a given array and its length
 */
halves_t get_halves(float *arr, size_t len) {
    size_t right_offset = len / 2;
    size_t len_right = len - right_offset;
    size_t left_offset = 0;
    size_t len_left = right_offset - left_offset;
    halves_t half = {
        .len_right = len_right,
        .len_left = len_left,
        .left = arr + left_offset,
        .right = arr + right_offset,
    };
    return half;
}

/**
 * merge two sorted halves into result
 * this method allocates memory for holding result in the heap and returns it after merging the halves
 */
float *_merge(float *left, size_t len_left, float *right, size_t len_right) {
    float *result = (float *)malloc((len_left + len_right) * sizeof(float));
    size_t i = 0;
    size_t j = 0;
    size_t result_i = 0;
    while (i < len_left && j < len_right) {
        if (left[i] < right[j]) {
            result[result_i] = left[i];
            i++;
        } else {
            result[result_i] = right[j];
            j++;
        }
        result_i++;
    }
    while (i < len_left) {
        result[result_i] = left[i];
        result_i++;
        i++;
    }
    while (j < len_right) {
        result[result_i] = right[j];
        result_i++;
        j++;
    }
    return result;
}

/**
 * main recursive function
 */
float *_merge_sort(halves_t halves) {
    float *left;
    if (halves.len_left > 0) {
        // recursively sort if the array has elements
        left = _merge_sort(get_halves(halves.left, halves.len_left));
    } else {
        // left is technically empty here, its elements are never pushed in result
        left = halves.left;
    }
    float *right;
    if (halves.len_right > 1) {
        // recursively sort if the array has elements
        right = _merge_sort(get_halves(halves.right, halves.len_right));
    } else {
        // right has exactly `1` element here, instead of trying to sort it again (its gonna be infinite recursion if we do)
        // simply assign right to the initial array
        right = halves.right;
    }
    float *result = _merge(left, halves.len_left, right, halves.len_right);
    return result;
}

float *merge_sort(float arr[static 1], size_t len) {
    halves_t half = get_halves(arr, len);
    float *sorted = _merge_sort(half);
    return sorted;
}

int main() {
    float A[4] = {
        0,
        -1,
        0.4,
        1.5,
    };
    float *result = merge_sort(A, 4);
    for (size_t i = 0; i < 4; i++) {
        printf("%.1f\n", result[i]);
    }
    free(result);
    return EXIT_SUCCESS;
}
```


<img class="floating-right-picture" src="/assets/crying-full-duck.png">

And yes, this function leaks memory ;_;

I really hope managing memory becomes second nature. Otherwise I can only assume ridiculously geniuses touching C and laughing at us :(
Let’s count the memory allocations of this code for the sample array `[0, -1, 0.4, 1.5]`

<div class="image-with-caption">
  <img src="/assets/rust-why-ownership/tree.webp" alt="Memory allocations in circles, array for input to _merge_sort in squares">
  <p>Memory allocations in circles, array for input to _merge_sort in squares</p>
</div>

Now each recursive call ends up allocating memory and returning it. Since we are not cleaning up anything, this function allocates a total of 10 floats.  
In reality it should only end up allocating 4 floats (it returns a newly generated float array pointer, the `main` function expects it to return that much). We have generated 6 bytes in excess.  
Now, if you look carefully, we are allocating new bytes as a result of calling `_merge_sort`  
These are then assigned to `left` and `right`, maybe calling `free` on both of these after we merge might solve our issue.  

```c
/**
 * main recursive function
 */
float *_merge_sort(halves_t halves) {
    float *left;
    if (halves.len_left > 0) {
        // recursively sort if the array has elements
        left = _merge_sort(get_halves(halves.left, halves.len_left));
    } else {
        // left is technically empty here, its elements are never pushed in result
        left = halves.left;
    }
    float *right;
    if (halves.len_right > 1) {
        // recursively sort if the array has elements
        right = _merge_sort(get_halves(halves.right, halves.len_right));
    } else {
        // right has exactly `1` element here, instead of trying to sort it again (its gonna be infinite recursion if we do)
        // simply assign right to the initial array
        right = halves.right;
    }
    float *result = _merge(left, halves.len_left, right, halves.len_right);
    // free left and right
    free(left);
    free(right);
    return result;
}
```

Focus on Lines 23–24. If you try running the code with these, it would end up with a core dump. `clang-tidy` would also warn you saying you are trying to de-allocate a static array. So what went wrong?

```bash
free(): invalid pointer
[1]    26103 IOT instruction (core dumped)  ./a.out
```

The issue is at **lines 11 & 20**. When we don’t have any more sorting to do recursively, we are assigning `left` and `right` to the original references passed in the function. These references are actually pointers to the memory occupied by `A` (our main input in main function).
We can say we tried to free something we **“borrowed”**.
Fixing it is actually very easy, simply allocate new memory and copy the content of the original references. Create a new **“owned”** pointer.


```c
/**
 * main recursive function
 */
float *_merge_sort(halves_t halves) {
    float *left;
    if (halves.len_left > 0) {
        // recursively sort if the array has elements
        left = _merge_sort(get_halves(halves.left, halves.len_left));
    } else {
        // left is technically empty here, its elements are never pushed in result
        left = (float *) malloc(sizeof(float));
        *left = 0.0;   // left is always empty here, we assign a sentinel with 1 size and add 0.0
    }
    float *right;
    if (halves.len_right > 1) {
        // recursively sort if the array has elements
        right = _merge_sort(get_halves(halves.right, halves.len_right));
    } else {
        // right has exactly `1` element here, instead of trying to sort it again (its gonna be infinite recursion if we do)
        // simply assign right to the initial array
      
        // copy contents of halves.right into right.
        right = (float *) malloc(sizeof(float) * halves.len_right);
        for (int i = 0; i < halves.len_right; i++) {
            right[i] = halves.right[i];
        }
    }
    float *result = _merge(left, halves.len_left, right, halves.len_right);
    free(left);
    free(right);
    return result;
}
```

Lines 11–12 and 23–25 are interesting here. The code would pass and should no longer have any memory leaks
What I essentially did was just create a copy that I owned and could free anywhere inside this function

---

As I was writing this code and was struggling with managing my memory, I ended up using the strategy Rust uses.

> Always only free the memory assigned by you (the function), in essence owned by you

The code now follows the following assumptions:  
1. _merge_sort “borrows” its input arguments halves.left and halves.right
2. It only every reads those arguments, does not play with their memory
3. _merge function creates a new pointer with the sorted result and returns it, in essence transferring the ownership to the calling function
4. left and right created inside _merge_sort are always owned by _merge_sort (either with transfer from another _merge_sort down the stack, or by copying contents of the borrowed inputs)
5. This makes it safe for _merge_sort to free memory for left and right
6. Note that _merge_sort is not allowed to free the memory of result because if returns it (transfers the ownership to the caller later on)
7. If you own something, make sure you clean it up if you do not return it to the caller.

> Rust puts all of these checks at compile time, it also creates a consistent framework for you to manage memory. It also automatically cleans up stuff you owned (or created) but did not return, essentially reducing boilerplate.

These are still really small toy examples, I hope to work on something substantial in future in C and understand the reasoning behind Rust's new programming constructs  
