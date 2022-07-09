---
title: "快速排序"
excerpt: ""
date: 2022-07-07
last_modified_at: 2022-07-07
categories:
  - 算法
tags:
  - 排序
---

快速排序的思路：

1. 选择一个 pivot；
2. 将数组分成：小于 pivot | pivot | 大于 pivot
3. 递归排序小于 pivot 和大于 pivot 两部分

时间复杂度：_O(nlogn)_。

空间复杂度： _O(logn)_。

稳定性：不稳定。

快速排序有很多种实现方式，下面简单介绍几种。

```javascript
function quickSort(arr, low, high) {
  if (low < high) {
    let p = partion(arr, low, high);
    quickSort(arr, low, p - 1);
    quickSort(arr, p + 1, high);
  }
}

function partion(arr, low, high) {
  // 选择pivot
  const pivot = arr[low];
  // i 是第一个大于pivot的位置
  let i = low + 1;
  for (let j = low + 1; j <= high; j++) {
    // 如果比pivot小，则与第一个大于pivot的位置交换
    if (arr[j] < pivot) {
      if (i !== j) {
        [arr[i], arr[j]] = [arr[j], arr[i]];
      }
      i++;
    }
  }
  // 最后一个小于pivot的值与pivot交换
  [arr[low], arr[i - 1]] = [arr[i - 1], arr[low]];
  // 返回最新pivot的位置
  return i - 1;
}
```

上面的实现中，`i`指针指向的元素并不绝对大于 pivot，只是可能，如果选择的 pivot 就是最大值，那么不存在大于 pivot 的元素了，例如`10, 7, 8, 4, 3`。

[动画模拟](https://visualgo.net/en/sorting)

```javascript
function quickSort(arr, low, high) {
  if (low < high) {
    let p = partion(arr, low, high);
    quickSort(arr, low, p - 1);
    quickSort(arr, p + 1, high);
  }
}

function partion(arr, low, high) {
  const pivot = arr[low];

  // 使用两个指针，从两边向中间移动
  let i = low;
  let j = high + 1;

  while (true) {
    do {
      i++;
    } while (arr[i] < pivot);

    do {
      j--;
    } while (arr[j] > pivot);

    if (j <= i) {
      [arr[low], arr[j]] = [arr[j], arr[low]];
      return j;
    }

    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
}
```

[动画演示](https://www.cs.usfca.edu/~galles/visualization/ComparisonSort.html)

如果数组中存在很多重复元素，那么把数组分成三部分是更好的做法。

```javascript
function quickSort(arr, low, high) {
  if (low < high) {
    let [fe, le] = partion(arr, low, high);
    quickSort(arr, low, fe - 1);
    quickSort(arr, le + 1, high);
  }
}

function partion(arr, low, high) {
  const pivot = arr[low];
  // [low, i - 1]小于pivot
  let i = low;
  // [i, j - 1]等于pivot
  let j = low;
  // [j, k]未排序
  // [k + 1, high]大于pivot
  let k = high;

  while (j <= k) {
    if (arr[j] < pivot) {
      [arr[i], arr[j]] = [arr[j], arr[i]];
      i++;
      j++;
    } else if (arr[j] > pivot) {
      [arr[j], arr[k]] = [arr[k], arr[j]];
      k--;
    } else {
      j++;
    }
  }

  return [i, k];
}
```
