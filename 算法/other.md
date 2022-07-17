#### [136. 只出现一次的数字](https://leetcode-cn.com/problems/single-number/)

首先针对异或运算，这里做一个知识点的总结：

任何数和自己做异或运算，结果为 00，即 a⊕a=0a⊕a=0 。
任何数和 00 做异或运算，结果还是自己，即 a⊕0=⊕a⊕0=⊕。
异或运算中，满足交换律和结合律，也就是 a⊕b⊕a=b⊕a⊕a=b⊕(a⊕a)=b⊕0=ba⊕b⊕a=b⊕a⊕a=b⊕(a⊕a)=b⊕0=b

```go
func singleNumber(nums []int) int {
    single := 0
    for _, num := range nums {
        single ^= num
    }
    return single
}
```



#### [7. 整数反转](https://leetcode-cn.com/problems/reverse-integer/)

```go
import "math"
func reverse(x int) int {
    res := 0
    for x != 0 {
        res = res * 10 + x % 10  
        x= x / 10
    }

    if res <= math.MinInt32 || res >= math.MaxInt32 {
		return 0
	}

    return res
}
```

#### [12. 整数转罗马数字](https://leetcode-cn.com/problems/integer-to-roman/)

```go
func intToRoman(num int) (res string) {
    if num == 0 {
        return ""
    }
    hash := map[int]string{
        1000:   "M",
        900:    "CM",
        500:    "D",
        400:    "CD",
        100:    "C",
        90:     "XC",
        50:     "L",
        40:     "XL",
        10:     "X",
        9:      "IX",
        5:      "V",
        4:      "IV",
        1:      "I",
    }
    indexes := []int{1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1}

    for _, index := range indexes{
        if index <= num {
            times := num / index
            num %= index
            for i := 0 ; i < times ; i++ {
                res += hash[index] 
            }
        }
    }
    return

}
```



#### [31. 下一个排列](https://leetcode-cn.com/problems/next-permutation/)

```go
func nextPermutation(nums []int)  {
    
    i := len(nums) - 2
    // 从右往左遍历，得到第一个左边小于右边的数
    for i >= 0 && nums[i] >= nums[i+1] {
        i--
    }

    // 如果i不是最后一个序列
    if i >= 0 {
        j := len(nums) - 1
        // 找到从右往左第一个比A[i]大的数
        for nums[i] >= nums[j] {
            j--
        }
        // 交换两个数
        nums[i],nums[j] = nums[j],nums[i] 
    }

    // 交换A[j:end]
    // 因为i右边的数是从右往左递增的，交换ij后，仍然保持单调递增特性
    // 此时需要从头到尾交换
    for l,r := i + 1, len(nums) - 1; l < r; l,r = l+1,r-1 {
        nums[l],nums[r] = nums[r],nums[l]
    }
}
```

#### [6. Z 字形变换](https://leetcode-cn.com/problems/zigzag-conversion/)

```go
func convert(s string, numRows int) string {
    res := make([]string,numRows)
    i := 0
    for i < len(s) {
        for j := 0; j < numRows && i < len(s); j++ {
            res[j] += string(s[i])
            i++
        }

        for j := numRows-2; j >0 && i < len(s); j-- {
            res[j] += string(s[i])
            i++
        }
    }

    var a string
    for _,j := range res {
        a += j
    }

    return a
}
```



#### [152. 乘积最大子数组](https://leetcode.cn/problems/maximum-product-subarray/)

```go
func maxProduct(nums []int) int {
    maxF, minF, ans := nums[0], nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        mx, mn := maxF, minF
        maxF = max(mx * nums[i], max(nums[i], mn * nums[i]))
        minF = min(mn * nums[i], min(nums[i], mx * nums[i]))
        ans = max(maxF, ans)
    }
    return ans
}

func max(a,b int) int {
    if a >b {
        return a
    }
    return b
}

func min(a,b int) int {
    if a < b {
        return a 
    }

    return b
}
```



#### [56. 合并区间](https://leetcode.cn/problems/merge-intervals/)

```go
func merge(intervals [][]int) [][]int {
    // 排序
    quickSort(intervals)

    var res [][]int
    for _, interval := range intervals{
        var length = len(res)
        if length == 0 || res[length-1][1] < interval[0]{
            res = append(res, interval)
        }else{
            res[length-1][1] = max(res[length-1][1], interval[1])
        }
    }

    return res
}

func quickSort(s [][]int) {
    len := len(s)
    if len < 2 {
        return
    }

    head,trip := 0, len-1
    value := s[head][0]
    for head < trip {
        if s[head+1][0] > value {
            s[head+1],s[trip] = s[trip],s[head+1]
            trip--
        }else if s[head+1][0] < value {
            s[head],s[head+1] = s[head+1],s[head]
            head++
        }else {
            head++
        }
    }
    quickSort(s[:head])
    quickSort(s[head+1:])
}

func max(a,b int) int {
    if a > b {
        return a
    }

    return b
}
```

#### [380. O(1) 时间插入、删除和获取随机元素](https://leetcode.cn/problems/insert-delete-getrandom-o1/)

```go
type RandomizedSet struct {
    // key-数字，value-数组中的下标
    m map[int]int
    nums []int
}


func Constructor() RandomizedSet {
    m := make(map[int]int)
    tmp := &RandomizedSet{m,[]int{}}
    return *tmp
}


func (this *RandomizedSet) Insert(val int) bool {
    if _,ok := this.m[val]; ok {
        return false
    }

    this.nums = append(this.nums,val)
    this.m[val] = len(this.nums)-1

    return true
}


func (this *RandomizedSet) Remove(val int) bool {
    if _,ok := this.m[val]; !ok {
        return false
    }
    this.m[this.nums[len(this.nums)-1]] = this.m[val]
    this.nums[this.m[val]] = this.nums[len(this.nums)-1]
    this.nums = this.nums[:len(this.nums)-1]
    delete(this.m,val)

    return true
}


func (this *RandomizedSet) GetRandom() int {
    return this.nums[rand.Intn(len(this.nums))]
}
```



#### [48. 旋转图像](https://leetcode.cn/problems/rotate-image/)

```go
func rotate(matrix [][]int)  {

    for i := 0; i < len(matrix); i++ {
        for j := i+1; j < len(matrix); j++ {
            matrix[i][j],matrix[j][i] = matrix[j][i],matrix[i][j] 
        }
    }

    for i := 0; i < len(matrix); i++ {
        left,right := 0,len(matrix)-1
        for left < right {
            matrix[i][left],matrix[i][right] = matrix[i][right],matrix[i][left]
            left++
            right--
        }
    }

}
```

