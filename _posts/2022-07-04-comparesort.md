---
title: "其他比较排序"
excerpt: ""
date: 2022-07-01
last_modified_at: 2022-07-01
categories:
  - 算法
tags:
  - 排序
---

## 冒泡排序

冒泡排序可能是我们接触的第一个排序算法，排序的过程就是每一次循环都从头开始比较相邻的两个元素，把大的元素一直往后推，在最多经历`size - 1`次循环后就排好序了，在循环了 n 次后可以排除后 n 个元素，如果某次循环没有任何的元素交换，那说明已经排好序了，可以直接跳出外层循环。

时间复杂度：_O(n2)_。

空间复杂度： _O(1)_。

稳定性：稳定。

[可视化排序过程](https://www.cs.usfca.edu/~galles/visualization/ComparisonSort.html)。

```javascript
function bubbleSort(arr) {
  let end = arr.length - 1;
  let swap;
  do {
    swap = false;
    for (let i = 0; i < end; i++) {
      if (arr[i] > arr[i + 1]) {
        [arr[i], arr[i + 1]] = [arr[i + 1], arr[i]];
        swap = true;
      }
    }
    end--;
  } while (swap);
}
```

## 插入排序

插入排序的过程可以想象为从桌子上的扑克牌堆依次拿起扑克，大的就向后移动，然后把拿起的扑克插在对应的位置上。从第二个元素开始，因为第一个元素已经排好序了。

时间复杂度：_O(n2)_。

空间复杂度： _O(1)_。

稳定性：稳定。

[可视化排序过程](https://www.cs.usfca.edu/~galles/visualization/ComparisonSort.html)。

```javascript
function insertionSort(arr) {
  for (let i = 1; i < arr.length; i++) {
    const pending = arr[i];
    let j = i - 1;
    while (j >= 0 && arr[j] > pending) {
      arr[j + 1] = arr[j];
      j--;
    }
    arr[j + 1] = pending;
  }
}
```

## 归并排序（递归版）

归并排序使用分治的思想

1. 把数组分成两半；
2. 对两个已排好序的数组采用归并操作，长度为 1 的数组就相当于已排好序。

```javascript
function mergeSort(arr) {
  if (arr.length === 1) {
    return arr;
  } else {
    const mid = arr.length >>> 1;
    const arr1 = arr.slice(0, mid);
    const arr2 = arr.slice(mid);
    return merge(mergeSort(arr1), mergeSort(arr2));
  }
}

function merge(arr1, arr2) {
  const result = [];
  let i = 0;
  let j = 0;
  while (i < arr1.length && j < arr2.length) {
    if (arr1[i] < arr2[j]) {
      result.push(arr1[i]);
      i++;
    } else {
      result.push(arr2[j]);
      j++;
    }
  }
  while (i < arr1.length) {
    result.push(arr1[i]);
    i++;
  }
  while (j < arr2.length) {
    result.push(arr2[j]);
    j++;
  }
  return result;
}
```

下面的实现不切分原始数组

```javascript
function mergeSort(arr, left, right) {
  if (left < right) {
    const mid = (left + right) >>> 1;
    mergeSort(arr, left, mid);
    mergeSort(arr, mid + 1, right);
    merge(arr, left, mid, right);
  }
}

function merge(arr, left, mid, right) {
  const temp = [];
  let i = left;
  let j = mid + 1;
  while (i <= mid && j <= right) {
    if (arr[i] < arr[j]) {
      temp.push(arr[i]);
      i++;
    } else {
      temp.push(arr[j]);
      j++;
    }
  }
  while (i <= mid) {
    temp.push(arr[i]);
    i++;
  }
  while (j <= right) {
    temp.push(arr[j]);
    j++;
  }

  for (let k = left; k <= right; k++) {
    arr[k] = temp.shift();
  }
}
```

时间复杂度：_O(nlogn)_。

空间复杂度： _O(n)_。

稳定性：稳定。
