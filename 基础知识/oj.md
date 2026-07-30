
1、滑动窗口背景知识
1.1、滑动窗口
1、该算法在一个特定大小的字符串或数组上进行操作，而不是在整个字符串或数组上操作，这样就降低了问题的复杂度，从而也达到降低循环嵌套深度
2、可以用来解决一些查找满足一定条件的连续区间的形式（长度等）的问题，由于区间连续，因此当区间发生变化时，可以通过旧有计算结果来对搜索空间进行剪枝，这样便减少了重复计算，降低了时间复杂度，往往类似与“请查找XX的最X区间（自串、子数组）”这类问题，都可以使用改方法
3、解题思路
在字符串S中使用双指针中左右指针技巧，初始化时 Left = Right = 0，使搜索区间【Left，Right】称为一个窗口
先不断增加Right指针，扩大窗口【Left，Right】直到窗口中的字符串符合要求
此时先停止Right增加，转而不断增加Left指针，缩小【Left，Right】窗口，直到窗口中的字符串不在满足要求，同时每次增加Left更新一轮结果
重复上述步骤，直到Right到达S尾部
代码模板
2、滑动窗口经典题目
1、至多包含K个字符的问题
1、159. 至多包含两个不同字符的最长子串 【中等】
2、340. 至多包含 K 个不同字符的最长子串 【中等】
3、题解
每次在【Left，Right】内统计不同的字符出现的次数count，当count > 2不满足条件时，开始移动Left，直count <=2，每次更新同步更新【Left，Right】的长度，Right一直到字符串尾部
int countDiff(char *s, int start, int end)
{
    int count = 0;
    int array[200] = { 0 };
    for (int i = start; i < end; i++) {
        if (array[s[i]] == 0) {
            array[s[i]]++;
            count++;
        }
    }
    return count;
}

int lengthOfLongestSubstringTwoDistinct(char *s)
{
    int slen = strlen(s);
    int left = 0, right = 0, maxLen = 0;
    while (right < slen) {
        int tmpLen = countDiff(s, left, right);
        while(tmpLen > 2) {
            left++;
            tmpLen = counDiff(s, left, right);
        }
        maxLen = fmax(maxLen, right - left + 1);
        right++;
    }
}
2、right沿着字符串一直往右扩展，在扩展的同时统计不同字符出现的次数count，以及每个字符出现的次数num[char]，当count>2时，left向右移动，同时减少num[char-]-left的值，如果该值为0，就表明left移动过一个字符，count-1，直到right扩展到尾部
int lengthOfLongestSubstringTwoDistinct(char *s)
{
    int len = strlen(s);
    int left = 0, right = 0, maxLen = 0, count = 0;
    int charTable[256] = { 0 };
    while(right < len) {
        if (charTable[s[right]] == 0) {
            count++;
        }
        charTable[s[right]]++;
        while(count > 2) {
            charTable[s[left]]--;
            if (charTable[s[left]] == 0) {
                count--;
            }
            left++;
        }
        right++;
        maxLen = fmax(maxLen, right = left)
    }
    return maxLen;
}
3、包含K个，更通用的写法
int lengthOfLongestSubstringTwoDistinct(char *s)
{
    int len = strlen(s);
    int left = 0, right = 0, maxLen = 0, count = 0;
    int charTable[256] = { 0 };
    while(right < len) {
        if (charTable[s[right]] == 0) {
            count++;
        }
        charTable[s[right]]++;
        while(count > K) {
            charTable[s[left]]--;
            if (charTable[s[left]] == 0) {
                count--;
            }
            left++;
        }
        right++;
        maxLen = fmax(maxLen, right = left)
    }
    return maxLen;
}
2、【中等】1151. 最少交换次数来组合所有的 1
1：题目链接：https://leetcode-cn.com/problems/minimum-swaps-to-group-all-1s-together/
2：题目描述
3：题解
1、首先统计1的个数，假设为k，那么最终所有的1聚集在一起的时候一定是一个长度为k的连续区间或者说窗口
2、我们用一个大小为k的窗口从左到右扫描一遍数组，找到窗口内1的个数最多的窗口即是答案，假设最终某个包含1最多的窗口内1的个数为N，那么调整数为K-N，
也就是把K-N个0与窗口外的同样数目的1调换位置即可
4：代码实现
int minSwaps(int* data, int dataSize){
    /* 求出当前序列中1的所有个数 k */
    int k = 0;
    for (int i = 0; i < dataSize; i++) {
        k += data[i];
    }
    /* 以k为窗口大小统计在这个窗口中1的个数 */
    int oneCount= 0;
    for (int i = 0; i < k; i++) {
        oneCount += data[i];
    }
    /* 记录第一个窗口中的1的个数 */
    int num = oneCount;
    /* 滑动窗口来计算窗口中1个数最大值 */
    int left = 0, right = k -1;
    while (right < dataSize - 1) {
        oneCount += data[++right] - data[left++];
        num = fmax(num, oneCount);
    }
    return k - num;
}
3、【中等】1100. 长度为 K 的无重复字符子串
1：题目链接：https://leetcode-cn.com/problems/find-k-length-substrings-with-no-repeated-characters/
2：题目描述：
3：题解
/*
int GetCount(char *s, int left, int k) {
    int right = fmin((left + k), strlen(s));
    int charTable[256] = { 0 };
    int num = 0;
    while (left < right) {
        if (charTable[s[left]] == 0) {
            num++;
        }
         charTable[s[left++]]++;
    }
    return num;
}
int numKLenSubstrNoRepeats(char * s, int k){
    int len = strlen(s);
    if (len == 0 || len < k) {
        return 0;
    }
    int charTable[256] = { 0 };
    int right = 0, left = 0, count = 0, num = 0;
    while (left <= len - k) {
        int count = GetCount(s, left, k);
        if (count == k) {
            num++;
        }
        left++;
    }
    return num;
}
*/
int numKLenSubstrNoRepeats(char * s, int k){
    int len = strlen(s);
    if (len == 0 || len < k) {
        return 0;
    }
    int charTable[256] = { 0 };
    int right = 0, left = 0, count = 0, num = 0;
    while (right < len) {
        if (charTable[s[right]] == 0) {
            count++;
        }
        charTable[s[right]]++;
        if ((right - left + 1) == k) {
            if (count == k) {
                num++;
            }
            charTable[s[left]]--;
            if (charTable[s[left]] == 0) {
                count--;
            }
            left++;
        }
        right++;
    }
    return num;
}
4、【中等】1004. 最大连续1的个数 III
1：题目链接：https://leetcode.cn/problems/max-consecutive-ones-iii/
2：题目描述
3：题解
对于数组A的区间[left,right]而言，只要它包含不超过k个0，我们就可以根据它构造出一段满足要求，并且长度为right - left + 1的区间，因此
我们可以将该问题进行如下转换
对于任意的右端点right，希望找到最小的左端点left，是的【left，right】包含不超过k个0
只要我们枚举所有可能的右端点，将得到的区间的长度取最大值，即可得到答案
4：代码实现
#define MAX(a,b) (a) > (b) ? (a) : (b)
int longestOnes(int* A, int ASize, int K){
    int left = 0, right = 0;
    int max = -1;
    int count = 0;
    while (right < ASize) {
        if (A[right] == 0) {
            count++;
        } 
        right++;
        while (count > K) {
            if (A[left] == 0){
                count--;
            }
            left++;
        }
        max = MAX(max, (right - left));
    }
    return max;
}
5、【中等】3. 无重复字符的最长子串
1：题目链接：https://leetcode.cn/problems/longest-substring-without-repeating-characters/
2：题目描述：
3：题解
子串：原序列中必须连续的一段
子序列：原序列中可以不连续的一段
注意：无论是子串和子序列，元素的顺序都是原序列中的顺序
方法一：利用HashMAP，标识访问过的元素，当出现相同时则计算当前子串的长度，遍历完成之后求最大值
int lengthOfLongestSubstring_Array(char * s)
{
    int len = strlen(s);
    if (len <= 1) {
        return len;
    }
    int ret[256] = { 0 };
    int i = 0, max = 0, k= 0;
    while (i < len) {
        int j = i, count  = 0;
        while (j < len && ret[s[j]] == 0) {
            count++;
            ret[s[j]]++;
            j++;
        }
        max = fmax(max, count);
        int t = i;
        while (t < j) {
            ret[s[t]] = 0;
            t++;
        }
        i++;
    }
    return max;
}
方法二：滑动窗口
1、利用两个指针标识字符串中的某个子串的左右边界， 左指针代表子串的起始位置，右指针代表子串的结束位置
2、每一步操作中，左指针移动一格代表开始枚举下一个字符作为子串的起始位置， 在移动右指针时需要保证左右指针之间的子串中无重复字符， 遇到重复字符时则计算左右之间之间的长度，并同时移动左指针
3、依次枚举每个字符求出最长的子串长度即可
int lengthOfLongestSubstring_MW(char * s)
{
    int res = 0;
    int len = strlen(s);
    /** 存储 ASCII 字符在子串中出现的次数 */
    int freq[256] = {0};
    /** 定义滑动窗口为 s[l...r] */
    int l = 0, r = -1;
    while (l < len) {
        /** freq 中不存在该字符，右边界右移，并将该字符出现的次数记录在 freq 中 */
        if (r < len - 1 && freq[s[r + 1]] == 0) {
            freq[s[++r]]++;
        /** 右边界无法拓展，左边界右移，刨除重复元素，
           并将此时左边界对应的字符出现的次数在 freq 的记录中减一
       */
        } else {
            freq[s[l++]]--;
        }
        /** 当前子串的长度和已找到的最长子串的长度取最大值 */
        res = fmax(res, r - l + 1);
    }
    return res;
}
6、【中等】209. 长度最小的子数组
1：题目链接：https://leetcode.cn/problems/minimum-size-subarray-sum/
2：题目描述：
3：代码实现
int minSubArrayLen(int target, int* nums, int numsSize){
    if (nums == NULL) {
        return 0;
    }
    int minLen = numsSize + 1, left = 0, right = 0, sum = 0;
    while (right < numsSize) {
        sum += nums[right];
        right++;
        while (sum >= target) {
            minLen = fmin(minLen, right - left);
            sum -= nums[left++];
        }
    }
    return (minLen == numsSize + 1) ? 0 : minLen;
}
7、【中等】1456. 定长子串中元音的最大数目
1：题目链接：https://leetcode.cn/problems/maximum-number-of-vowels-in-a-substring-of-given-length/
2：题目描述
3：代码实现
int isYuanYin(char c) {
    return (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') ? 1 : 0;
}
int maxVowels(char * s, int k){
    if (s == NULL) {
        return 0;
    }
    int right = 0, sum = 0, maxSum = 0;
    int len = strlen(s);
    while (right < len) {
        sum += isYuanYin(s[right]);
        right++;
        if (right >= k) {
            maxSum = fmax(maxSum, sum);
            sum -= isYuanYin(s[right - k]);
        }
    }
    return maxSum;
}
8、【中等】1208. 尽可能使字符串相等
1：题目链接：https://leetcode.cn/problems/get-equal-substrings-within-budget/
2：题目描述
3：题解
1、滑动窗口的思想是：
维护两个指针Start和End表示数组Diff的子数组的开始下标和结束下标， 满足子数组的元素和不超过MaxCost 子数组的长度是End - Start + 1,初始时Start 和 End的值是0 还需要维护子数组的元素和Sum 初始值为0，在移动两个指针的过程中，更新Sum值，判断子数组的元素和是否大于MaxCost，并决定应该如何移动指针
2、为了符合要求的最长子数组的长度，应遵循以下两点原则
1、当Start的值固定时，End的值应尽可能大
2、当End的值固定时，Start的值应尽可能小
3、具体实操
1、将Diff[i]的值累加到Sum
2、如果 Sum <= MaxCost,则表示子数组的元素和不超过MaxCost，使用当前的子数组的长度 End - Start + 1更新最大子数组的长度
3、如果 Sum > MaxCostm,则表示子数组的元素和大于MaxCost，需要向右移动Start 并同时更新Sun的值，直到Sum <= MaxCost,此时子数组的元素和不超过MaxCost，使用子数组的长度 End - Start + 1 更新最大子数组的长度
4、将指针End右移一位，直到数组尾部
5、遍历结束后即可得到符合要求的最长子数组的长度
4：代码实现
int equalSubString(char *s, char *t, int maxCost)
{
    int lens = strlen(s);
    int i = 0, j = 0;
    int sumValue = 0, maxLen = 0;
    while (i < lens && j < lens && i <= j) {
        sumValue += fabs(s[j] - t[j]);
        if (sumValue > maxCost) {
            while (sumValue > maxCost) {
                sumValue -= fabs(s[i] - t[i]);
                i++;
            }
        }
        maxLen = fmax(maxLen, j - i + 1)；
        j++;
    }
    return maxLen;
}
3、差分背景知识
数列的前N项和：Sn = a1 + a2 + a3 + ...+ an
S5 = a1 + a2 + a3 + a4 + a5
S2 = a1 + a2
S5 - S2 = a3 + a4 + a5
==>>推测
Si - Sj = a(j+1) + a(j+2) + a(j+3) + ... +a(i)
perSum[0] = 0
perSum[1] = perSum[0] + nums[0]
perSum[2] = perSum[1] + nums[1]
perSum[3] = perSum[2] + nums[2]
perSum[4] = perSum[3] + nums[3]
perSum[5] = perSum[4] + nums[4]
perSum[6] = perSum[5] + nums[5]
perSum[5] - perSum[2] = perSum[4] + nums[4] - perSum[1] - nums[1]
                                               = perSum[3] + nums[3] + nums[4] - perSum[1] - nums[1]
                                               = perSum[2] + nums[2] + nums[3] + nums[4] - perSum[1] - nums[1]
                                               = perSum[1] + nums[1] + nums[2] + nums[3] + nums[4] - perSum[1] - nums[1]
                                               = nums[2] + nums[3] + nums[4]
4、差分经典题目
1、【简单】724. 寻找数组的中心下标
1：题目链接：https://leetcode.cn/problems/find-pivot-index/
2：题目描述
3：题解
方法：前缀和
1、整个序列和为SUM, 则num[i]的左侧和为LeftSum,则右侧和为 RightSum = SUM - LeftSum - nums[i]
2、中心下标是数组的一个下标，其左侧所有元素相关的和等于右侧所有元素相加的和
2、LeftSum = RightSum = SUM - LeftSum - nums[i] => 2 * LeftSum = SUM - nums[i]
4：代码实现
int pivotIndex(int* nums, int numsSize){
    int sum = 0, i =0, leftSum = 0;
    while (i < numsSize) {
        sum += nums[i++];
    }
    for (i = 0; i < numsSize; i++) {
        if ((sum - leftSum - nums[i]) == leftSum) {
           return i;
        }
        leftSum += nums[i];
    }
    return -1;
}
2、【中等】1109. 航班预订统计
1：题目链接;https://leetcode.cn/problems/corporate-flight-bookings/
2:题目描述：
3：题解
4：代码实现
int* corpFlightBookings(int** bookings, int bookingsSize, int* bookingsColSize, int n, int* returnSize){
    *returnSize = 0;
    if (bookings == NULL || n == 0) {
        return NULL;
    }
    int *answer = (int *)malloc(sizeof(int) * (n + 1));
    memset(answer, 0, sizeof(int) * (n + 1));
    /** 暴力破解方法：超时
    for (int i = 0; i < bookingsSize; i++) {
        int start = bookings[i][0] -1, end = bookings[i][1], value = bookings[i][2];
        while (start < end) {
            answer[start++] += value;
        }
    }*/
    /** [L,R]之间增减X时，只需令查分数组d[], d[L] + X , d[R + 1] - X */
    for(int i = 0; i < bookingsSize; i++) {
        int L = bookings[i][0] - 1;
        int R = bookings[i][1] - 1;
        int value = bookings[i][2];
        answer[L] += value;
        if (R != n -1) {
            answer[R + 1] -= value;
        }
    }
    /** 求查分数组的前缀和，SUM(i) = a[i]  a[]代表原数组 */
    for (int i = 1; i < n; i++) {
        answer[i] = answer[i-1] + answer[i];
    }
    *returnSize = n;
    return answer;
}
3、【中等】1094. 拼车
1：题目链接：https://leetcode.cn/problems/car-pooling/
2：题目描述：
3：题解
4：代码实现
bool carPooling(int** trips, int tripsSize, int* tripsColSize, int capacity){
    int *map = (int *)malloc(sizeof(int) * 1001);
    memset(map, 0, sizeof(int) * 1001);
    
    for (int i = 0; i < tripsSize; i++) {
        int L = trips[i][1];
        int R = trips[i][2];
        map[L] += trips[i][0];
        map[R] -= trips[i][0];
    }
    int num = 0;
    for (int i = 0; i < 1001; i++) {
        num += map[i];
        if (num > capacity) {
            return false;
        }
    } 
    return true;
}
4、【中等】974. 和可被 K 整除的子数组
1：题目链接：https://leetcode.cn/problems/subarray-sums-divisible-by-k/
2：题目描述：
3：题解
4：代码实现
int subarraysDivByK(int* nums, int numsSize, int k){
    if (numsSize == 0) {
        return numsSize;
    }
    int *map = (int *)malloc(sizeof(int) *(numsSize + k));
    memset(map, 0, sizeof(int) * (numsSize + k));
    map[0] = 1;
    int preSum = 0, count = 0, key = 0;
    for (int i = 0; i < numsSize; i++) {
        preSum += nums[i];
        /** +K是纠正当preSum为负数时key不合法的情况 */
        key = (preSum % k + k) % k;
        count += map[key]++;
    }
    return count;
}
5、【中等】1248. 统计「优美子数组」
1：题目链接：https://leetcode.cn/problems/count-number-of-nice-subarrays/
2：题目描述：
3：题解
LeetCode 560中是求和为K的子数组，此题要求我们求[恰好有K个奇数数字的连续子数组],
560题中我们将将前缀区间的和、以及该[前缀和]出现的次数分别做[Key, Value]保存在HashMap中，
此题中[Key, Value]
Key： 前缀区间中含有多少个奇数作为Key，
Value：含有Key个奇数的前缀区间的个数作为Value
4：代码实现
int numberOfSubarrays(int* nums, int numsSize, int k){
    if (numsSize == 0) {
        return 0;
    }
    /** 数组中的奇数个数肯定不会超过原数组的长度，用数组代替HashMap */
    int *map = (int *)malloc(sizeof(int) * numsSize * 2);
    memset(map, 0, sizeof(int) * numsSize);
    map[0] = 1;
    int count = 0, preNum = 0;
    for (int i = 0; i < numsSize; i++) {
        // 用 & 比 % 效率高
        // preNum += (nums[i] % 2);
        preNum += (nums[i] & 1);
        if (preNum - k >= 0) {
            count += map[preNum -k];
        }
        map[preNum]++;
    }
    return count;
}
6、【中等】560. 和为 K 的子数组
1：题目链接：https://leetcode.cn/problems/subarray-sum-equals-k/
2：题目描述：
3：题解
定义pre[i]为[0,...,i]里所有数的和，则pre[i] 可以由pre[i-1]递推而来，即
pre[i] = pre[i-1] + nums[i]
那么[j,,,i]这个子数组和为k这个条件可以转换为
pre[i] - pre[j-1] = k ===》 pre[j-1] = pre[i] - k
考虑 i 结尾的和为k的连续子数组个数时只要统计有多少个前缀和为pre[i] - k的pre[j]即可，建立哈希表mp，以和为键，出现次数为对应的值，记录pre[i] 出现的次数
从左往右更新mp边计算答案，那么以 i 结尾的答案mp[pre[i] - k]即可在O(1)时间内获得，最后的答案即为所有下标结尾的和为k的子数组个数之和
4：代码实现
class Solution {
public:
    int subarraySum(vector<int>& nums, int k) {
        unordered_map<int, int> mp;
        mp[0] = 1;
        int count = 0, pre = 0;
        for (auto& x:nums) {
            pre += x;
            if (mp.find(pre - k) != mp.end()) {
                count += mp[pre - k];
            }
            mp[pre]++;
        }
        return count;
    }
};
#define MAX_HASH_LEN 20001
#define MAX_HASH_NODE_LEN (MAX_HASH_LEN + MAX_HASH_LEN + 1)
typedef struct {
    int valid;
    int value;
    int count;
} HashNode;
HashNode g_hashMap[MAX_HASH_NODE_LEN];
int g_max;
int GetHashCode(int value) {
    int hashCode = (abs(value) * 131) % MAX_HASH_LEN;
    return hashCode;
}
int GetHashMapIndex(int value) {
    int hashCode = GetHashCode(value);
    for (int i = hashCode; i < MAX_HASH_NODE_LEN; i++) {
        if (g_hashMap[i].valid == 0) {
            return i;
        }
        if (g_hashMap[i].value == value) {
            return i;
        }
    }
    return 0;
}
void UpdateHashMap(int index, int value) {
    if (g_hashMap[index].valid == 0) {
        g_hashMap[index].valid = 1;
        g_hashMap[index].value = value;
        g_hashMap[index].count = 1;
        
        return;
    }
    
    if (g_hashMap[index].value  != value) {
        return;
    }
    
    g_hashMap[index].count++;
    return;
}
void InitHashMap() {
    g_max = 0;
    for (int i = 0; i < MAX_HASH_NODE_LEN; i++) {
        g_hashMap[i].count = 0;
        g_hashMap[i].valid = 0;
        g_hashMap[i].value = 0;
    }
    g_hashMap[0].valid = 1;
    g_hashMap[0].count = 1;
}
int subarraySum(int* nums, int numsSize, int k) {
    int index = 0, sum = 0, preSum = 0;
    
    InitHashMap();
    
    for (int i = 0; i < numsSize; i++) {
        sum += nums[i];
        index = GetHashMapIndex(sum - k);
        if (g_hashMap[index].valid == 1) {
            g_max += g_hashMap[index].count;
        }
        index = GetHashMapIndex(sum);
        UpdateHashMap(index, sum);
    }
    
    return g_max;
}
5、双指针经典题目
1、【困难】面试题 17.21. 直方图的水量
1：题目链接：https://leetcode.cn/problems/volume-of-histogram-lcci/
2：题目描述：
3：题解
方法：双指针
1、维护两个指针Left和Right，以及两个变量LeftMax和RightMax，初始时Left = 0， Right = n -1， LeftMax = 0, RightMax = 0，指针Left
只会向右移动，指针Right只会向左移动，在移动指针的过程中维护两个变量LeftMax和RightMax
2、当两个指针没有相遇时，进入如下操作
2.1：使用heigh[Left] 和heigh[Right]的值更新LeftMax和RightMax的值
2.2：如果heigh[Left] < heigh[Right]则必有LeftMax < RightMax,下标Left处能接到的水量等于LeftMax - heigh[Left]，将下标Left处能接的水的量
加到能接的水的总量上，然后将Left右移一位
2.3：如果heigh[Left] >= heigh[Right]则必有LeftMax >= RightMax,下标Right处能接到水量等于RightMax - heigh[Right]，将下标 Right处能接的水的量
加到能接的水的总量上，然后将Right左移一位
3、当两个指针相遇时即可得到能接的水的总量
4：代码实现
int trap(int* height, int heightSize){
   int ans = 0;
    int left = 0, right = heightSize - 1;
    int leftMax = 0, rightMax = 0;
    while (left < right) {
        leftMax = fmax(leftMax, height[left]);
        rightMax = fmax(rightMax, height[right]);
        if (height[left] < height[right]) {
            ans += leftMax - height[left];
            ++left;
        } else {
            ans += rightMax - height[right];
            --right;
        }
    }
    return ans;
}
2、【中等】11. 盛最多水的容器
1：题目链接：https://leetcode.cn/problems/container-with-most-water/
2：题目描述：
3：题解
方法：双指针
1、定义两个指针指向数组的左右边界
2、对应数字较小的那个指针丢弃并移动该指针
4：代码实现
#define MIN(a, b) ((a) > (b) ? (b) : (a))
#define MAX(a, b) ((a) > (b) ? (a) : (b))
int maxArea(int* height, int heightSize){
    int ara = 0;
    for (int i = 0, j = heightSize -1; i < j; ) {
        ara = MAX(ara, MIN(height[i], height[j]) * (j - i));
        if (height[i] < height[j]) {
            i++;
        } else {
            j--;
        }
    }
    return ara;
}
3、【中等】424. 替换后的最长重复字符
1：题目链接：https://leetcode.cn/problems/longest-repeating-character-replacement/
2：题目描述
3、题解
方法一：双指针
可以枚举字符串中的每一个位置作为右端点，然后找到其最远的左端点位置，满足该区间内除了出现次数最多的那一类字符外，剩余的字符(即非最长重复字符)数量不超过K个。
这样可以使用双指针维护这些区间，每次右指针右移如果区间仍然满足条件，那么左指针不移动，否则左指针至多右移一格，保证区间长度不减小。虽然这样的操作会导致部分区间不符合条件，即该区间内非最长重复字符超过K个，但这样的区间也同样不可能对答案产生贡献，当我们右指针移动到尽头，左右指针对应的区间的长度必然对应一个长度最大的符合条件的区间，实际代码中，由于字符串仅包含大写字母，可以使用一个长度为26的数组维护每一个字符的出现次数，每次区间右移，更新右移位置的字符出现的次数，然后尝试用它更新重复字符出现次数的历史最大值，最后我们使用该最大值计算区间内非最长重复字符的数量，以此判断左指针是否需要右移即可
4、代码实现
#define MAX(a, b) (a) > (b) ? (a) : (b)
int characterReplacement(char * s, int k){
    int len = strlen(s);
    int left = 0, right = 0;
    int maxLen = 0, count = 0;
    int ret[26] = { 0 };
    while (right < len) {
        ret[s[right] - 'A']++;
        maxLen = MAX(maxLen, ret[s[right] - 'A']);
        if ((right - left + 1) > (maxLen + k)) {
            ret[s[left] - 'A']--;
            left++;
        }
        right++;
    }
    return right - left;
}
4、【中等】19. 删除链表的倒数第 N 个结点
1：题目链接：https://leetcode.cn/problems/remove-nth-node-from-end-of-list/
2：题目描述：
3：题解
1、可以使用两个指针First和Second同时对链表进行遍历，并且First比Second超前N个节点， 当First遍历到链表的末尾时，Second就恰好处于倒数第N个节点
2、具体地，初始时First 和 Second均指向头节点。首先使用 First 对链表进行遍历，遍历的次数为 n。此时First 和 Second之间间隔了 n-1个节点，即First 比Second超前了 n 个节点。
在这之后，同时使用First和Second 对链表进行遍历。当First遍历到链表的末尾（即First为空指针）时，Second恰好指向倒数第 n 个节点
4：代码实现
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
struct ListNode* removeNthFromEnd(struct ListNode* head, int n){
    if (n == 0) {
        return head;
    }
    struct ListNode *First = NULL, *Second = NULL; /* 快慢指针 */
    struct ListNode *fort = NULL; /* 待删除节点的前驱节点 */
    for (First = head; n > 0; n--) {
        First = First->next;
    }
    Second = head;
    while(First) {
        fort = Second;
        Second = Second->next;
        First = First->next;
    }
    if (Second == head) {
        head = head->next;
    } else {
        fort->next = Second->next;
        free(Second);
    }
    return head;
}
5、【中等】删除有序数组中的重复项
1:题目链接：
https://leetcode.cn/problems/remove-duplicates-from-sorted-array-ii/
https://leetcode.cn/problems/remove-duplicates-from-sorted-array/
2：题解
/**
  思想算法：
  快慢指针
  扩展：如果每个元素最多出现K次
        if (i < k || nums[i - k] < num)
**/
int removeDuplicates80_1(int* nums, int numsSize)
{
    /** 题目要求最多出现2次，若总长度小于2，直接输出即可 **/
    if (numsSize <= 2) {
        return numsSize;
    }
    /** 定义快慢指针，按题意皆从2开始 **/
    int slow = 2, fast = 2;
    while (fast < numsSize) {
        /**
        题目要求最多两个数相同，如果 nums[slow - 2] == nums[fast]
        代表已有两个数相等，则nums[fast]就不能放入nums中，否则
        就可以放入，慢指针前移
        **/
        if (nums[slow - 2] != nums[fast]) {
            nums[slow] = nums[fast];
            slow++
        }
        fast++;
    }
    return slow;
}
/**
在removeDuplicates80_1原始的快慢指针的基础上优化代码
**/
int removeDuplicates80_2(int* nums, int numsSize)
{
    int k = 0;
    for (int i = 0; i < numsSize; i++) {
        /** 用小于号判断是因为原数组有序 **/
        if (k < 2 || nums[k - 2] < nums[i]) {
            nums[k++] = num[i];
        }
    }
    return k;
}
/**
LeetCode 26
**/
int removeDuplicates26_1(int* nums, int numsSize)
{
    int k = 0;
    for (int i = 0; i < numsSize; i++) {
        /** 用小于号判断是因为原数组有序 **/
        if (k < 1 || nums[k - 1] < nums[i]) {
            nums[k++] = num[i];
        }
    }
    return k;
}
6、【中等】15. 三数之和
1：题目链接：https://leetcode.cn/problems/3sum/
2：题目描述
3：题解
方法：双指针+排序
1、题目要求不重复
第2重循环枚举到的元素不小于当前第1重枚举到的元素
第3重循环枚举到的元素不小于当前第2重枚举到的元素
枚举的三元组(a,b,c)满足a <= b <= c,为了保证这样的顺序将原始序列排序
2、第一重循环，枚举每个元素0,1,2,....,N-3
第二重循环，在第一重的基础上枚举1,2,3,,,,N-2
第三重循环，在第二重的基础上枚举2,3,,,,N-1
其中第二重与第三重循环可以使用双指针来降低复杂度
4：代码实现
int CompareByIncrease(const void* a, const void* b)
{
    return *(int*)a - *(int*)b; // 快排构造递增序列
}
int** threeSum(int* nums, int numsSize, int* returnSize, int** returnColumnSizes) {
    // 参数returnSize用来作为二维数组行数下标的指针
    // 排除[]时只输出半个]的平台bug
    *returnSize = 0;
    // 排除少于3个数的情况
    if( numsSize < 3 )
    {
        return NULL;
    }
    
    int cur = 0; // 相当于自变量，low和high的和相当于它的相反数
    int low = cur + 1;
    int high = numsSize - 1;
    
    // LeetCode 用realloc报错,malloc本以该分配A(3)(numsSize)排列空间，但是还报错,无奈用了平方，有什么解决办法？
    int** returnArray = (int**)malloc(sizeof(int*) * (numsSize)*(numsSize));
    *returnColumnSizes = (int*)malloc(sizeof(int) * (numsSize)*(numsSize)); // 新版增加的参数，用来返回列数？还不能直接用 
    // 调用stdlib库的快速排序函数
    qsort(nums, numsSize, sizeof(int), CompareByIncrease);
    
    // 第一层循环用来遍历cur，第二层循环用来双指针往中间夹
    while((nums[cur] <= 0) && (cur + 1 < numsSize - 1)) // 当相反数大于0时停下，后面的都是正数了，肯定不可能
    {
        // 每次更新
        low = cur + 1;
        high = numsSize - 1;
        
        while(low < high)
        {
            if(0 == (nums[low] + nums[high] + nums[cur]))
            {
                returnArray[*returnSize] = (int*)malloc(sizeof(int) * 3); // 每次找到一组，二级指针就分配三个空间
                (*returnColumnSizes)[*returnSize] = 3; // 记录列数，非常骚而吊诡的操作
                returnArray[*returnSize][0] = nums[cur];
                returnArray[*returnSize][1] = nums[low];
                returnArray[*returnSize][2] = nums[high];
                (*returnSize)++;
                
                // 去low和high的重,非常不规范的写法，但看着极端舒服
                while( (nums[low] == nums[++low]) && (low < high) ); // 往后去重
                while( (nums[high] == nums[--high]) && (low < high) ); // 往前去重
            }
            else if(0 < (nums[low] + nums[high] + nums[cur]))
            {
                high--;
            }
            else
            {
                low++;
            }
        }
        // 去cur的重，同样非常不规范的写法，但看着极端舒服
        while( (nums[cur] == nums[++cur]) && (cur + 1 < numsSize - 1) ); // 往后去重，同时cur往后移  
    }
    return returnArray;
}
6、二分法经典题目
1、【中等】1011. 在 D 天内送达包裹的能力
1:题目链接：https://leetcode.cn/problems/capacity-to-ship-packages-within-d-days/
2:题目描述：
3：题解
要求出船的最低运载能力，同时满足在Days天内运完所有包裹，可以先设船的运载量为w，初值为所有包裹重量之和，求出在运载量
为w的情况下所需要的天数D，然后不断调整w，已知随着w降低D会不断正大，当D刚好发过Days时，得到的w即为满足要求的最低运载量
4：代码实现
int getDayByWeight(int *weights, int weightsSize, int weight) {
    int days = 1;
    int w = 0;
    for (int i = 0; i < weightsSize; i++) {
        if ((w + weights[i]) > weight) {
            w = weights[i];
            ++days;
        } else {
            w += weights[i];
        }
    }
    return days;
}
int shipWithinDays(int* weights, int weightsSize, int days){
    int wMin = weights[0], wMax = 0;
 
    for (int i = 0; i < weightsSize; i++) {
        wMax += weights[i];
        wMin = fmax(wMin, weights[i]);
    }
    int lb = wMin, ub = wMax + 1, mid = 0;
    while ((ub - lb) > 1) {
        mid = (lb + ub) / 2;
        if (getDayByWeight(weights, weightsSize, mid) <= days) {
            ub = mid; // 当前船的运载能力太大了，需要减低运输能力
        } else {
            lb = mid; // 当前船的运载能力太小了，需要增大运输能力
        }
    }
    return getDayByWeight(weights, weightsSize, lb) <= days ?  lb : ub;
}
7、动态规划背景知识
● 【案例】蛋糕最高售价
【题解】
1、设重量为n蛋糕的售价为p(n), 切分的最高售价为f(n)
子问题：f（n）的子问题包括f(0)、f(1)、、、、f(n-1)分别代表重量为0,1，，，n-1蛋糕的最高售价，已知无蛋糕时f(0)=0，蛋糕重量为1时，1不可切分f(1)=p(1)
最优子结构：
定义：如果一个问题最优解可以由其子问题最优解组合构成，那么称此问题具有最优子结构
对于本题：重量为n的蛋糕的总售价可切分为n种组合，即重量为0,1,2,3，，，，n-1蛋糕最高售价加上
n，n-1，n-2，，，，1剩余重量蛋糕的售价；从这些组合中售价最高的组合便是原问题的解f（n）,这便是本题的最优子结构
状态转移方式：找出最优子结构后易构建 
// 输入蛋糕价格列表 priceList ，求重量为 n 蛋糕的最高售价
int maxCakePrice(int n, vector<int> priceList) {
    if (n <= 1) return priceList[n];  // 蛋糕重量 <= 1 时直接返回
    vector<int> dp(n + 1, 0);         // 初始化 dp 列表
    for (int j = 1; j <= n; j++) {    // 按顺序计算 f(1), f(2), ..., f(n)
        for (int i = 0; i < j; i++)   // 从 j 种组合种选择最高售价的组合作为 f(j)
            dp[j] = max(dp[j], dp[i] + priceList[j - i]);
    }
    return dp[n];
}
8、动态规划经典题目
1、【中等】丑数
1：题目链接 ：https://leetcode.cn/problems/ugly-number/ https://leetcode.cn/problems/ugly-number-ii/
2：题目描述
3：题解：
方法：动态规划
定义数组dp，其中dp[i]表示第 i 个丑数，第n个丑数即为dp[n]
由于最小的丑数的是1，即dp[1] = 1
如何得到其余丑数？定义三个指针p2,p3,p5,表示下一个丑数是当前指针指向的丑数乘以对应的质因素，初始时，三个指针的值都是1
当2<=i<=n时，令dp[i] = min(dp[p2] *2, dp[p3] * 3,dp[p5]*5)然后分别比较dp[i]和dp[p2]、dp[p3]、dp[p5]是否相等，如果相等则对应的指针加1
int nthUglyNumber(int n) {
    int dp[n + 1];
    dp[1] = 1;
    int p2 = 1, p3 = 1, p5 = 1;
    for (int i = 2; i <= n; i++) {
        int num2 = dp[p2] * 2, num3 = dp[p3] * 3, num5 = dp[p5] * 5;
        dp[i] = fmin(fmin(num2, num3), num5);
        if (dp[i] == num2) {
            p2++;
        }
        if (dp[i] == num3) {
            p3++;
        }
        if (dp[i] == num5) {
            p5++;
        }
    }
    return dp[n];
}
2、【中等】1143. 最长公共子序列
1：题目链接：https://leetcode.cn/problems/longest-common-subsequence/
2：题目描述
3、题解
方法：动态规划
最长公共子序列问题是典型的二维动态规划问题
假设字符串text1和text2的长度分别为m和n，创建m+1行n+1列的二维数组dp，其中dp[i][j]表示text1[0:i]和text2[0:j]的最长公共子序列的长度
上述表示中，text1[0:i]表示text1的长度为i的前缀；text2[0:j]表示text2的长度为j的前缀
考虑动态规划的边界情况
1、当i=0时，text1[0:0]为空，空字符和任何字符串的最长公共子序列的长度都是0，因此对任意0<=j<=n，有dp[0][j] = 0
2、当j=0时，text2[0:0]为空，同理可得，对于任意0<=i<=m，有dp[i][0] = 0
因此动态规划的边界情况是：当i=0或j=0时，dp[i][j] = 0
3、当i>0且j>0时，考虑dp[i][j]的计算
● 当text1[i-1] = text2[j-1]时，将两个相同的字符称为公共字符，考虑text1[0:i-1]和text2[0:j-1]的最长公共子序列，再增加一个字符(即公共字符)即可得到
text1[0:i]和text2[0:j]的最长公共子序列，因此dp[i][j] = dp[i-1][j-1] + 1
● 当text1[i-1] != text2[j-1]时考虑以下两项
1、text1[0:i-1]和text2[0:j]的最长公共子序列
2、text1[0:i]和text2[0:j-1]的最长公共子序列
要得到text1[0;i]和text2[0:j]的最长公共子序列，应取两项中的长度最大一项，因此dp[i][j] = max(dp[i-1][j], dp[i][j-1])
因此得到如下状态转移方程
dp[i][j] = dp[i-1][j-1] + 1 当text1[i-1] = text2[j-1]
dp[i][j] = max(dp[i-1][j],dp[i][j-1]) 当text1[i-1] != text2[j-1]
4、代码实现
int longestCommonSubsequence(char * text1, char * text2)
{
    int len1 = strlen(text1);
    int len2 = strlen(text2);
    if (len1 == 0 || len2 == 0) {
        return 0;
    }
    int dp[len1 + 1][len2 + 1];
    memset(dp, 0, sizeof(dp));
    for (int i = 1; i <= len1; i++) {
        char c1 = text1[i - 1];
        for (int j = 1; j <= len2; j++) {
            char c2 = text2[j - 1];
            if (c1 == c2) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = fmax(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[len1][len2];
}
3、【困难】LeetCode 32 最长有效括号
1：题目链接：https://leetcode.cn/problems/longest-valid-parentheses/
2：题目描述：
3：题解
方法一：动态规划
dp[i]表示以下标 i 字符结尾的最长有效括号的长度，将dp数组全部初始为0，显然有效的子串一定以 ）结尾，因为一定知道以 （ 结尾子串对应
的dp值一定为0，我们只需要求解以 ）结尾的dp值，从左往右遍历字符串求解dp值，每两个字符检查一次
1、s[i-1] = '(' && s[i] = ')' 即形如 “.....()”的字符串可以推测出 dp[i] = dp[i-1] + 2 可以这样转移是因为结尾部分是”()”是一个有效字符，并且将之前有效字符长度+2
2、s[i-1] = ')' && s[i] = ')' 即形如“....))”的字符串，可以推测出
如果s[i-dp[i-1]-1] = '(' 那么 dp[i] = dp[i-1] + 2 + dp[i-dp[i-1] -2]
如果我们考虑倒数第二个’)’是一个有效字符串的一部分记做sub_s，对于最后一个’)’如果它是一个有效子串的一部分，那它一定有一个’(‘与之对应，且的它的位置在倒数第二个’)’所在的有效字符串前面，及sub_s前面，因此如果sub_s前面恰好是’(‘，那我们的那我们就用2+sub_s的长度(dp[i-1])来更新dp[i]，同时我们也会把‘（sub_s）’之前的有效的子串的长度加上，也就是再加上dp[i-dp[i-1] -2]
最后的答案即为dp中的最大值
int longestValidParentheses(char* s) {
    int maxans = 0, n = strlen(s);
    if (n == 0) return 0;
    int dp[n];
    memset(dp, 0, sizeof(dp));
    for (int i = 1; i < n; i++) {
        if (s[i] == ')') {
            if (s[i - 1] == '(') {
                dp[i] = (i >= 2 ? dp[i - 2] : 0) + 2;
            } else if (i - dp[i - 1] > 0 && s[i - dp[i - 1] - 1] == '(') {
                dp[i] = dp[i - 1] + ((i - dp[i - 1]) >= 2 ? dp[i - dp[i - 1] - 2] : 0) + 2;
            }
            maxans = fmax(maxans, dp[i]);
        }
    }
    return maxans;
}
方法二：
始终保持栈底元素为当前已遍历过的元素中“最后一个没有被匹配的右括号的下标”，这样的做法主要是考虑边界条件的处理，栈里其他元素维护左括号的下标
1、对于遇到的每个 ‘(’将它下标直接放入栈中
2、对于遇到的每个')'先弹出栈顶元素表示匹配当前右括号
2.1、如果栈为空，说明当前右括号为没有被匹配的右括号，我们将其下标直接放入栈中来更新我们之前提到的“最后一个没有被匹配的右括号的下标”
2.2、如果栈不空，当前右括号的下标减去栈顶元素即为以该右括号结尾最长有效的括号长度
从左往右遍历字符串更新答案，需要注意如果一开始栈为空，第一个字符为左括号，将其放入栈中，就不满足“最后一个没有被匹配的右括号的下标”，为了保持
统一开始时将栈中放入-1
int longestValidParentheses(char* s) {
    int maxans = 0, n = strlen(s);
    int stk[n + 1], top = -1;
    stk[++top] = -1;
    for (int i = 0; i < n; i++) {
        if (s[i] == '(') {
            stk[++top] = i;
        } else {
            --top;
            if (top == -1) {
                stk[++top] = i;
            } else {
                maxans = fmax(maxans, i - stk[top]);
            }
        }
    }
    return maxans;
}
方法三：
利用两个计数器Left、Right首先从左往右遍历字符串，对于遇到的每个‘(’ Left增加对于遇到的每个‘)’ Right增加，每当Left == Right时我们就计算有效字符串的长度，
并且记录目前为止找到的最长子字符串，每当Right > Left时，Left、Right计数器同时置0，这样做贪心的考虑了当前字符下标结尾的有效括号长度，每次当右括号数量
多余左括号数量的时候之前的字符我们都扔掉不再考虑，重新从下一个字符开始计算，但这样会漏掉一种情况，就是遍历的时候左括号的数量始终大于右括号的数量，即(()
这种时候最长有效括号就求不出来了，解决方法就是需要从右往左用类似的方法计算即可，指示这个时候判断条件修改为
1、当Left > Right时，将Left、Right置0
2、当Left == Right时，计算当前有效字符串的长度，并且记录目前为止找到的最长子字符串
这样就涵盖了所有情况，从而求出答案
int longestValidParentheses(char* s) {
    int n = strlen(s);
    int left = 0, right = 0, maxlength = 0;
    for (int i = 0; i < n; i++) {
        if (s[i] == '(') {
            left++;
        } else {
            right++;
        }
        if (left == right) {
            maxlength = fmax(maxlength, 2 * right);
        } else if (right > left) {
            left = right = 0;
        }
    }
    left = right = 0;
    for (int i = n - 1; i >= 0; i--) {
        if (s[i] == '(') {
            left++;
        } else {
            right++;
        }
        if (left == right) {
            maxlength = fmax(maxlength, 2 * left);
        } else if (left > right) {
            left = right = 0;
        }
    }
    return maxlength;
}


