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



#### [448. 找到所有数组中消失的数字](https://leetcode.cn/problems/find-all-numbers-disappeared-in-an-array/)

```go
func findDisappearedNumbers(nums []int) []int {
    n := len(nums)
    tmp := make([]int,n+1)
    for k,_ := range tmp {
        tmp[k] = k
    }

    for _,num := range nums {
        tmp[num] = 0
    }

    j := 0
    for _,v := range tmp {
        if v != 0 {
            tmp[j] = v
            j++
        }
    }
    return tmp[:j]
}
```

删除元素

```go
    j := 0
    for _,v := range tmp {
        if v != 0 {
            tmp[j] = v
            j++
        }
    }
    return tmp[:j]
```



#### [338. 比特位计数](https://leetcode.cn/problems/counting-bits/)

```go
func countBits(n int) []int {
    dp := make([]int,n+1)
    for i := 1; i <= n; i++ {
        dp[i] = dp[i&(i-1)] + 1
    }
    return dp
}
```



#### [461. 汉明距离](https://leetcode.cn/problems/hamming-distance/)

```go
func hammingDistance(x int, y int) int {
    i :=x^y//x和y异或,得到一个新的数(异或相同为0,不同为1,此时咱们需要统计1的个数)
    count :=0//定义数量的初始值为0
    for(i!=0){//只要i不为0,那就继续循环
        if ((i&1)==1){//如果i和1相与,值为1的话就count++
            count++
        }
        i = i>>1//i右移一位
    }
    return count

}
```

#### [915. 分割数组](https://leetcode.cn/problems/partition-array-into-disjoint-intervals/)

```go
func partitionDisjoint(nums []int) int {
    leftMax := make([]int,len(nums))
    rightMin := make([]int,len(nums))

    leftMax[0] = nums[0]
    rightMin[len(nums)-1] = nums[len(nums)-1]

    for i := 1; i < len(nums);i++ {
        leftMax[i] = max(leftMax[i-1],nums[i])
    }

    for i := len(nums)-2; i >= 0; i-- {
        rightMin[i] = min(rightMin[i+1],nums[i])
    }

    res := len(nums)
    for i := 0; i < len(nums)-1; i++ {
        if leftMax[i] <= rightMin[i+1] {
            res = min(res,i)
        }
    }
    return res+1
}

func max(a,b int) int { 
    if a > b {
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

#### [207. 课程表](https://leetcode.cn/problems/course-schedule/)

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    var (
        // edges[i][],表示学完课程i，可以学习的课程
        edges = make([][]int, numCourses)
        indeg = make([]int, numCourses) 
        result []int
    )

    for _, info := range prerequisites {
        // 
        edges[info[1]] = append(edges[info[1]], info[0])
        // 入度+1
        indeg[info[0]]++
    }

    q := []int{}
    for i := 0; i < numCourses; i++ {
        if indeg[i] == 0 {
            q = append(q, i)
        }
    }

    for len(q) > 0 {
        u := q[0]
        q = q[1:]
        result = append(result, u)
        for _, v := range edges[u] {
            indeg[v]--
            if indeg[v] == 0 {
                q = append(q, v)
            }
        }
    }
    return len(result) == numCourses
}
```



#### [208. 实现 Trie (前缀树)](https://leetcode.cn/problems/implement-trie-prefix-tree/)

```go
type Trie struct {
    children [26]*Trie
    isEnd    bool
}

func Constructor() Trie {
    return Trie{}
}

func (t *Trie) Insert(word string) {
    node := t
    for _, ch := range word {
        ch -= 'a'
        if node.children[ch] == nil {
            node.children[ch] = &Trie{}
        }
        node = node.children[ch]
    }
    node.isEnd = true
}

func (t *Trie) SearchPrefix(prefix string) *Trie {
    node := t
    for _, ch := range prefix {
        ch -= 'a'
        if node.children[ch] == nil {
            return nil
        }
        node = node.children[ch]
    }
    return node
}

func (t *Trie) Search(word string) bool {
    node := t.SearchPrefix(word)
    return node != nil && node.isEnd
}

func (t *Trie) StartsWith(prefix string) bool {
    return t.SearchPrefix(prefix) != nil
}
```

#### [238. 除自身以外数组的乘积](https://leetcode.cn/problems/product-of-array-except-self/)

```go
func productExceptSelf(nums []int) []int {
    dp := make([]int,len(nums))
    dp[0] = 1
    
    for i := 1; i <len(nums);i++ {
        dp[i] = dp[i-1] * nums[i-1]
    }

    tmp := nums[len(nums)-1]
    for i := len(nums)-2;i >= 0; i-- {
        dp[i] *= tmp
        tmp = tmp * nums[i] 
    }
    return dp
}
```

#### [41. 缺失的第一个正数](https://leetcode.cn/problems/first-missing-positive/)

原地换位

```go
func firstMissingPositive(nums []int) int {
    for i := 0; i < len(nums); i++ {
        for nums[i] > 0 && nums[i] <= len(nums) && nums[nums[i]-1] != nums[i] {
            nums[i],nums[nums[i]-1] = nums[nums[i]-1],nums[i]
        }
    } 

    for k,_ := range nums {
        if nums[k] != k + 1 {
            return k + 1
        }
    }
    return len(nums) + 1 
}
```

#### [57. 插入区间](https://leetcode.cn/problems/insert-interval/)

```go
func insert(intervals [][]int, newInterval []int) [][]int {
    res := [][]int{}

    // 左侧无重叠部分
    i := 0
    for i < len(intervals) && intervals[i][1] < newInterval[0] {
        res = append(res,intervals[i])
        i++
    }

    // 重叠部分
    for i < len(intervals) && intervals[i][0] <= newInterval[1] {
        newInterval[0] = min(intervals[i][0],newInterval[0])
        newInterval[1] = max(intervals[i][1],newInterval[1])
        i++
    }
    res = append(res,newInterval)

    // 右侧无重叠部分
    for i < len(intervals) && intervals[i][0] > newInterval[1] {
        res = append(res,intervals[i])
        i++
    }
    return res
}

func max(a,b int) int {
    if a > b {return a}
    return b
}

func min(a,b int) int {
    if a < b {return a}
    return b
}
```

#### [59. 螺旋矩阵 II](https://leetcode.cn/problems/spiral-matrix-ii/)

```go
func generateMatrix(n int) [][]int {
    top, bottom := 0, n-1
    left, right := 0, n-1
    num := 1
    tar := n * n
    matrix := make([][]int, n)
    // 初始化矩阵
    for i := 0; i < n; i++ {
        matrix[i] = make([]int, n)
    }

    // 填充数量
    for num <= tar {
        for i := left; i <= right; i++ {
            matrix[top][i] = num
            num++
        }
        top++
        for i := top; i <= bottom; i++ {
            matrix[i][right] = num
            num++
        }
        right--
        for i := right; i >= left; i-- {
            matrix[bottom][i] = num
            num++
        }
        bottom--
        for i := bottom; i >= top; i-- {
            matrix[i][left] = num
            num++
        }
        left++
    }
    return matrix
}
```

#### [80. 删除有序数组中的重复项 II](https://leetcode.cn/problems/remove-duplicates-from-sorted-array-ii/)

```go
func removeDuplicates(nums []int) int {
    var process func(k int) int
    process = func(k int) int {
        cur := 0
        for _,v := range nums {
            if cur < k || nums[cur-k] != v {
                nums[cur] = v
                cur++
            }
        }
        return cur
    }

    return process(1)
}
```

