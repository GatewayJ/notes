---
author : "jihongwei"
tags : ["算法"]
date : 2020-12-20T11:24:00Z
title : "selection-sort"
---

## 二、经典算法:选择排序

#### 算法原理


选择排序依然是两层for循环。第一层循环和冒泡排序相同，任然是数组的长度。即需要将数组内的每个元素遍历一遍。
在冒泡排序里面提到一种模式，是需要一个游标将已排序和未排序的元素分开。在选择排序中第一层for循环的变量实际上就是该游标。第一层for循环元素和冒泡排序第一层for循环元素的意思不太一样，选择排序第一层for元素指的是游标，冒泡排序指的是循环的次数。


所以算法的基本思想是，在未排序的元素内找出最小的或者最大的元素，将其和已排序数组后的第一位也即未排序数组的第一位比较，如果需要调换则调换。


```golang
func main () {
    list := [...]int64{2,3,1,5,7}
    fmt.Println(list)
    for i := range (list){
        minIndex := i
        for j := i+1 ; j<len(list); j++ {
            if list[j]<list[minIndex]{
                minIndex = j
            }
        }
        list[minIndex],list[i] = list[i],list[minIndex]
    }
    fmt.Println(list)
}
```