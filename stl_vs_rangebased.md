# When performance matters...

... it is best to use STL algorithms. C++ is usually used in performance-critical environments. But for the sake of simplicity, many developers use range-based loops. 

```c++
for(const auto& e : {1, 2, 3, 4, 5})
{
   std::cout << e << std::endl;		
}
```
Which basically transforms into a ```for loop```. 
```c++
{
   std::initializer_list<int> && __range1 = std::initializer_list<int>{1, 2, 3, 4, 5};
   const int * __begin1 = __range1.begin();
   const int * __end1 = __range1.end();
   for(; __begin1 != __end1; ++__begin1) {
      const int & e = *__begin1;
      std::cout.operator<<(e).operator<<(std::endl);
   }
}
```
The code-transformation can be recreated very easily with [https://cppinsights.io](https://cppinsights.io). This simple for loop allows the compiler to make only a few optimizations.
Whereas with STL algorithms optimized code is called, for example special overloads for various [iterators](https://en.cppreference.com/w/cpp/iterator) is used. On top of that, the code gets even better with the optimizer. 

I would like to share my experiences with a simple example. The following code fills a map from the content of a vector. 
```c++
std::vector<int> input = get_data();
std::map<int, bool> output;

for (auto i : input)
{
   testmap.insert({ i, true });
}
```

## Benchmark

I used [QuickBench](https://www.quick-bench.com/) as a benchmark tool. The current test code can be found under the following link: https://quick-bench.com/q/ZmBU33u5AOqU4cGScAaD5efJbwo

The results speak for themselves. In the plot shown here, the results of different container sizes are plotted over time. The blue graph represents the implementation with the STL and the green the ranged-based-for-loop. As can be clearly seen, the performance gain increases with the container size. 

![elements_over_time](/images/elements_over_time.png)

As can be seen below, the performance gain increases with the number of elements in the container. With 8000 entries, the factor between the two implementations is around 5.6. But even with relatively small container sizes, the gain in performance is clear. Still with only 10 entries the factor is 1.6.

![elements_over_factor](/images/elements_factor.png)


## Readability

Readability is also significantly improved with the STL. The code underneath can be read as followed: "Count all values in a range that are odd". 

```c++
const auto is_odd = [](auto e) { return (e % 2 != 0); };
const auto n = std::count_if(begin(values), end(values), is_odd);
```
But how does the following code read? 
```c++
std::size_t n{0};
for (auto e : values) {
   if (e % 2 != 0) {
      ++n;
   }
}
```

By the way, the STL version is faster than the range-based-for-loop by a factor of 2.6 on average. The idependence from the container size is also shown, in the second plot. 

![count_if](/images/count_if.png)
![count_if_factor](/images/count_if_factor.png)

[Benchmark: 'count'](https://quick-bench.com/q/uA5_fuPslRkY77ArtGKmfphSSbs)


## Conclusion

If performance is a development requirement, make sure to take a look at the [algorithm](https://en.cppreference.com/w/cpp/algorithm) header. There are many specialized algorithms that not only increase the performance but also the readability of the code. 

#
Copyright &copy; 2022 by Michael Miller 
