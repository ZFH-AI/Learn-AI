4、【中等】LeetCode 78 子集
1：链接 https://leetcode.cn/problems/subsets/
2、题目描述
3：题解
记原序列中的总数为N，原序列中每个元素Ai有两个状态，存在子集中和不存在子集中，利用1标识在子集中，0标识不在子集中，那么每个子集可以对应为长度为N
的0/1序列，第i位标识Ai是否在子集中
例子：N = 3，A ={5,2,9}
0/1序列
	
子集
	
0/1序列对应的二进制数

000
	
{}
	
0

001
	
{9}
	
1

010
	
{2}
	
2

011
	
{2,9}
	
3

100
	
{5}
	
4

101
	
{5,9}
	
5

110
	
{5,2}
	
6

111
	
{5,2,9}
	
7
4：代码实现
int** subsets(int* nums, int numsSize, int* returnSize, int** returnColumnSizes){
    int num_Size = 1<<numsSize;
    int **ans = (int **)malloc(sizeof(int *) * num_Size);
    *returnSize = num_Size;
    *returnColumnSizes = (int *)malloc(sizeof(int) * num_Size);
  
    int *tmp = (int *)malloc(sizeof(int) * numsSize);
    for (int mask = 0; mask < num_Size; mask++) {
         int index = 0;
         for (int i = 0; i < numsSize; i++) {
             if (mask & (1 << i)) {
                 tmp[index++] = nums[i];
             }
         }
         ans[mask] = (int *)malloc(sizeof(int) * index);
         memcpy(ans[mask], tmp, sizeof(int) * index);
         returnColumnSizes[0][mask] = index;
    }
    return ans;
}
5、【动态规划题目-1】剑指 Offer 42. 连续子数组的最大和
1、题目链接：https://leetcode.cn/leetbook/read/illustration-of-algorithm/59gq9c/
2、题目描述
3、题解
动态规划解析
状态定义：
设动态规划列表dp，dp[i]代表以元素nums[i]为结尾的连续子数组最大和{为何定义最大和dp[i]中必须包含元素nums[i],保证dp[i]递推到dp[i+1]的正确性
如果不包含nums[i]递推时则不满足题目的连续子数组要求}
转移方程：
若dp[i-1] <= 0 说明 dp[i-1]对dp[i]产生负贡献，即dp[i-1] + nums[i] 还不如nums[i]本身大
dp[i] = max(dp[i-1] + nums[i], nums[i])
初始状态：
dp[0] = nums[0],即以nums[0]结尾的连续子数组最大和为nums[0]
返回值：
返回dp列表中的最大值，代表全局最大值
空间复杂度：
由于dp[i] 只与dp[i-1] 和nums[i]有关系，因此将原数组nums用作dp数组，直接在nums上修改
int maxSubArray(int* nums, int numsSize){
    if (numsSize == 0) {
        return 0;
    }
    int maxValue =  nums[0];
    /*
    int *dp = (int *)malloc(sizeof(int) * (numsSize + 1));
    dp[0] = nums[0];
    for (int i = 1; i < numsSize; i++) {
        dp[i] = fmax(dp[i-1] + nums[i], nums[i]);
        maxValue = fmax(maxValue, dp[i]);
    }
    */
    for (int i = 1; i < numsSize; i++) {
        nums[i] = fmax(nums[i-1] + nums[i], nums[i]);
        maxValue = fmax(maxValue, nums[i]);
    }
    return maxValue;
}
6、【困难】LeetCode 32 最长有效括号
1：链接 https://leetcode.cn/problems/longest-valid-parentheses/
2：题目描述
3：题解
方法一：动态规划
dp[i]表示以下标 i 字符结尾的最长有效括号的长度，将dp数组全部初始为0，显然有效字符串一定以‘)’结尾，因此我们一定知道以‘(’结尾的子串对应的dp值一定为0
我们只需要求解以‘)’结尾的dp值。
从左往右遍历字符串求解dp值，每两个字符检查一次
1、s[i-1] = '(' and s[i] =')' 即形如“...()”的字符串可以推出：dp[i] = dp[i-2] + 2 可以这样转移是因为结尾部分是"()"是一个有效字符串，并且将之前有效字符长度+2
2、s[i-1] =')' and s[i]=')' 即形如"..))"的字符串，可以推出
s[i - 1 - dp[i-1]] = '(' 那么 dp[i] = dp[i -1] + dp[i - dp[i-1] - 2] +2
如果倒数第二个‘)’是一个有效字符串的一部分记为SUB_S，对于最后一个')'如果它是一个有效字符串的一部分，那它一定有一个‘(’与之对应且它的位置在倒数第二个‘)’
所在的有效字符串前面即SUB_S前面，因此如果SUB_S前面恰好是'('，那我们就用2 + SUB_S的长度长度dp[i-1]来更新dp[i]，同时我们也会把“(SUB_S)”之前的有效字符串
的长度加上，也就是再加上dp[i - 2 - dp[i-1]]，最后的答案即为dp中的最大值
方法二：栈
始终保持栈底元素为当前已遍历过的元素中“最后一个没有被匹配的右括号的下标”，这样做法主要是考虑边界条件的处理，栈里其他元素维护左括号的下标
1、对于遇到的每个‘(’直接将其下标放入栈中
2、对于遇到的每个')'我们先弹出栈顶元素表示匹配当前右括号
2.1、如果栈为空说明当前右括号未没有被匹配的右括号，将其下标直接放入栈中来更新我们之前提到的“最后一个没有被匹配的右括号的下标”
2.2、如果栈不空，说明当前右括号的下标减去栈顶元素即为以该右括号结尾最长有效的括号长度
从左到右遍历字符串即可得到答案
3、需要注意的是，如果一开始栈为空，第一个字符为左括号时将其放入栈中，这时就不满足“最后一个没有被匹配的右括号的下标”的原则，为了保持一致，
一开始时往栈中放入一个值为-1的元素
方法三：贪心
利用两个计数器Left、Right首先从左往右遍历字符串，对于每遇到‘(’ Left增加，对于每遇到‘)’ Right增加，每当Left = Right时我们就计算有效的字符串长度，并且记录目前
为止找到的最长子字符串，每当Right > Left时，Left、Right回0，这样做贪心的考虑了以当前字符下标结尾的有效括号长度，每次当右括号数量多于左括号数量的时候之前的字符我们都扔掉不再考虑，重新从下一个字符开始计算，但这样会漏掉一种情况，就是遍历的时候左括号的数量始终大于右括号的数量，即(() 这种时候最长有效括号就求不出来了，解决方法就是需要从右往左用类似的方法计算即可，只是这个时候判断条件修改为：
1、 当Left > Right时将Left、Right同时回0
2、 当Left == Right时，计算当前有效字符串的长度，并且记录目前为止找到的最长子字符串
这样就能涵盖所有情况，从而求出答案
4：代码实现
方法一代码实现
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
方法二代码实现
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
方法三代码实现
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
7、【简单】LeetCode 20 有效的括号
1：链接： https://leetcode.cn/problems/valid-parentheses/
2：题目描述
3：题解
判断括号的有效性可以使用栈来解决，在遍历给定的字符串S时，遇到左括号我们会期望在后续的遍历中有个相同类型的右括号将其闭合
由于后入的左扩号要先闭合，因此可以将左扩号存入栈顶，遇到右括号时将栈顶的左扩号取出判断是否为相同类型的括号，如果不是相同类型
或者栈中无左扩号则字符串S失效，注意到有效字符串的长度一定是偶数，因此如果字符串的长度为奇数则直接判断字符串失效
4：代码实现
bool isValid(char * s){
    if (s == NULL) {
        return false;
    }
    int len = strlen(s);
    if (len % 2 == 1) {
        return false;
    }
    char *ret = (char *) malloc(sizeof(char *) * (len + 1));
    
    int k = -1;
    for (int i = 0; i < len; i++) {
        if (i == 0) {
            ret[++k] = s[i];
            continue;
        }
        if ((k >= 0) && (((ret[k] == '(') && (s[i] == ')')) || 
            ((ret[k] == '{') && (s[i] == '}')) || 
            ((ret[k] == '[') && (s[i] == ']')))) {
            k--;
        } else {
            ret[++k] = s[i];
        }
    }
    // 当前下标指向栈顶
    if (k == -1) {
        return true;
    }
    return false;
}
8、剑指 Offer 48. 最长不含重复字符的子字符串
1：题目链接：https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/
2：题目描述
3：题解：
方法一：滑动窗口方法
1、利用两个指针标识字符串中的某个子串的左右边界
左指针代表子串的起始位置
右指针代表子串的结束位置
2、每一步操作中，左指针移动一步代表开始枚举下一个字符作为子串的起始位置，在移动右指针时需要保证左右指针之间的子串是无重复
子串，遇到重复字符时则计算左右指针之间的长度，同时开始移动左指针
3、依次遍历每个字符求出最长子串长度即可
方法二：动态规划
1、状态定义：设动态规划列表中dp，dp[j]代表以字符s[j]为结尾的“最长不重复子字符串”的长度
2、转移方程：固定右边界j，设字符s[j]左边距离最近相同字符s[i]，即s[i] ==s[j]
1、当i < 0 即s[j]左边无相同字符，则dp[j] = dp[j-1] + 1
2、当dp[j-1] < j - i,说明字符s[i]在子字符串dp[j-1]区间外，则dp[j] = dp[j-1] + 1
3、当dp[j-1] >= j -i,说明字符s[i]在子字符串dp[j-1]区间内，则dp[j]的左边界由s[i]决定，即dp[j] = j-i
3、当i < 0时，由于dp[j-1] <= j 恒成立
所以
4、返回值
max（dp）即全局的最长不重复子字符串的长度，返回dp数组的最大值，可以借助遍历temp存储dp[j]
变量res每轮更新最大值即可
4：代码实现
/** 滑动窗口方法*/
// 4ms 5.7M
int lengthOfLongestSubstring(char * s){
    int res = 0;
    int len = strlen(s);
    /* 存储 ASCII 字符在子串中出现的次数 */
    int freq[256] = {0};  
    /* 定义滑动窗口为 s[l...r] */
    int l = 0, r = -1; 
    while (l < len) {
        /* freq 中不存在该字符，右边界右移，并将该字符出现的次数记录在 freq 中 */
        if (r < len - 1 && freq[s[r + 1]] == 0) {
            freq[s[++r]]++;
        /* 右边界无法拓展，左边界右移，刨除重复元素，并将此时左边界对应的字符出现的次数在 freq 的记录中减一 */
        } else {
            freq[s[l++]]--;
        }
        /* 当前子串的长度和已找到的最长子串的长度取最大值 */
        res = fmax(res, r - l + 1);
    }
    return res;
}
int lengthOfLongestSubstring(char* s){
    if (strlen(s) <= 1) {
        return strlen(s);
    }
    int max = 0;
    int sLen = strlen(s);
    int temp = 0, i = 0;
    int intS[256] = { -1 };
    for (int n = 0; n < 256;n++) {
        intS[n] = -1;
    }
    for (int j = 0; j < sLen; j++) {
        if (intS[s[j]] == -1) {
            i = -1;
        } else {
            i = intS[s[j]];
        }
        intS[s[j]] = j;
        temp = temp < (j - i) ? (temp + 1) : (j - i);
        max = fmax(temp, max);
    }
    return max;
}
9、【中等】把数组排列成最大最小值
1：题目链接：https://leetcode.cn/problems/largest-number/
https://leetcode.cn/problems/ba-shu-zu-pai-cheng-zui-xiao-de-shu-lcof/
2：题目描述：
3、题解
● 要想把组成最大的整数，一种直观的想法就是把数值最大的数字放到高位，于是可以比较输入数组的每个元素的最高位，最高位相同
的时候比较次高位，依次类推，完成排序，然后把他们拼接起来
● 上述排查方式对于输入数组没有相同数字开头的时候是有效的，比如【45,56,81,76,123】
● 考虑输入数组有相同数字开头时，比如【4,42】和【4,45】
对于【4,42】，442 > 424,需要把4放到前面
对于【4,45】，445 < 454,需要把45放到前面
因此需要考虑不同的拼接方式，进而决定拼接序列
4、代码实现
/**
  数字 a，b
  比较：ba - ab 的大小
**/
long largeCmp(int *x, int *y)
{
    long sx = 10, sy = 10;
    while (sx <= *x) {
        sx *= 10;
    }
    while (sy <= *y) {
        sy *= 10;
    }
    return (sx * (*y) + (*x)) - (sy * (*x) + (*y));
}
char *largestNumber(int *nums, int numsSize)
{
    qsort(nums, numsSize, sizeof(int), largeCmp);
    if (nums[0] == 0) {
        char *ret = (char *)malloc(sizeof(char) * 2);
        ret[0] = '0';
        ret[1] = '\0';
        return "0";
    }
    char *ret = (char *)malloc(sizeof(char) * 1000);
    char *p = ret;
    for (int i = 0; i < numsSize; i++) {
        sprintf(p, "%d", nums[i]);
        p += strlen(p);
    }
    return ret;
}
/**
  数字 a，b
  比较：ab - ba 的大小
**/
long minCmp(int *x, int *y)
{
    long sx = 10, sy = 10;
    while (sx <= *x) {
        sx *= 10;
    }
    while (sy <= *y) {
        sy *= 10;
    }
    return (sy * (*x) + (*y)) - (sx *(*y) + (*x));
}
char *minNumber(int *nums, int numsSize)
{
    qsort(nums, numsSize, sizeof(int), minCmp);
    if (nums[0] == 0) {
        char *ret = (char *)malloc(sizeof(char) * 2);
        ret[0] = '0';
        ret[1] = '\0';
        return "0";
    }
    char *ret = (char *)malloc(sizeof(char) * 1000);
    char *p = ret;
    for (int i = 0; i < numsSize; i++) {
        sprintf(p, "%d", nums[i]);
        p += strlen(p);
    }
    return ret;
}
10、剑指 Offer 49. 丑数
1：题目链接：https://leetcode-cn.com/problems/ugly-number-ii/
2、题目描述：
3：题解
1）、丑数的递推性质：丑数只包含因子2,3,5，因此有丑数 = 某较小丑数 * 某因子
设已知长度为n的丑数序列x1，x2，，，，xn，求第N+1个丑数xn+1 根据递推性质，丑数xn+1只能是从以下三种情况其中之一
4、代码实现
int nthUglyNumber(int n){
    if (n <= 0) {
        return 0;
    }
    int *UlyNumber = (int *)malloc(sizeof(int) * 1700);
    memset(UlyNumber, 0, sizeof(UlyNumber));
    UlyNumber[0] = 1;
    int index = 1, p2 = 0, p3 = 0, p5 = 0;
    while (index < n) {
        UlyNumber[index] = fmin(UlyNumber[p2] * 2, fmin(UlyNumber[p3] * 3, UlyNumber[p5] * 5));
        while (UlyNumber[p2] * 2 <= UlyNumber[index]) {
            p2++;
        }
        while (UlyNumber[p3] * 3 <= UlyNumber[index]) {
            p3++;
        }
        while (UlyNumber[p5] * 5 <= UlyNumber[index]) {
            p5++;
        }
        index++;
    }
    return UlyNumber[index - 1];
}
11、【简单】LeetCode 20 有效的括号
1：题目链接：https://leetcode.cn/problems/valid-parentheses/
2：题目描述：
3：题解
判断括号的有效性可以使用栈来解决，在遍历给定的字符串S时，遇到左括号我们会期望在后续的遍历中有个相同类型的右括号将其闭合，
由于后入的左扩号要先闭合，因此可以将左扩号存入栈顶，遇到右括号时将栈顶的左扩号取出判断是否为相同类型的括号，如果不是相同类型或者栈中
无左扩号则字符串S无效，注意到有效字符串的长度一定为偶数，因此如果字符串的长度为奇数则直接判断字符串失效
4：代码实现
bool isValid(char * s){
    if (s == NULL) {
        return false;
    }
    int len = strlen(s);
    if (len % 2 == 1) {
        return false;
    }
    char *ret = (char *) malloc(sizeof(char *) * (len + 1));
    
    int k = -1;
    for (int i = 0; i < len; i++) {
        if (i == 0) {
            ret[++k] = s[i];
            continue;
        }
        if ((k >= 0) && (((ret[k] == '(') && (s[i] == ')')) || 
            ((ret[k] == '{') && (s[i] == '}')) || 
            ((ret[k] == '[') && (s[i] == ']')))) {
            k--;
        } else {
            ret[++k] = s[i];
        }
    }
    // 当前下标指向栈顶
    if (k == -1) {
        return true;
    }
    return false;
}
12、【中等】剑指 Offer 20. 表示数值的字符串
1：题目链接：https://leetcode.cn/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/
2：题目描述
3：题解
4：代码实现
enum State {
    STATE_INITIAL,
    STATE_INT_SIGN,
    STATE_INTEGER,
    STATE_POINT,
    STATE_POINT_WITHOUT_INT,
    STATE_FRACTION,
    STATE_EXP,
    STATE_EXP_SIGN,
    STATE_EXP_NUMBER,
    STATE_END,
    STATE_ILLEGAL
};
enum CharType {
    CHAR_NUMBER,
    CHAR_EXP,
    CHAR_POINT,
    CHAR_SIGN,
    CHAR_SPACE,
    CHAR_ILLEGAL
};
enum CharType toCharType(char ch) {
    if (ch >= '0' && ch <= '9') {
        return CHAR_NUMBER;
    } else if (ch == 'e' || ch == 'E') {
        return CHAR_EXP;
    } else if (ch == '.') {
        return CHAR_POINT;
    } else if (ch == '+' || ch == '-') {
        return CHAR_SIGN;
    } else if (ch == ' ') {
        return CHAR_SPACE;
    } else {
        return CHAR_ILLEGAL;
    }
}
enum State transfer(enum State st, enum CharType typ) {
    switch (st) {
        case STATE_INITIAL: {
            switch (typ) {
                case CHAR_SPACE:
                    return STATE_INITIAL;
                case CHAR_NUMBER:
                    return STATE_INTEGER;
                case CHAR_POINT:
                    return STATE_POINT_WITHOUT_INT;
                case CHAR_SIGN:
                    return STATE_INT_SIGN;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_INT_SIGN: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_INTEGER;
                case CHAR_POINT:
                    return STATE_POINT_WITHOUT_INT;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_INTEGER: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_INTEGER;
                case CHAR_EXP:
                    return STATE_EXP;
                case CHAR_POINT:
                    return STATE_POINT;
                case CHAR_SPACE:
                    return STATE_END;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_POINT: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_FRACTION;
                case CHAR_EXP:
                    return STATE_EXP;
                case CHAR_SPACE:
                    return STATE_END;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_POINT_WITHOUT_INT: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_FRACTION;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_FRACTION: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_FRACTION;
                case CHAR_EXP:
                    return STATE_EXP;
                case CHAR_SPACE:
                    return STATE_END;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_EXP: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_EXP_NUMBER;
                case CHAR_SIGN:
                    return STATE_EXP_SIGN;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_EXP_SIGN: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_EXP_NUMBER;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_EXP_NUMBER: {
            switch (typ) {
                case CHAR_NUMBER:
                    return STATE_EXP_NUMBER;
                case CHAR_SPACE:
                    return STATE_END;
                default:
                    return STATE_ILLEGAL;
            }
        }
        case STATE_END: {
            switch (typ) {
                case CHAR_SPACE:
                    return STATE_END;
                default:
                    return STATE_ILLEGAL;
            }
        }
        default:
            return STATE_ILLEGAL;
    }
}
bool isNumber(char* s) {
    int len = strlen(s);
    enum State st = STATE_INITIAL;
    for (int i = 0; i < len; i++) {
        enum CharType typ = toCharType(s[i]);
        enum State nextState = transfer(st, typ);
        if (nextState == STATE_ILLEGAL) return false;
        st = nextState;
    }
    return st == STATE_INTEGER || st == STATE_POINT || st == STATE_FRACTION || st == STATE_EXP_NUMBER || st == STATE_END;
}
13、【简单】剑指 Offer 06. 从尾到头打印链表
1:题目链接：https://leetcode.cn/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/
2:题目描述
3：题解
略
4：代码实现
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
/**
 * Note: The returned array must be malloced, assume caller calls free().
 */
int* reversePrint(struct ListNode* head, int* returnSize){
    int *ret = (int *)malloc(sizeof(int) * 10000);
    struct ListNode *p = head;
    int index = 0;
    while (p) {
        ret[index++] = p->val;
        p = p->next;
    }
    for (int i = 0, j = index - 1; i < j; i++, j--) {
        int temp = ret[i];
        ret[i] = ret[j];
        ret[j] = temp;
    }
    *returnSize = index;
    return ret;
}
14、【困难】剑指 Offer 59 - I. 滑动窗口的最大值
1：题目链接：https://leetcode-cn.com/problems/sliding-window-maximum/
2：题目描述
3：题解
1、设窗口区间为【i，j】 最大值为xj, 当窗口向前移动一格，则区间变为【i+1, j+1】，即添加nums[j+1]，删除nums[i]，若指向窗口【i，j】
右边添加数字nums[j+1]，则新窗口最大值可以通过一次对比得到即 xj+1 = max(xj, nums[j+1])，而由于删除的nums[i]可能恰好是窗口内唯一的最大值xj，
因此就不能通过与nums[j+1]对比获取xj+1，而必须遍历整个区间获取最大值即 xj+1 = max(nums[i+1],....nums[j+1])
2、使用单调队列在0(1)内获取窗口中最大值
算法流程：
初始化： 双端队列 deque ，结果列表 res，数组长度 n ；
滑动窗口： 左边界范围i∈[1−k,n−k] ，右边界范围j∈[0,n−1] ；
1、若 i > 0 且 队首元素 deque[0] = 被删除元素 nums[i - 1]：则队首元素出队；
2、删除 deque 内所有 < nums[j]的元素，以保持 deque 递减；
3、将 nums[j]添加至 deque 尾部；
4、若已形成窗口（即 i≥0 ）：将窗口最大值（即队首元素 deque[0] ）添加至列表 res；
返回值： 返回结果列表 res；
4：代码实现
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if(nums.length == 0 || k == 0) return new int[0];
        Deque<Integer> deque = new LinkedList<>();
        int[] res = new int[nums.length - k + 1];
        for(int j = 0, i = 1 - k; j < nums.length; i++, j++) {
            // 删除 deque 中对应的 nums[i-1]
            if(i > 0 && deque.peekFirst() == nums[i - 1])
                deque.removeFirst();
            // 保持 deque 递减
            while(!deque.isEmpty() && deque.peekLast() < nums[j])
                deque.removeLast();
            deque.addLast(nums[j]);
            // 记录窗口最大值
            if(i >= 0)
                res[i] = deque.peekFirst();
        }
        return res;
    }
}
15、【中等】64. 最小路径和
1：题目链接：https://leetcode.cn/problems/minimum-path-sum/
2：题目描述
3：题解
方法：动态规划
1、由于路径的方向只能向下或向右移动一步，因此网格的第一行的每个元素只能从左上角的元素开始向右移动到达；
网格的第一列的每个元素只能从左上角开始向下移动到达
2、对于非第一行或第一列的元素，可从其上方相邻元素向下移动一格到达或者从其左方相邻元素向右移动一格到达，元素
对应的最小路径之后等于两个方向上最小值再加上当前元素的值
3、创建动态规划二维数据dp,与原始网格大小相同，dp[i][j]表示从左上角出发到(i,j)位置的最小路径之和，显然dp[0][0]=grid[0][0],对于
其余元素可以通过状态转移方程获得
● j = 0 && i != 0时；dp[i][0] = dp[i-1][0] + grid[i][0]
● j !=0 && i = 0时；dp[0][j] = dp[0][j-1] + grid[0][j-1]
● j !=0 && i !=0时：dp[i][j] = min(dp[i-1][j], sp[i][j-1]) + grid[i][j]
最后得到dp[m-1][n-1]的值即为网格左上角到网格右下角的最小路径和
4：代码实现
#define MIN(a,b) ((a) > (b) ? (b) : (a))
int minPathSum(int** grid, int gridSize, int* gridColSize){
    if (grid == NULL || gridColSize == NULL) {
        return 0;
    }
    int rows = gridSize;
    int clos = gridColSize[0];
    if (clos == 0 || gridSize == 0 ) {
        return 0;
    }
    int dp[rows][clos];
    dp[0][0] = grid[0][0];
    for (int j = 1; j < clos; j++) {
        dp[0][j] = dp[0][j - 1] + grid[0][j];
    }
    for (int i = 1; i < rows; i++) {
        dp[i][0] = dp[i -1][0] + grid[i][0];
    }
    for (int rIndex = 1; rIndex < rows; rIndex++) {
        for (int cIndex = 1; cIndex < clos; cIndex++) {
            dp[rIndex][cIndex] = MIN(dp[rIndex - 1][cIndex], dp[rIndex][cIndex - 1]) + grid[rIndex][cIndex];
        }
    }
    
    return dp[rows - 1][clos - 1];
}
16、【中等】LeetCode 78 子集
1：题目链接：https://leetcode.cn/problems/subsets/
2：题目描述
3：题解
记原序列中的总数为n，原序列中每个元素ai有两个状态，在子集中和不在子集中，我们用1：标识在子集中；0：标识不在子集中
那么每个子集可以对应为长度为n的0、1序列，第i位标识ai是否在子集中
N = 3 a = {5， 2，9}
0/1序列
	
子集
	
0/1序列对应的二进制数

000
	
{}
	
0

001
	
{9}
	
1

010
	
{2}
	
2

011
	
{2,9}
	
3

100
	
{5}
	
4

101
	
{5,9}
	
5

110
	
{5,2}
	
6

111
	
{5,2,9}
	
7
4：代码实现
int** subsets(int* nums, int numsSize, int* returnSize, int** returnColumnSizes){
    int num_Size = 1 << numsSize;
    int **ans = (int **)malloc(sizeof(int *) * num_Size);
    *returnSize = num_Size;
    *returnColumnSizes = (int *)malloc(sizeof(int) * num_Size);
  
    int *tmp = (int *)malloc(sizeof(int) * numsSize);
    for (int mask = 0; mask < num_Size; mask++) {
         int index = 0;
         for (int i = 0; i < numsSize; i++) {
             if (mask & (1 << i)) {
                 tmp[index++] = nums[i];
             }
         }
         ans[mask] = (int *)malloc(sizeof(int) * index);
         memcpy(ans[mask], tmp, sizeof(int) * index);
         returnColumnSizes[0][mask] = index;
    }
    return ans;
}
17、【中等】322. 零钱兑换
1：题目链接：https://leetcode.cn/problems/coin-change/
2：题目描述
3：题解
F(i)为组成金额 i 所需最少的硬币数量，假设在计算F(i)之前，我们已经计算出F(0)---F(i-1)的答案，则F(i)对应的转移方程为
Cj 代表的是第 j 枚硬币的面值，即我们枚举最后一枚硬币面额是Cj，那么需要从i - Cj这个金额的状态F(i-Cj)转移过来，再算上枚举的这枚硬币数量1的贡献，
由于要硬币数量最少，所以F(i)为前面能转移过来的状态的最小值加上枚举的硬币数量1
4：代码
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        int Max = amount + 1;
        vector<int> dp(amount + 1, Max);
        dp[0] = 0;
        for (int i = 1; i <= amount; ++i) {
            for (int j = 0; j < (int)coins.size(); ++j) {
                if (coins[j] <= i) {
                    dp[i] = min(dp[i], dp[i - coins[j]] + 1);
                }
            }
        }
        return dp[amount] > amount ? -1 : dp[amount];
    }
};
18、【简单】796. 旋转字符串
1：题目链接：https://leetcode.cn/problems/rotate-string/
2：题目描述
3：题解
4：代码实现
bool rotateString(char * A, char * B){
    int aL = strlen(A);
    int bL = strlen(B);
    if (aL != bL) {
        return false;
    }
    if (strcmp(A,B) == 0) {
        return true;
    }
    char *res = (char *)malloc(sizeof(char) * (aL + bL + 1));
    res[aL + bL] = '\0';
    strcpy(res,B);
    strcat(res,B);
    return strstr(res,A);
}
19、【简单】剑指 Offer 05. 替换空格
1：链接 https://leetcode.cn/problems/ti-huan-kong-ge-lcof/
2：题目描述
3：题解
略
4：代码实现
char* replaceSpace(char* s){
    int len = strlen(s);
    if (s <= 0 || len > 10000) {
        return s;
    }
    char *ret = (char *)malloc(sizeof(char) * 10 * len);
    int index = 0;
    for (int i = 0; i < len; i++) {
        if (s[i] != ' ') {
            ret[index++] = s[i];
        }  else {
            ret[index++] = '%';
            ret[index++] = '2';
            ret[index++] = '0';
        }
    }
    ret[index] = '\0';
    return ret;
}
9、字符串经典题目
1、【中等】5. 最长回文子串
1：题目链接:https://leetcode.cn/problems/longest-palindromic-substring/
2:题目描述
3：题解
方法一：动态规划
1、对于一个子串而言，如果它是回文串，那么将它的首尾字符去掉之后仍然是回文串，例如“ababa”是回文串，那么“bab”是回文串，
p(i,j)表示字符串S的第i到j字母组成是的串是否是回文串
p(i,j) = true 则Si,...,Sj是回文串
p(i,j) = false 则非回文串【1、Si,...,Sj不是回文串，2、i>j 非合法字符串】
动态规划的状态转移方程
p(i,j) = p(i+1, j-1) && (Si == Sj)
上述状态转移是在子串长度大于2的前提下，当中子串长度小于等于1时需要判定边界条件，长度为1是回文串，长度为2两个字符相同也是回文串
char * longestPalindrome(char * s)
{
    int len = strlen(s);
    if (len < 2) {
        return s;
    }
    int maxLen = 1, start = 0;
    /* dp[i][j] 表示s[i,...,j]是否是回文串 */
    bool **dp = (bool **)malloc(sizeof(bool *) * (len +1));
    for (int i = 0; i <= len; i++) {
        dp[i] = (bool *)malloc(sizeof(bool) * (len + 1));
    }
    /* 长度为1的子串都是回文串 */
    for (int i =0; i <=len; i++) {
        dp[i][i] = true;
    }
    
    for (int L = 2; L <= len; L++) {
        for (int i = 0; i < len; i++) {
            /* L: 表示回文串的长度
               i：回文串的左边界
               j：回文串的右边界 j - i + 1 = L，j = L + i -1
            */
            int j = L + i - 1;
            if (j >= len) {
                break;
            }
            if (s[i] != s[j]) {
                dp[i][j] = false;
            } else {
                if (j - i < 3) {
                    dp[i][j] = true;
                } else {
                    dp[i][j] = dp[i + 1][j - 1];
                }
            }
            
            if (dp[i][j] && (j - i + 1) > maxLen) {
                maxLen = j - i + 1;
                start = i;
            }
        }
    }
    s[start + maxLen] = '\0';
    
    return  s + start;
}
方法二：中线扩展算法
方法一种的状态转移方程
p(i,i) = true
p(i,i+1）=（Si = Si+1）
p(i,j) = p(i+1,j-1) &(Si == Sj)
找出其中的状态转移链
p(i,j) <—— p(i+1, j-1) <——p(i+2,j-2) <——,,,,,,<——某一个边界情况
所有的状态在转移的时候可能性都是唯一的，也就是我们可以从某一种边界情况开始扩展，也可以得出所有的状态对应的答案；
边界情况即为子串长度为1或2的情况
枚举所有的回文中心并尝试扩展，直到无法扩展为止，此时的回文长度即为回文中下的最长回文串长度
对所有长度求出最大值即可得到最终答案
char * longestPalindrome(char * s){
    int length = strlen(s);
    if (length == 0 || length == 1) {
        return s;
    }
    int left = 0, right = 0, count = 0, start = 0, len = 0;
    for(int i = 0; i < length; i += count) {
        count = 1;
        left = i - 1;
        right = i + 1;
        while (s[right] != '\0' && s[i] == s[right]) {
            right++;
            count++;
        }
        while (left >= 0 && s[left] != '\0' && s[left] == s[right]) {
            left--;
            right++;
        }
        if (right - left - 1 > len) {
            start = left + 1;
            len = (right - left - 1);
        }
    }
    s[start + len] ='\0';
    return s + start;
}
2、【中等】43. 字符串相乘
1：题目链接：https://leetcode.cn/problems/multiply-strings/
2：题目描述：
3：代码实现
char *multiply(char * num1, char * num2)
{
    int len1 = strlen(num1);
    int len2 = strlen(num2);
    int totalLen = len1 + len2;
    int charIndex = 0, valueIndex = 0;
    int *value = (int *)malloc(sizeof(int) * totalLen);
    memset(value, 0, sizeof(int) * totalLen);
    char *result = (char *)malloc(sizeof(char) * (totalLen + 1));
    /** 列竖式求解 */
    for (int i = len1 -1; i >= 0; i--) {
        for (int j = len2 -1; j >= 0; j--) {
            value[i + j + 1] += (num1[i] - '0') * (num2[j] - '0');
        }
    }
    /**获取每个位置上面的数字处理并考虑进位 */
    for (int i = totalLen -1; i > 0; i--) {
        value[i - 1] += value[i] / 10;
        value[i] %= 10;
    }
    /** 忽略掉前面多余的0，但是最高位也就是唯一的一位0不能忽略 */
    while (value[valueIndex] == 0 && valueIndex < totalLen -1) {
        valueIndex++;
    }
    /** 将整型转为字符 */
    while (valueIndex < totalLen) {
        result[charIndex++] = value[valueIndex++] + '0';
    }
    result[charIndex] = '\0';
    return result;
}
10、单调栈背景知识
1、单调栈类别
1）、单调递增栈即栈内元数保持单调递增的栈，在插入新的元素到栈中时不要破坏栈中元素的单调性
2）、单调递减栈即栈内元素保持单调递减的栈，在插入新的元素到栈中时不要破坏栈中元素的单调性
// 单调递增栈，是栈中的元素是单调递增，最起码是单调不减的
for (int i = 0; i < T.size(); i++) {
    // 当前栈顶元素大于序列中的元素，则将栈顶元素出栈，直到栈顶中的元素<=序列中的元素或栈为空
    // 最后将序列元素入栈，成为新的栈顶,这样的栈为单调递增栈
    if (!stk.empty() && stk.top() > T[i]) {
        stk.pop();
    }
    stk.push(T[i]);
}

// 单调递减栈，即栈中的元素是单调递减，最起码是单调不增的
for (int i = 0; i < T.size(); i++) {
    // 当前栈顶元素小于或等于序列中的元素，则将栈顶元素出栈，直到栈顶中的元素>序列中的元素或栈为空
    // 最后将序列元素入栈，成为新的栈顶,这样的栈为单调递减栈
    if (!stk.empty() && stk.top() <= T[i]) {
        stk.pop();
    }
    stk.push(T[i]);
}
11、LeetCode-其他题目
1、【中等】692. 前K个高频单词
1:题目链接：https://leetcode.cn/problems/top-k-frequent-words/
2:题目描述:
3:代码实现
/**
 * Note: The returned array must be malloced, assume caller calls free().
 */
typedef struct node
{
    int num;
    char *value;
}Node, *pNode;
int cmp(const void *_a, const void *_b)
{
    pNode a = (pNode)_a;
    pNode b = (pNode)_b;
    
    if (a[0].num == b[0].num) {
        return strcmp(a[0].value, b[0].value);
    }
    return b[0].num - a[0].num;
}
int GetNumWords(char *temp, char **words, int wordsSize, int *uesd)
{
    int count = 0;
    for (int i = 0; i < wordsSize; i++) {
        if (strcmp(temp, words[i]) == 0) {
            count++;
            uesd[i] = 1;
        }
    }
    return count;
}
char ** topKFrequent(char ** words, int wordsSize, int k, int* returnSize){
    *returnSize = 0;
    if (words == NULL) {
        return words;
    }
    Node *map = (Node *)malloc(sizeof(Node) * wordsSize);
    int *uesd = (int *)malloc(sizeof(int) * wordsSize);
    int index = 0;
    for (int i = 0; i < wordsSize; i++) {
        if (uesd[i] == 1) {
            continue;
        }
        map[index].num = GetNumWords(words[i], words, wordsSize, uesd);
        map[index].value = (char *)malloc(sizeof(char) * (strlen(words[i]) + 1));
        strcpy(map[index].value, words[i]);
        index++; 
    }
    qsort(map, index, sizeof(Node), cmp);
    char **ret = (char **)malloc(sizeof(char *) * (wordsSize + 1));
    for (int j = 0; j < index; j++) {
        ret[j] = (char *)malloc(sizeof(char) * (strlen(map[j].value) + 1));
        strcpy(ret[j], map[j].value);
    }
    *returnSize = k;
    return ret;
}
2、【中等】17. 电话号码的字母组合
1:题目链接：https://leetcode.cn/problems/letter-combinations-of-a-phone-number/
2:题目描述：
3：代码实现
/**
 * Note: The returned array must be malloced, assume caller calls free().
 */
char *ARR[] = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
int NUM[] = {0, 0, 3, 3, 3, 3, 3, 4, 3, 4};
void VistArr(char *digits, int *temp, int index, int len, char **re, int *pathIdx)
{
    if (index == len) {
       for (int i = 0; i < len; i++) {
           re[*pathIdx][i] = ARR[digits[i] - '0'][temp[i]];
       }
       re[*pathIdx][len] = '\0';
       (*pathIdx)++;
       return;
    }
    for (temp[index] = 0; temp[index] < NUM[digits[index] - '0']; temp[index]++) {
        VistArr(digits, temp, index + 1, len, re, pathIdx);
    }
}
char ** letterCombinations(char * digits, int* returnSize){
    int len = strlen(digits);
    *returnSize = 0;
    if (len == 0) {
        return NULL;
    }
    int num = pow(4, len);
    char **re = (char **)malloc(sizeof(char *) * num);
    for (int i = 0; i < num; i++) {
        re[i] = (char *)malloc(sizeof(char) * (len + 1));
    }
    int temp[100] = { 0 };  // 记录数字对应字符串的第几个字符
    VistArr(digits, temp, 0, len, re, returnSize);
    return re;
}
3、【中等】649. Dota2 参议院
1：题目链接：https://leetcode.cn/problems/dota2-senate/
2：题目描述：
3：题解
算法思路：贪心 + 循环队列
1、以天辉的议员为例，当一名天辉方的议员行使权利时：
1）、如果目前所有的议员全是天辉方，那么该议员可以直接宣布天辉方取得胜利
2）、如果目前仍然有夜魔方的议员，那么这么天辉方的议员只能行使禁止一名参议员
的权利，应该贪心的挑选按照投票顺序的下一个夜魔方的议员
2、由于总要挑选投票顺序最早的议员，因此可以使用两个对列Radiat和Dire分布按照投票
顺序存在天辉和夜魔每一名议员的投票时间，模拟整个投票过程
1）、如果Radiat或者Dire为空，那么就可以宣布另一方获得胜利
2）、如果均不空，那么比较整个两个对列的首元素，就可以确定当前行使权利的是那一名
议员，如果Radiat的首元素较小，那说明轮到天辉方行使权利，其会挑选Dire的首元素
对应的那一名议员，因此，将Dire首元素永久的弹出，并将Radiat首元素弹出，增加N之后
重新放回队列，这里N是给定的字符串Senate的长度，即表示该议员参加下一轮的投票
4：代码实现
char* predictPartyVictory(char* senate) {
    int n = strlen(senate);
    int radiant[n], dire[n];
    int left_r = 0, right_r = 0;
    int left_d = 0, right_d = 0;
    for (int i = 0; i < n; ++i) {
        if (senate[i] == 'R') {
            radiant[right_r++] = i;
        } else {
            dire[right_d++] = i;
        }
    }
    while (left_r < right_r && left_d < right_d) {
        if (radiant[left_r] < dire[left_d]) {
            radiant[right_r++] = radiant[left_r] + n;
        } else {
            dire[right_d++] = dire[left_d] + n;
        }
        left_r++;
        left_d++;
    }
    int* ret;
    if (left_r < right_r) {
        ret = malloc(sizeof(char) * 8);
        ret = "Radiant";
    } else {
        ret = malloc(sizeof(char) * 5);
        ret = "Dire";
    }
    return ret;
}
4、【中等】781. 森林中的兔子
1：题目链接：https://leetcode.cn/problems/rabbits-in-forest/
2：题目描述：
3：题解：
算法思路：
1、同一个颜色的兔子回答的数值必然一样
2、回答同样数值的不一定就是同颜色的兔子
不妨设某种颜色的兔子M只，他们回答的答案数值为Cnt，显然两者之间的关系：M = Cnt + 1
但是如果在answer数值里回答Cnt的数量为t
1、t <= Cnt + 1,为了达到最少的兔子数量，可以假设t只为同一颜色，满足题意同时也不会导致额外兔子数量增加
2、t >= Cnt + 1,回答Cnt的兔子应该有Cnt + 1只兔子，这时说明有数量相同但颜色不同的兔子进行了回答，为了达到
最少的兔子数量，我们应该将t分为若干颜色，并尽可能让某一种颜色的兔子为Cnt + 1
总之:我们应该让同一颜色的兔子数量尽量多，从而实现总的兔子的数量最少
4：代码实现
int cmFunc(const void *a, const void *b)
{
    return (*(int *)a) - (*(int *)b);
}
int numRabbits(int* answers, int answersSize){
    if (answersSize == 0) {
        return answersSize;
    }
    qsort(answers, answersSize, sizeof(int), cmFunc);
    
    int counts[1000] = { 0 };
    int ret = 0;
    for (int i = 0; i < answersSize; i++) {
        if (counts[answers[i]] > 0) {
            counts[answers[i]]--;
        } else {
            ret += answers[i] + 1;
            counts[answers[i]] = answers[i];
        }
    }
    return ret;
}
5、N!
1、题目链接：http://oj.rnd.huawei.com/problems/19/details
2、题目描述：
3、题目分析：
这个N的取值较大时会超出单一数值类型的范围，需要使用数组来实现大数乘法
4、代码实现
int Multiply(int res, int size, int x)
{
    int carry = 0;
    for (int i = 0; i < size; i++) {
        int prod = res[i] x + carry;
        res[i] = prod % 10;
        carry = prod / 10;
    }
    while (carry) {
        res[size] = carry % 10;
        carry /= 10;
        (size)++;
    }
}
void Factorial(int n)
{
    int res[100000] = { 1 };
    int size = 1;
    for (int i = 2; i <= n; i++) {
        Multiply(res, &size, i);
    }
    for (int i = size - 1; i >= 0; i--) {
        printf("%d", res[i]);
    }
}

int main()
{
    int n;
    scanf("%d", &n);
    Factorial(n);
    return 0;
}
素数
1、题目链接：http://oj.rnd.huawei.com/problems/22/details
2、题目描述：
3、代码实现
int eratosthenesSieve(unsigned long long int N) {
  // prime numbers are positive, better to use largest unsiged integer
  unsigned long long int i, j, total; // total: number of prime numbers < N
  _Bool *a = malloc(sizeof(_Bool) * N);

  if (a == NULL) {
    printf("No enough memory.\n");
    return -1;
  }
  
  /* a[i] equals 1: i is prime number.
     a[i] equals 0: i is not prime number.
     From beginning, set i as prime number. Later filter out non-prime numbers
  */
  for (i = 2; i < N; i++) {
    a[i] = 1; 
  }

  // mark multiples(<N) of i as non-prime numbers
  for (i = 2; i < N; i++) {
    if (a[i]) { // a[i] is prime number at this point
      for (j = i; j < (N / i) + 1; j++) {
	/* mark all multiple of 2 * 2, 2 * 3, as non-prime numbers;
	   do the same for 3,4,5,... 2*3 is filter out when i is 2
	   so when i is 3, we only start at 3 * 3
	*/
	a[i * j] = 0;
      }
    }
  }

  // count total. print prime numbers < N if needed.
  total = 0;
  for (i = 2; i < N; i++) {
    if (a[i]) { // i is prime number
      total += 1;
    }
  }

  return total;
}

int main() {
  unsigned long long int a1 = 0, N;
  scanf("%llu", &N);
  a1 = eratosthenesSieve(N); // print the prime numbers
  printf("%llu\n", a1);
  

  
  return 0;
}

4、算法思想
1、原文链接：https://zhuanlan.zhihu.com/p/151432852
质数（Prime number）又称素数，指在大于1的自然数中，除了1和该数自身外，无法被其他自然数整除的数（也可定义为只有1与该数本身两个正因数的数）
示例：
求出所有不超过100的素数：
解： 因为小于等于10的所有素数为2、3、5、7，所以依次删除2、3、5、7的倍数。
初始列表
[ 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99，100]
2的倍数：（从22=4开始去掉）
[4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36, 38, 40, 42, 44, 46, 48, 50, 52, 54, 56, 58, 60, 62, 64, 66, 68, 70, 72, 74, 76, 78, 80, 82, 84, 86, 88, 90, 92, 94, 96, 98, 100]
3的倍数：（从33=9开始去掉）
[9, 12, 15, 18, 21, 24, 27, 30, 33, 36, 39, 42, 45, 48, 51, 54, 57, 60, 63, 66, 69, 72, 75, 78, 81, 84, 87, 90, 93, 96, 99]
5的倍数：（从55=25开始去掉）
[25, 30, 35, 40, 45, 50, 55, 60, 65, 70, 75, 80, 85, 90, 95, 100]
7的倍数：（从77=49开始去掉）
[49, 56, 63, 70, 77, 84, 91, 98]
剩下的就是100以内的素数了。
[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
12、背包问题
1、0-1背包问题
 定义：给定 n 种物品和一个容量为 c 的背包，物品 i 的重量是 wi，其价值为 vi
问：应如何选择装入背包的物品，使的装入背包中的物品的总价值最大？每个物品只使用一次
【示例】
【背包容量：物品种类】=【c：n】=>【10:4】
【物品种类，价值】=【w1，v1】=> {【2,1】、【3,3】、【4,5】、【7,9】}
d[i][j] = value i : 代表某个物品；j：代表背包当前容量；value：代表背包容量为j时此时背包内物品的价值
行代表：某个物品 ； 列代表：当前容量

IF j < w[i] THEN
     物品重量w[i]大于当前背包容量j，不足以放下当前物品，只能不往背包存放，此时背包的价值是上一个物品的价值
     dp[i][j] = dp[i-1][j]
ELSE
     物品重量w[i]小于等于当前背包容量j，能放下当前物品，此时存在拿或者不拿当前物品的选择
      IF 不拿当前物品 THEN
                 此时背包内的物品价值和存放上一个物品时背包内的物品价值一样
                  dp[i][j] = dp[i-1][j]
       ELSE
               此时背包内的物品价值等于当前物品的价值v[i]，和当前背包容量减去当前物品重量之后剩余的容量的价值之和
              dp[i][j] = dp[i-1][j-w[i]] + v[i]
     END IF
END IF

IF j < w[i] THEN
    dp[i][j] = dp[i-1][j]
ELSE
    dp[i][j] = MAX(不拿，拿） = MAX(dp[i-1][j], dp[i-1][j - w[i]] + v[i])
END IF


/**
    0-1 背包 原始版
    capacity:背包容量
    n：物品种类/数量
    weigh[i]: 物品 i 的容量或重量
    value[i]: 物品 i 本身的价值
    dp[i][j]; i：代表某个物品 j: 代表背包当前容量
**/
int dp[100][100];
int Beibao_0_1_Origin(int capacity, int n, int weigh,int value)
{
    int i, j;
    for (i = 1; i <= n; i++) {
        for (j = 1; j <= capacity; j++) {
            if (j < weigh[i -1]) {
                /*物品 i 的容量 大于 背包当前容量，不足以放下这个物品，此时背包呈现的价值为上一个(i-1)物品的价值*/
                dp[i][j] = dp[i -1][j];
            } else {
                /* max(参数1：不拿i物品其价值为上个物品（i-1) 的价值，
                参数2：拿i物品其价值为当前i物品本身价值 + 当前背包容量减去 i 物品容量之后剩余背包容量所呈现的价值之和
                */
            dp[i][j] = fmax(dp[i-1][j], dp[i-1][j - weigh[i -1]] + value[i -1]);
        }
    }
}
/* 输出DP数组 */
for (i = 0; i <= n; i++) {
    for (j = 0; j <= capacity; j++) {
        printf(" %d\t", dp[i][j]);
    }
    printf("\n");
}

    /*  返回结果值 */
    return dp[n][capacity];
}

int main()
{
    int w[] = {2,3,4,7};
    int v[] = {1,3,5,9};
    Beibao_0_1_Origin(10,4,w,v);
    return 0;
}

/**
    0-1 背包 原始版
    capacity:背包容量
    n：物品种类/数量
    weigh[i]: 物品 i 的容量或重量
    value[i]: 物品 i 本身的价值
    dp[i][j]; i：代表某个物品 j: 代表背包当前容量
**/
int dp[100][100];
int Beibao_0_1_Origin(int capacity, int n, int weigh,int value){
    int i, j;
    for (i = 1; i <= n; i++) {
        for (j = 1; j <= capacity; j++) {
            if (j < weigh[i -1]) {
                /*
                物品 i 的容量 大于 背包当前容量，不足以放下这个物品，此时背包呈现的价值为上一个(i-1)物品的价值
                */
                dp[i][j] = dp[i -1][j];
            } else {
                /* max(参数1：不拿i物品其价值为上个物品（i-1) 的价值，
                参数2：拿i物品其价值为当前i物品本身价值 + 当前背包容量减去 i 物品容量之后剩余背包容量所呈现的价值之和
                */
                dp[i][j] = fmax(dp[i-1][j], dp[i-1][j - weigh[i -1]] + value[i -1]);
            }
        }
    }

    /* 输出DP数组 */
    for (i = 0; i <= n; i++) {
        for (j = 0; j <= capacity; j++) {
        printf(" %d\t", dp[i][j]);
        }
        printf("\n");
    }

    /* 返回结果值 */
    return dp[n][capacity];
}

int main() {
    int w[] = {2,3,4,7};

    int v[] = {1,3,5,9};

    Beibao_0_1_Origin(10,4,w,v);

    return 0;
}

DP存在后无效原则：即当前状态只与上一个状态有关，与其他状态无关，优化下DP的占用空间，缩小空间使用滚动数组，每次迭代前都更新之前状态的数组
状态转移方程
IF j >= w[i]
    dp[j] = MAX(dp[j], dp[j-w[i]] + v[j])
END IF

/**
    0-1 背包 升级版
    capacity:背包容量
    n：物品种类/数量
    weigh[i]: 物品 i 的容量或重量
    value[i]: 物品 i 本身的价值
    dp[j]; j: 代表背包当前容量/ value: 代表当背包容量为j时的价值
**/
int dp2[100];
int Beibao_0_1_RollArr(int capacity, int n, int weigh,int value) {
    for (int i = 1; i <= n; i++) {
        // 两种实现方式任选其一
        // 实现方式1： 逆向递推
        for (int j = capacity; j >= weigh[i -1]; j--) {
            dp2[j] = fmax(dp2[j], dp2[j - weigh[i -1]] + value[i -1]);
        }
        // 实现方式2：正向递推，从能装下 i 物品的背包容量 j 开始
        for (int j = weigh[i -1]; j <= capacity; j++) {
            dp2[j] = fmax(dp2[j], dp2[j - weigh[i -1]] +value[i -1]);
        }
    }
    /* 输出滚动数组 */
    for (int t = 0; t <= capacity; t++) {
        printf("  %d\t", dp2[t]);
    }
    printf("\n");

    return dp2[capacity];
}

int main()
{
    int w[] = {2,3,4,7};
    int v[] = {1,3,5,9};
    Beibao_0_1_RollArr(10,4,w,v);
    return 0;
}

2、完全背包
 
定义：给定 n 种物品和一个容量为 c 的背包，物品 i 的重量是 wi，其价值为 vi问：应如何选择装入背包的物品，使的装入背包中的物品的总价值最大？每个物品使用次数不限
【完全背包-VS-0/1背包】
完全背包：提供的N种物品中某个物品只有拿和不拿但可拿的数量>=1
0/1背包：提供的N种物品中某个物品只有拿和不拿且只拿一次
【示例】
已0/1背包中的示例为基础，将某个物品使用的次数有【0,1】次扩展为【0，j / w[i] 】；j：背包容量，w[i]：物品i的重量
/**
    完全背包，在0-1背包的基础上修正
    capacity:背包容量
    n：物品种类/数量
    weigh[i]: 物品 i 的容量或重量
    value[i]: 物品 i 本身的价值
    dp[j]; j: 代表背包当前容量/ value: 代表当背包容量为j时的价值
**/
int dp2[100];
int Beibao_0_1_All(int capacity, int n, int weigh,int value)
{
    for (int i = 1; i <= n; i++) {
        for (int j = capacity; j >= weigh[i -1]; j--) {
            /* 将物品 i 从【0，j / w[i]】遍历求最大 */
            for (int k = 0; k <= (j / weigh[i -1]); k++) {
                dp2[j] = fmax(dp2[j], dp2[j - k weigh[i -1]] + k * value[i -1]);
            }
        }

    /* 输出滚动数组 */
    for (int t = 0; t <= capacity; t++) {
        printf(" %d\t", dp2[t]);
    }
        printf("\n");
    }
    return dp2[capacity];
}
int main()
{
    int w[] = {2,3,4,7};
    int v[] = {1,3,5,9};
    Beibao_0_1_All(10,4,w,v);
    return 0;
}



 3、多重背包
 
定义：有N种物品和一个容量为V的背包，第 i 种物品最多有 n[i] 件可用，每件费用是 c[i]，价值是w[i]。
问：将那些物品装入背包可使这些物品的费用总和不超过背包容量，且价值总和最大
【多重背包-VS-完全背包-VS-0/1背包】
多重背包：提供的N中物品，每个物品有多个，某个物品只有拿或不拿但拿的数量>=1
完全背包：提供的N种物品中某个物品只有拿和不拿但可拿的数量>=1
0/1背包：提供的N种物品中某个物品只有拿和不拿且只拿一次
【示例】
/**
多重背包，在0-1背包的基础上修正
n：代表希望购买的奖品的种数
m：代表拨款金额
v[i]、w[i]、s[i]：代表第 i 种奖品的价格、价值、能购买的最大数量(0-s[i]件均可）
**/
int dp2[6100];
int Beibao_0_1_DuoChong(int n, int m, int v, int w, int s)
{
    for (int i = 1; i <= n; i++) {
        for (int j = m; j >= v[i -1]; j--) {
            /* 将物品 i 从【0，s[i]】遍历求最大 */
            for (int k = 0; k <= s[i-1] && j >= k v[i -1]; k++) {
                dp2[j] = fmax(dp2[j], dp2[j - k * v[i -1]] + k * w[i -1]);
            }
        }
    }
    return dp2[m];
}
int main()
{
    int v[] = {80, 40, 30, 40, 20};
    int w[] = {20, 50, 50, 30, 20};
    int s[] = {4, 9, 7, 6, 1};
    Beibao_0_1_DuoChong(5,1000,v,w,s);
    return 0;
}
 4、二维费用背包问题
 
有N件物品和一个容量是V的背包（条件1），背包能承受的最大重量是M（条件2）
每件物品只能用一次，体积是ai，重量是bi，价值是wi
求解将那些物品装入背包，可使物品总体积不超过背包容量，总重量不超过背包可承受的最大重量（必须同时满足两
13、C语言基础
1、【C】【指针用法】【指针运算】
指针用法
int PointerOp()
{
    /* C 指针操作  C Primer Plus 第六版 10.5章节 */
    int urn[5] = {100, 200, 300, 400, 500};
    int *ptr1, *ptr2, *ptr3;
    ptr1 = urn; 
    ptr2 = &urn[2];
    printf("ptr1 = %p, ptr1 + 1 = %p,  ptr1 + 2 = %p, ptr1 + 3 = %p, ptr1 + 4 = %p\n", 
           ptr1, ptr1 + 1, ptr1 + 2, ptr1 + 3, ptr1 + 4);
    printf("pointer value \n");
    /* 含义解释
       ptr1  = ptr1指针指向的地址
       *ptr1 = ptr1指针指向的地址的值
       &ptr1 =  ptr1指针自身在内存中的地址
    */
    printf("ptr1 = %p, *ptr1 = %d, &ptr1 = %p\n", ptr1, *ptr1, &ptr1);
    printf("pointer adding value \n");
    ptr3 = ptr1 + 4; // 与 ptr3 指针指向 urn[4] 所在的地址
    printf("ptr1 + 4 = %p, *(ptr1 + 4) = %d \n", ptr1 + 4, *(ptr1 + 4));
    
    ptr1++; // 递增指针,当前指针指向 urn[2]
    printf("after pointer ptr1++ value \n");
    printf("ptr1 = %p, *ptr1 = %d, &ptr1 = %p \n", ptr1, *ptr1, &ptr1);
    ptr2--; // 递减指针，当前指针指向
    printf("after pointer ptr2-- value \n");
    printf("ptr2 = %p, *ptr2 = %d, &ptr2 = %p \n", ptr2, *ptr2, &ptr2);
    ptr1--; // 恢复初始值
    ptr2++; // 恢复初始值
    printf("reset pointer to original value \n");
    printf("ptr1 = %p, ptr2 = %p \n", ptr1, ptr2);
    /* 两个指针相减 */
    printf("subtractiong one pointer from another \n");
    printf("ptr2 = %p, ptr1 = %p, ptr2 - ptr1 = %td \n", ptr2, ptr1, ptr2 - ptr1);
    /* 指针减去一个整数 */
    printf("subtracting an int from a pointer \n");
    printf("ptr3 = %p, ptr3 - 2 = %p \n", ptr3, ptr3 - 2);
    getchar();
    return 0;
/*
ptr1     = 000000000065FE00,
ptr1 + 1 = 000000000065FE04,  
ptr1 + 2 = 000000000065FE08,
ptr1 + 3 = 000000000065FE0C, 
ptr1 + 4 = 000000000065FE10
pointer value
ptr1 = 000000000065FE00, *ptr1 = 100, &ptr1 = 000000000065FDF8
pointer adding value
ptr1 + 4 = 000000000065FE10, *(ptr1 + 4) = 500
after pointer ptr1++ value
ptr1 = 000000000065FE04, *ptr1 = 200, &ptr1 = 000000000065FDF8
after pointer ptr2-- value
ptr2 = 000000000065FE04, *ptr2 = 200, &ptr2 = 000000000065FDF0
reset pointer to original value
ptr1 = 000000000065FE00, ptr2 = 000000000065FE08
subtractiong one pointer from another
ptr2 = 000000000065FE08, ptr1 = 000000000065FE00, ptr2 - ptr1 = 2
subtracting an int from a pointer
ptr3 = 000000000065FE10, ptr3 - 2 = 000000000065FE08
*/
}



int main()
{
    int zippo[4][2] = {{2,4},{6,8},{1,3},{5,7}};
    printf("zippo = %p, zippo + 1 = %p, zippo + 2 = %p, zippo + 3 = %p \n",
            zippo, zippo + 1, zippo + 2, zippo + 3);
    
    printf("zipp[0] = %p, zippo[0] + 1 = %p\n", zippo[0], zippo[0] + 1);
    printf("*zippo = %p, *zippo + 1 = %p, *zippo + 2 = %p, *zippo + 3 = %p, *zippo + 4 = %p, *zippo + 5 = %p, *zippo + 6 = %p, *zippo + 7 = %p\n",
            *zippo, *zippo + 1, *zippo + 2, *zippo + 3, *zippo + 4, *zippo + 5, *zippo + 6, *zippo + 7);
    printf("zippo[0][0] = %d, *zippo[0] = %d, **zippo = %d\n", zippo[0][0], *zippo[0], **zippo);
    printf("zippo[2][1] = %d, *(*(zippo + 2) + 1) = %d\n", zippo[2][1], *(*(zippo +2 ) + 1));
    getchar();
    return 0; 
/*
zippo     = 000000000065FDF0, zippo + 1 = 000000000065FDF8, 
zippo + 2 = 000000000065FE00, zippo + 3 = 000000000065FE08
zipp[0] = 000000000065FDF0, zippo[0] + 1 = 000000000065FDF4
*zippo     = 000000000065FDF0, *zippo + 1 = 000000000065FDF4, 
*zippo + 2 = 000000000065FDF8, *zippo + 3 = 000000000065FDFC, 
*zippo + 4 = 000000000065FE00, *zippo + 5 = 000000000065FE04, 
*zippo + 6 = 000000000065FE08, *zippo + 7 = 000000000065FE0C
zippo[0][0] = 2, *zippo[0] = 2, **zippo = 2
zippo[2][1] = 3, *(*(zippo + 2) + 1) = 3
*/  
}
	
	
	
int main()
{
    int zippo[4][2] = {{2,4},{6,8},{1,3},{5,7}};
    printf("*zippo = %p, *zippo + 1 = %p, *zippo + 2 = %p, *zippo + 3 = %p, *zippo + 4 = %p, *zippo + 5 = %p, *zippo + 6 = %p, *zippo + 7 = %p\n",
        *zippo, *zippo + 1, *zippo + 2, *zippo + 3, *zippo + 4, *zippo + 5, *zippo + 6, *zippo + 7);
    int (*pz)[2]; // pz指向一个内涵两个int类型值的数组
    pz = zippo;
    printf("pz = %p, pz + 1 = %p\n", pz, pz + 1);
    printf("pz[0] = %p, pz[0] + 1 = %p\n", pz[0], pz[0] + 1);
    printf("*pz = %p, *pz + 1= %p\n", *pz, *pz + 1);
    printf("pz[0][0] = %d\n", pz[0][0]);
    getchar();
    return 0; 
/*
*zippo     = 000000000065FDE0, *zippo + 1 = 000000000065FDE4, 
*zippo + 2 = 000000000065FDE8, *zippo + 3 = 000000000065FDEC, 
*zippo + 4 = 000000000065FDF0, *zippo + 5 = 000000000065FDF4, 
*zippo + 6 = 000000000065FDF8, *zippo + 7 = 000000000065FDFC
pz = 000000000065FDE0, pz + 1 = 000000000065FDE8
pz[0] = 000000000065FDE0, pz[0] + 1 = 000000000065FDE4
*pz = 000000000065FDE0, *pz + 1= 000000000065FDE4
pz[0][0] = 2
*/
}	
	
	
	2、【C】【CONST用法】
C CONST用法
int main()
{
    double rates[5] = {88.99, 1001.12, 59.45, 183.11, 340.5};
    /* 该指针可以修改它所指向的地址，但不能修改它所指向的值 */
    const double *ptr1 = rates;
    ptr1 = &rates[3];
    *ptr1 = 99.88; // 不允许 Read-only variable is not assignable
    /* 该指针可以修改它所指向的值，但不能修改它所指向的地址 */
    double *const ptr2 = rates;
    ptr2 = &rates[3]; // 不允许 Cannot assign to variable 'ptr2' with const-qualified type 'double *const'
    *ptr2 = 99.88;
    
    /* 该指针即不能更改它所指向的地址，也不能修改指向地址上的值 */
    const double * const ptr3 = rates;
    ptr3 = &rates[3]; // 不允许 assignment of read-only variable 'ptr3'
    *ptr3 = 99.88; // 不允许 Cannot assign to variable 'ptr3' with const-qualified type 'const double *const'
    getchar();
    return 0;
}
CONST TYPE   *VarName; 该指针可以修改它所指向的地址，但不能修改它所指向的值
TYPE  *CONST  VarName; 该指针可以修改它所指向的值，但不能修改它所指向的地址
CONST TYPE * CONST VarName;该指针即不能更改它所指向的地址，也不能修改指向地址上的值
3、【C】#define、typedef、const
	<table border="1"><tr><td></td><td>typedef</td><td>define</td><td>define</td><td>const</td></tr><tr><td>原理</td><td>是C语言的关键字，他在编译时处理，所以typedef有类型检查的功能</td><td>是C语言中定义的语法，它是预处理指令，在预处理时进行简单而机械的字符串替换，不做正确性检查，不管含义是否正确照样代入，只有在编译已被展开的源程序时才会返现可能的错误并报错</td><td>只是用来进行单纯的文本替换，define常量的声明周期止于编译期，不分配内存空间，它处于程序的代码段，实际成中它只是一个常数，一个命中的参数并没有实际存在</td><td>存在与程序的数据段，并在堆栈中分配空间，const常量在程序中确实存在，并且可以被调用、传递</td></tr><tr><td>功能</td><td>用来定义类型的别名，这些类型不知包含内部类型(intchar等)，还包括自定义类型(struct\enum)</td><td>可以为类型取别名，还可定义变量、常量、编译开关等</td><td>define没有数据类型</td><td>有数据类型，编译期可以进行类型检查如类型、语句结构等</td></tr><tr><td>作用域</td><td>有自己的作用域</td><td>没有作用域的限制，只要之前预定义过宏，以后程序都可以使用</td><td>IDE不支撑define调试</td><td>IDE支撑const调试</td></tr></table>

