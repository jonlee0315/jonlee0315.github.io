---
layout:     post
title:      "排序算法总结"
subtitle:   "Sort Algorithms"
date:       2018-08-08
author:     "Jon Lee"
header-img: "img/in-post/2018-08-08-sort-algorithm/bg.jpg"
catalog: true
categories : 算法与数据结构
tags:
    - 算法
    - 排序
    - Java
---

### 概述

排序算法的目的是将数组中的元素按照某种方式排列（通常是按照大小或者是字母顺序）。  
经常会用到的两个操作就是 **比较** 和 **交换**，为了方便，先定义一下这两个函数`less`和`swap`，排序算法中就可以直接调用了。

    private static boolean less(Comparable v, Comparable w) {
            return v.compareTo(w) < 0;
    }

    private static void swap(Comparable[] a, int i, int j) {
            Comparable t = a[i];
            a[i] = a[j];
            a[j] = t;
	}

然后是检验是否排序成功的`isSorted`。

    public static boolean isSorted(Comparable[] a) {
            for (int i = 1; i < a.length; i++) {
                    if (less(a[i], a[i - 1])) {
                            return false;
                    }
            }
            return true;
    }

下面排序算法默认都是按升序排序。

### 冒泡排序

冒泡排序（Bubble Sort）,每次比较相邻两个元素，每一趟操作都将最大元素移动到最后，不断缩小排序范围。  
增加`changed`标志位，如果当前这一趟操作没有发生交换操作，说明已经有序，将最好情况下的时间复杂度$O(n^2)$ 减小为 $O(n)$。

![](/img/in-post/2018-08-08-sort-algorithm/bubble.jpg)

	public static void bubbleSort(Comparable[] a) {
		boolean changed = true;
		int N = a.length;
		for (int i = 0; i < N - 1 && changed; i++) {
			changed = false;
			for (int j = 0; j < N - i - 1; j++) {
				if (less(a[j + 1], a[j])) {
					swap(a, j, j + 1);
					changed = true;
				}
			}
		}
	}

### 选择排序

选择排序（Selection Sort），最简单直观的一种排序，每次选择最小的元素放到数组前面。  
选择排序的时间复杂度和输入无关，每次比较并没有可用的信息被记录，选择排序的优点是数据移动的次数少，为N次。

![](/img/in-post/2018-08-08-sort-algorithm/selection.gif)

	public static void selectionSort(Comparable[] a) {
		int N = a.length;
		for (int i = 0; i < N; i++) {
			int min = i;
			for (int j = i + 1; j < N; j++) {
				if (less(a[j], a[min])) {
					min = j;
				}
			}
			swap(a, i, min);
		}
	}

### 插入排序

插入排序（Insertion Sort），和玩扑克牌时码牌类似，将目标元素插入到合适位置。  
插入排序是希尔排序的基础。

![](/img/in-post/2018-08-08-sort-algorithm/insertion.gif)

	public static void insertionSort(Comparable[] a) {
		int N = a.length;
		for (int i = 1; i < N; i++) {
			for (int j = i; j > 0 && less(a[j], a[j - 1]); j--) {
				swap(a, j, j - 1);
			}
		}
	}

### 希尔排序

希尔排序（Shell Sort），一种对插入排序的改进算法。希尔排序通过将比较的全部元素分为几个区域来提升插入排序的性能。这样可以让一个元素可以一次性地朝最终位置前进一大步。然后算法再取越来越小的步长进行排序，算法的最后一步就是普通的插入排序，但是到了这步，需排序的数据几乎是已排好的了。  
希尔排序的效率受步长选择的影响。

	public static void shellSort(Comparable[] a) {
		int N = a.length;
		int h = 1;
		while (h < N / 3) {
			h = 3 * h + 1;
		}
		while (h >= 1) {
			for (int i = 1; i < N; i++) {
				for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
					swap(a, j, j - h);
				}
			}
			h /= 3;
		}
	}

### 归并排序

归并排序（Merge Sort），创建在归并操作上的一种有效的排序算法。1945年由约翰·冯·诺伊曼首次提出。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用，且各层分治递归可以同时进行。  
归并排序需要 $O(n)$ 大小的辅助空间，时间复杂度为 $O(nlogn)$ ，比上面的几种算法都要小。

![](/img/in-post/2018-08-08-sort-algorithm/merge.gif)

	private static Comparable[] aux;

	public static void mergeSort(Comparable[] a) {
		int N = a.length;
		aux = new Comparable[N];
		sort(a, 0, N - 1);
	}

	private static void sort(Comparable[] a, int lo, int hi) {
		if (lo >= hi) {
			return;
		}
		int mid = lo + (hi - lo) / 2;
		sort(a, lo, mid);
		sort(a, mid + 1, hi);
		merge(a, lo, mid, hi);
	}

	private static void merge(Comparable[] a, int lo, int mid, int hi) {
		for (int i = lo; i <= hi; i++) {
			aux[i] = a[i];
		}
		int i = lo, j = mid + 1;
		for (int k = lo; k <= hi; k++) {
			if (i > mid) {
				a[k] = aux[j++];
			} else if (j > hi) {
				a[k] = aux[i++];
			} else if (less(aux[j], aux[i])) {
				a[k] = aux[j++];
			} else {
				a[k] = aux[i++];
			}
		}
	}

### 快速排序

快速排序（Quick Sort），每次选出一个元素，将它放到一个位置，使它左边的元素都小于等于它，右边的元素大于等于它，然后递归左右两个子数组。  
快排有很多种，都对原来的算法进行了改进，下面列出最基本快排的代码。

	public static void quickSort(Comparable[] a) {
		qsort(a, 0, a.length - 1);
	}

	private static void qsort(Comparable[] a, int lo, int hi) {
		if (lo >= hi) {
			return;
		}
		int j = partition(a, lo, hi);
		qsort(a, lo, j - 1);
		qsort(a, j + 1, hi);
	}

	private static int partition(Comparable[] a, int lo, int hi) {
		int i = lo, j = hi + 1;
		Comparable v = a[lo];
		while (true) {
			while (less(a[++i], v)) {
				if (i == hi) {
					break;
				}
			}
			while (less(v, a[--j])) {
				// a[lo] 不可能比自己小，所以省略判断
			}
			if (i >= j) {
				break;
			}
			swap(a, i, j);
		}
		swap(a, lo, j);
		return j;
	}

### 堆排序

堆排序（Heap Sort）相对复杂一点，具体可以参考这篇文章[Heapsort堆排序算法详解](https://www.cnblogs.com/jetpie/p/3971382.html)。

	public static void heapSort(Comparable[] a) {
		int heapSize = a.length;
		for (int i = heapSize / 2 - 1; i >= 0; i--) {
			heapify(a, heapSize, i);
		}
		for (int i = a.length - 1; i > 0; i--) {
			Comparable v = a[0];
			a[0] = a[i];
			a[i] = v;
			heapSize--;
			heapify(a, heapSize, 0);
		}
	}

	private static void heapify(Comparable[] a, int heapSize, int i) {
		int left = 2 * i + 1;
		int right = 2 * i + 2;
		int max = i;
		if (left < heapSize && less(a[max], a[left])) {
			max = left;
		}
		if (right < heapSize && less(a[max], a[right])) {
			max = right;
		}
		if (max != i) {
			Comparable v = a[i];
			a[i] = a[max];
			a[max] = v;
			heapify(a, heapSize, max);
		}
	}

### 总结比较

各种排序算法的比较。

名称|平均时间复杂度|最坏时间复杂度|最好时间复杂度|额外空间复杂度|稳定性
:--:|:----------:|:-----------:|:-----------:|:-----------:|:----:
冒泡排序|$O(n^2)$|$O(n^2)$|$O(n)$|$O(1)$|稳定
选择排序|$O(n^2)$|$O(n^2)$|$O(n^2)$|$O(1)$|不稳定
插入排序|$O(n^2)$|$O(n^2)$|$O(n)$|$O(1)$|稳定
希尔排序|与步长有关|与步长有关|$O(n)$|$O(1)$|不稳定
归并排序|$O(nlogn)$|$O(nlogn)$|$O(nlogn)$|$O(n)$|稳定
快速排序|$O(nlogn)$|$O(n^2)$|$O(nlogn)$|与具体实现有关|不稳定
堆排序|$O(nlogn)$|$O(nlogn)$|$O(nlogn)$|$O(1)$|不稳定

### 实验

不同大小的测试用例下测试各种排序算法的性能，时间单位ms。

名称|10,000|100,000|1,000,000|10,000,000|$10^6$（有序）|$10^6$（逆序）
:--:|:----------:|:-----------:|:-----------:|:-----------:|:----:|:-------:
冒泡排序|439|53790|...|...|<10|...
选择排序|157|24423|...|...|...|...
插入排序|205|30843|...|...|<10|...
希尔排序|<10|70|1158|33736|38|116
归并排序|<10|78|392|5307|157|146
快速排序|<10|45|332|3962|栈溢出|栈溢出
堆排序  |<10|55|807|14260|291|244

可以看到不同算法的运行时间基本与时间复杂度一致。  
但是快排在有序情况下会出现栈溢出的情况，快排的最好情况就是每次都能将数组对半分，而有序情况下并不能满足这点，被排序元素不是出现在最左端就是最右端，所以在快排前最好先将待排序数组打乱。

### 参考资料

>《Algorithms 4th Edition》   
[https://zh.wikipedia.org/wiki/排序算法](https://zh.wikipedia.org/wiki/排序算法)
