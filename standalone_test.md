# Standalone Test Summary

## Experiment Overview
In the standalone test, we try to compare the performance of ```pg_qsort``` with other sorting routines in an independent environment. The sorting routines that are tested in this experiment are:

- ```pg_qsort``` The original sorting algorithms in PostgreSQL, which is an optimized quicksort using median of 3 (or 9 if the array size is large) technique to select pivots.
- ```heap_sort``` A heap sort routine based on the qseudo code [here](https://en.wikipedia.org/wiki/Heapsort)
- ```tim_sort``` A hybrid sorting algorithm derived from merge sort and insertion sort, claimed to be very fast on nearly sorted data
- ```dual_pivot_quicksort``` A quicksort routine that selects two pivots in each recursion, which is implemented based on the JAVA code in JDK8 (```java.util.DualPivotQuicksort```)
- ```radix_sort``` A radix sort routine based on this [article](https://www.geeksforgeeks.org/radix-sort/), expected to sort **integer** arrays in linear time.
- ```intro_sort``` Intro sort is basically ```pg_qsort``` with depth limit. This hybrid algorithm switches to heap sort when the recursions exceed depth limit (depth limit = 2logn in this implementation).
- ```intro_sort_check_once``` This is a variation of intro sort which performs the presorted check only once.
- ```rand_pg_qsort``` This is a variation of ```pg_qsort``` which uses random pivot selection to avoid extremely bad performance on killer input sequence.

Here are specifications about the input data:

Data type: 
- 32-bit integers
- 64-bit double floatings
- strings

Data patterns:
- Sorted: already sorted array 
- Random: random array 
- Reversed: reversely sorted array 
- Mostly sorted: divide a sorted array into bins of size 10, then randomly shuffle these bins 
- Mostly reversed: divide a reversed array into bins of size 10, then randomly shuffle these bins 
- Killer sequence: a specially generated sequence that makes qsort reach 𝑛^2 time complexity 

Data size:

- 100000
- 1000000
- 10000000 (not available for killer sequence)

Cardinality:
- low
- high

## Experiment Steps
1. Clone the test code [repo](https://github.com/Strider-Alex/PostgreSQLSorting)  
2. Compile the source code and run the executable. 
3. Modify macro ```TYPE_CODE``` in ```benchmark.h``` to test different data types. 

## Experiment Environment
- **OS:** Windows 10

- **CPU:** 4 * Intel® Core™ i5-7300U CPU @ 2.60GHZ

- **Compiler:** Visual C++ with all optimization on

## Experiment Result
**The detailed result can be found under ```analyzed_data``` folder.**

#### ```heap_sort```
- When sorting small high cardinality random data with cheap comparison function (n = 100000, int/double, high cardinality, random), heap sort has the same performance (100%, 98.06%) as ```pg_qsort```
- Outperfored greatly by ```pg_qsort``` on large partially-sorted low-cardinality data with an expensive comparison function (heap sort can be more than 10 times slower in some extreme cases, which are low cardinality partially-sorted strings)
- Most significant factors are cardinality and data pattern.
- O(nlogn) worst case time complexity, only takes less than 1% time of ```pg_qsort``` to sort the killer sequence of size 1000000

#### ```tim_sort```
- In comparison of ```pg_qsort```, ```tim_sort``` Takes about 150% time to sort disctinct random strings, 150%-200% to sort integers and doubles. For low cardinilty random data, this can increase to about 400%
- For dinstinct partially-sorted data, ```tim_sort only``` takes 20%-90% time of ```pg_qsort```
- For low cardinality data, this advantage no longer exists
- Generally performs better on strings than other data types
- O(nlogn) worst case time complexity

#### ```dual_pivot_quicksort``` 

- ```dual_pivot_quicksort``` takes about 150% time to sort high cardinality random data and 200% to sort partially-sorted data
- For low cardinality integers/doubles, this can increase to around 400%
- Generally performs better on strings than other data types
- O(n^2) worst case time complexity

#### ```radix_sort```
- Takes 70%-80% time to sort 10000000 random integers
- Radix sort can not take advantage of data pattern, so it needs 150-250% time of ```pg_qsort``` to sort partially-sorted data
- Works better on low cardinality integers

#### ```intro_sort```
- For high cardinality data, ```intro_sort``` has nearly the same performance as ```pg_qsort``` (about 98%-105%)
- For low cardinality data, ```intro_sort``` can be relatively slower (about 98%-110%)
- No significant difference between string and integer
- Only need 85%-95% time of ```pg_qsort``` to sort double floatings. This is a little bit strange and may be platform related
- O(nlogn) worst case time complexity, takes less than 1% time of ```pg_qsort``` to sort the killer sequence of size 1000000

#### ```intro_sort_check_once```
- This variation of intro sort takes 90%-100% time to sort high cardinality random data.
- For partially-sorted integers and high cardinality partially-sorted strings, it takes 100%-110% time. For low cardinality partially-sorted strings, the performance is nearly the same.
- Other properties are very similar with the ```intro_sort```

#### ```rand_pg_qsort```
 - About 100%-110% time to sort random data
 - 110%-140% time of ```pg_qsort``` to sort paritialy-sorted data
 - Impossible to generate killer sequence for this algorithm, so we can consider the time complexity to be O(nlogn)

## Conclusion

- Generally speaking, it's very difficult to greatly improve the average performance of the sorting routine in ```postgresql```. Sorting random integers using radix sort might be an option because of the 20%-30% gain illustrated in this test. 
- ```pg_qsort``` may not be the best for random input, but it works very well for paritically sorted data.
- If we want O(nlogn) worst case time complexity, intro sort is better than random pivot selection.
- In this experiment, the performance of ```intro_sort``` and ```pg_qsort``` is very similar.
- ```intro_sort_check_once``` is better at sorting random data, but slower on partially sorted data.
- In this experiment, a killer sequence of 1000000 elements can make ```pg_qsort``` more than 100 times slower than other sorting algorithms.