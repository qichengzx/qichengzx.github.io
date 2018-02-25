title: Go slice，struct排序
categories: golang
date: 2018-02-25 21:40:16
tags:  [go]
---

Go中有时会需要对slice，或多个struct进行排序，其实很简单。

## slice
对于 slice 的排序，可以直接使用 sort 包提供的方法，

#### int
```Go
s := []int{3,2,4,1}
sort.Ints(s)
fmt.Println(s) // [1,2,3,4]
```

#### string
```Go
s := []string{"Go", "Bravo", "Gopher", "Alpha", "Grin", "Delta"}
sort.Strings(s)
fmt.Println(s) // [Alpha Bravo Delta Go Gopher Grin]
```

#### float
```GO
s := []float64{5.2, -1.3, 0.7, -3.8, 2.6} 
sort.Float64s(s)
fmt.Println(s) // [-3.8,-1.3,0.7,2.6,5.2]
```

## struct

以上都是 sort 默认提供的方法，但是对于 struct ，就需要自己实现 sort.Interface。

```Go
type Person struct {
	Name string
	Age int
}

type byAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    family := []Person{
        {"Alice", 23},
        {"Eve", 2},
        {"Bob", 25},
    }
    sort.Sort(ByAge(family))
    fmt.Println(family) // [{Eve 2} {Alice 23} {Bob 25}]
}
```