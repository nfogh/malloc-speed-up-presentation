---
title: Use malloc to speed up your code... Wait... What?!
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
overviewSnapshots: true
---

# Use malloc to speed up your code... Wait... What?!

---
---

# A little background

<v-clicks>

 - Did internal C++ training (beginner/intermediate)
 - Wanted to show difference between heap and stack allocations

</v-clicks>

<img v-click src="/quick-bench.png" width=80%>

<!--
Difference between free store and automatic storage duration
-->

---

# Code to benchmark

```cpp
struct MyClass {
  float f;
  int i;
  int j;
};

struct AllocatingStruct {
  AllocatingStruct() : i(std::unique_ptr<MyClass>(new MyClass)) {}

  std::unique_ptr<MyClass> i;
};

struct NonAllocatingStruct {
  NonAllocatingStruct() {}
  MyClass i;
};
```
<v-clicks>

 - I don't use make_unique, as it zeroes the allocated memory.
 - That's why we have make_unique_for_overwrite.

</v-clicks>

---
---

# Benchmarking code

```cpp
static void AllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    AllocatingStruct myStruct;
    benchmark::DoNotOptimize(myStruct);
  }
}

static void NonAllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    NonAllocatingStruct myStruct;
    benchmark::DoNotOptimize(myStruct);
  }
}
```

---

# Benchmark result

<img src="/benchmark1.png" width=70%>

<v-clicks>

 - Great, just as we expected.
 - What about if we increase the size of the struct?
 - Will the allocating case get even worse?

</v-clicks>

---
---

# New class definition

```cpp
struct MyClass
{
  float f;
  int i[1024];
  int j;
};
```

<v-clicks>

 - What do you think?
 - I expected no real change
 - Maybe slightly slower malloc

</v-clicks>

---
---

# Benchmark results (larger struct)

<img src="/benchmark2.png" width=70%>

<h2 v-click>Wait… What!?</h2>

<v-clicks>

 - Allocating is actually _faster_ than the non-allocating case?

</v-clicks>

---

Non-allocating             |  Allocating
:-------------------------:|:-------------------------:
![Non-allocating assembly](/NonAllocatingStructBM.png) | ![Non-allocating assembly](/AllocatingStructBM.png)

---

# Let's fix up the benchmark

````md magic-move
```cpp
static void AllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    AllocatingStruct myStruct;
    benchmark::DoNotOptimize(myStruct);
  }
}
```
```cpp
static void AllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    AllocatingStruct myStruct;
    benchmark::DoNotOptimize(*myStruct.i);
  }
}
```
````

---

# Sane results again

<img src="/saneresults.png" width=75%>

---
layout: two-cols-header
---

# But wait, there is more

::left::

<v-clicks>

 - Let's try original code on compiler-explorer
 - Same code
 - Same compiler version (gcc 9.4)
 - Same compilation flags (-O2)

</v-clicks>

::right::

<v-clicks>

```text
----------------------  ----------  ----------  -----------
Benchmark                   Time         CPU     Iterations
----------------------  ----------  ----------  -----------
NonAllocatingStructBM   0.297 ns    0.178 ns     3932310046
AllocatingStructBM        169 ns     67.3 ns       10048315
Noop                    0.426 ns    0.174 ns     3884135829
```

<img src="/benchmark2.png">

</v-clicks>

---
---
# What can we learn?
<br>
<br>
<br>
<br>
<br>
<h1 v-click style="text-align:center;">"Benchmarking is hard..."</h1>
<br>
<br>

<v-clicks>

 - Don't trust other results
 - Use your own compiler to benchmark

</v-clicks>

---
layout: end
---

Thank you