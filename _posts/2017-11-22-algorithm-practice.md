---
layout:     post
title:      "经典面试算法题"
subtitle:   ""
date:       2017-11-22 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - 面试
    - 算法
---

### 前言. 
经历求职记，面试各大互联网公司，总结一下常考的算法笔试面试题，方便以后的学习与使用。
     
### 1.最大子序列和
这里把最大子序列和放在第一个位置，它并不是字符串相关的问题，事实上它的目的是要找出由数组成的一维数组中和最大的连续子序列。比如[0，-2，3，5，-1，2]应返回9，[-9，-2，-3，-5，-3]应返回-2。
#### 方法一:动态规划法
你也许从这两个例子中已经可以看出，使用动态规划的方法很容易完成这个任务，只要前i项的和还没有小于0那么子序列就一直向后扩展，否则丢弃之前的子序列开始新的子序列，同时我们要记下各个子序列的和，最后找到和最大的子序列。但是你可能需要谨慎一些，在整个数组都为负的情况下，所以初始的和最大值赋值不当的话可能会出问题。  

根据以上的思路我们可以有以下的代码：  
    
```C
********************************************************** 
动态规划求最大子序列和 
**********************************************************/  
int Maxsum(int * arr, int size)  
{  
    int maxSum = -INF; //很重要，初始值赋值为负无穷大  
    int sum = 0;  
    for(int i = 0; i < size; ++i)  
{  
        //小于0则舍弃  
        if(sum < 0)  
        {  
            sum = arr[i];  
        }else  
        {  
            sum += arr[i];  
        }  
        //比现有最大值大，则替换  
        if(sum > maxSum)  
        {  
            maxSum = sum;  
        }  
    }  
    return maxSum;  
}  

```
如果想获得最大子序列和的初始和结束位置怎么办呢？我们知道，每当当前子数组和的小于0时，便是新一轮子数组的开始，每当更新最大和时，便对应可能的结束下标，这个时候，只要顺便用本轮的起始和结束位置更新始末位置就可以，程序结束，最大子数组和以及其始末位置便一起被记录下来了  
  
```C 
*****************************************************************************/  
void Maxsum_location(int * arr, int size, int & start, int & end)  
{  
    int maxSum = -INF;  
    int sum = 0;  
   /* curstart记录每次当前起始位置 */  
    int curstart = start = 0;    
    for(int i = 0; i < size; ++i)  
    {  
        if(sum < 0)  
        {  
            sum = arr[i];  
            curstart = i;     /* 记录当前的起始位置 */  
        }else  
        {  
            sum += arr[i];  
        }  
        if(sum > maxSum)  
        {  
            maxSum = sum;  
            start = curstart; /* 记录并更新最大子数组起始位置 */  
            end = i;  
        }  
    }  
}  
```

#### 方法二:分治法
其实数组的问题，最好留点心，有一大部分题目是可以用分治的办法完成的，比如说这道题里面：最大子序列和可能出现在三个地方，1）整个出现在输入数据的左半部分，2）整个出现在输入数据的右半部分，3）或者跨越输入数据的中部从而占据左右两个半部分。可以有以下代码：  

```
/************************************************************** 
分治法求解最大子序列和 
***************************************************************/  
int MaxSumRec( const vector<int> & a, int left, int right )  
{  
    if( left == right )  // Base case  
        if( a[ left ] > 0 )  
            return a[ left ];  
        else  
            return 0;  
    int center = ( left + right ) / 2;  
    int maxLeftSum  = maxSumRec( a, left, center );  
    int maxRightSum = maxSumRec( a, center + 1, right );  
    int maxLeftBorderSum = 0, leftBorderSum = 0;  
    for( int i = center; i >= left; i-- )  
    {  
        leftBorderSum += a[ i ];  
        if( leftBorderSum > maxLeftBorderSum )  
            maxLeftBorderSum = leftBorderSum;  
    }  
    int maxRightBorderSum = 0, rightBorderSum = 0;  
    for( int j = center + 1; j <= right; j++ )  
    {  
        rightBorderSum += a[ j ];  
        if( rightBorderSum > maxRightBorderSum )  
            maxRightBorderSum = rightBorderSum;  
    }  
    return max3( maxLeftSum, maxRightSum, maxLeftBorderSum + maxRightBorderSum );  
}  

```

### 2.最长递增子序列
和上一问题一样，这是数组序列中的问题，比如arr={1,5,8,2,3,4}的最长递增子序列是1,2,3,4
#### 方法一:动态规划法
    结合上一题的思路，在数组的这类问题里面使用动态规划还是很常见的，从后向前分析，很容易想到，第i个元素之前的最长递增子序列的长度要么是1（比如说递减的数列），要么就是第i-1个元素之前的最长递增子序列加1，我们可以得到以下关系：
LIS[i] = max{1,LIS[k]+1}，其中，对于任意的k<=i-1，arr[i] > arr[k]，这样arr[i]才能在arr[k]的基础上构成一个新的递增子序列。这种方法代码如下：  

```C
#include <iostream>  
using namespace std;  
   
//动态规划法求最长递增子序列 LIS  
   
int dp[101]; /* 设数组长度不超过100，dp[i]记录到[0,i]数组的LIS */  
int lis;    /* LIS 长度 */  
   
int LIS(int * arr, int size)  
{  
    for(int i = 0; i < size; ++i)  
    {  
        dp[i] = 1;  
        for(int j = 0; j < i; ++j)  
        {  
            if(arr[i] > arr[j] && dp[i] < dp[j] + 1)  
            {  
                dp[i] = dp[j] + 1;  
                if(dp[i] > lis)  
                {  
                    lis = dp[i];  
                }  
            }  
        }  
    }  
    return lis;  
}  
   
/* 输出LIS */  
void outputLIS(int * arr, int index)  
{  
    bool isLIS = 0;  
    if(index < 0 || lis == 0)  
    {  
        return;  
    }  
    if(dp[index] == lis)  
    {  
        --lis;  
        isLIS = 1;  
    }  
   
    outputLIS(arr,--index);  
   
    if(isLIS)  
    {  
        printf("%d ",arr[index+1]);  
    }  
}  
   
void main()  
{  
    int arr[] = {1,-1,2,-3,4,-5,6,-7};  
   
    /* 输出LIS长度； sizeof 计算数组长度 */  
    printf("%d\n",LIS(arr,sizeof(arr)/sizeof(int)));  
   
    /* 输出LIS */  
    outputLIS(arr,sizeof(arr)/sizeof(int) - 1);  
    printf("\n");  
}  

```
#### 方法二：数组排序后，与原数组求最长公共子序列
这个方法还是非常巧妙的，因为LIS是单调递增的性质，所以任意一个LIS一定跟排序后的序列有最长公共子序列，并且就是LIS本身。不过这里还没有提到最长公共子序列，可以先移步下一节，看完后再回来看这个方法的代码，代码如下：  

```C
#include <iostream>  
using namespace std;  
   
/* 最长递增子序列 LIS 
 * 设数组长度不超过 100 
 * quicksort + LCS 
*/  
   
void swap(int * arr, int i, int j)  
{  
    int tmp = arr[i];  
    arr[i] = arr[j];  
    arr[j] = tmp;  
}  
   
void qsort(int * arr, int left, int right)  
{  
    if(left >= right)    return ;  
    int index = left;  
    for(int i = left+1; i <= right; ++i)  
    {  
        if(arr[i] < arr[left])  
        {  
            swap(arr,++index,i);  
        }  
    }  
    swap(arr,index,left);  
    qsort(arr,left,index-1);  
    qsort(arr,index+1,right);  
}  
   
int dp[101][101];  
   
int LCS(int * arr, int * arrcopy, int len)  
{  
    for(int i = 1; i <= len; ++i)  
    {  
        for(int j = 1; j <= len; ++j)  
        {  
            if(arr[i-1] == arrcopy[j-1])  
            {  
                dp[i][j] = dp[i-1][j-1] + 1;  
            }else if(dp[i-1][j] > dp[i][j-1])  
            {  
                dp[i][j] = dp[i-1][j];  
            }else  
            {  
                dp[i][j] = dp[i][j-1];  
            }  
        }  
    }  
    return dp[len][len];  
}  
   
void main()  
{  
    int arr[] = {1,-1,2,-3,4,-5,6,-7};  
    int arrcopy [sizeof(arr)/sizeof(int)];  
   
    memcpy(arrcopy,arr,sizeof(arr));  
    qsort(arrcopy,0,sizeof(arr)/sizeof(int)-1);  
   
    /* 计算LCS，即LIS长度 */  
    int len = sizeof(arr)/sizeof(int);  
    printf("%d\n",LCS(arr,arrcopy,len));  
}  
```

#### 方法三:动态规划和二分查找结合
我们期望在前i个元素中的所有长度为len的递增子序列中找到这样一个序列，它的最大元素比arr[i+1]小，而且长度要尽量的长，如此，我们只需记录len长度的递增子序列中最大元素的最小值就能使得将来的递增子序列尽量地长。在这里我们维护一个数组MaxV[i]，记录长度为i的递增子序列中最大元素的最小值，并对于数组中的每个元素考察其是哪个子序列的最大元素，二分更新MaxV数组，最终i的值便是最长递增子序列的长度。这个方法真是太巧妙了，妙不可言。
具体代码如下：  

```C
#include <iostream>  
using namespace std;  
   
/* 最长递增子序列 LIS 
 * 设数组长度不超过 30 
 * DP + BinarySearch 
*/  
   
int MaxV[30]; /* 存储长度i+1（len）的子序列最大元素的最小值 */  
int len;      /* 存储子序列的最大长度 即MaxV当前的下标*/  
   
/* 返回MaxV[i]中刚刚大于x的那个元素的下标 */  
int BinSearch(int * MaxV, int size, int x)  
{  
    int left = 0, right = size-1;  
    while(left <= right)  
    {  
        int mid = (left + right) / 2;  
        if(MaxV[mid] <= x)  
        {  
            left = mid + 1;  
        }else  
        {  
            right = mid - 1;  
        }  
    }  
    return left;  
}  
   
int LIS(int * arr, int size)  
{  
    MaxV[0] = arr[0]; /* 初始化 */  
    len = 1;  
    for(int i = 1; i < size; ++i) /* 寻找arr[i]属于哪个长度LIS的最大元素 */  
    {  
        if(arr[i] > MaxV[len-1]) /* 大于最大的自然无需查找，否则二分查其位置 */  
        {  
            MaxV[len++] = arr[i];  
        }else  
        {  
            int pos = BinSearch(MaxV,len,arr[i]);  
            MaxV[pos] = arr[i];  
        }  
    }  
    return len;  
}  
   
void main()  
{  
    int arr[] = {1,-1,2,-3,4,-5,6,-7};  
   
    /* 计算LIS长度 */  
    printf("%d\n",LIS(arr,sizeof(arr)/sizeof(int)));  
}  
```

### 3.最长公共子串（LCS）
回到最常见的字符串问题了，这里的找两个字符串的最长公共子串，要求在原字符串中是连续的。其实和上面两个问题一样，这里依旧可以用动态规划来求解，其实博主自己也不大擅长动态规划，但是可以仿照上面的思路来操作。我们采用一个二维矩阵来记录中间的结果。
     这样矩阵中的最大元素就是 最长公共子串的长度。
     在构造这个二维矩阵的过程中由于得出矩阵的某一行后其上一行就没用了，所以实际上在程序中可以用一维数组来代替这个矩阵(这样空间复杂度就降低了哈)。
代码如下：  

```C
#include<iostream>  
#include<cstring>  
#include<vector>  
using namespace std;  
//str1为横向，str2这纵向  
const string LCS(const string& str1,const string& str2){  
    int xlen=str1.size();       //横向长度  
    vector<int> tmp(xlen);        //保存矩阵的上一行  
    vector<int> arr(tmp);     //当前行  
    int ylen=str2.size();       //纵向长度  
    int maxele=0;               //矩阵元素中的最大值  
    int pos=0;                  //矩阵元素最大值出现在第几列  
    for(int i=0;i<ylen;i++){  
        string s=str2.substr(i,1);  
        arr.assign(xlen,0);     //数组清0  
        for(int j=0;j<xlen;j++){  
            if(str1.compare(j,1,s)==0){  
                if(j==0)  
                    arr[j]=1;  
                else  
                    arr[j]=tmp[j-1]+1;  
                if(arr[j]>maxele){  
                    maxele=arr[j];  
                    pos=j;  
                }  
            }         
        }  
        tmp.assign(arr.begin(),arr.end());  
    }  
    string res=str1.substr(pos-maxele+1,maxele);  
    return res;  
}  
int main(){  
    string str1("21232523311324");  
    string str2("312123223445");  
    string lcs=LCS(str1,str2);  
    cout<<lcs<<endl;  
    return 0;  
}  
```  
 
### 4.最长公共子序列
这才是笔试面试中出现频度最高的问题，前面提到了一个最长公共子串，这里的最长公共子序列与它的区别在于最长公共子序列不要求在原字符串中是连续的，比如ADEFG和ABCDEG的最长公共子序列是ADEG。
#### 方法一:递归方法求解
这个地方可能最容易想到的方法就是递归处理了，设有字符串a[0...n]，b[0...m]，则易知当数组a的i位置上和b的j位置上对应位相同时，则直接求解两个串从下一个位置开始的剩下部分的最长公共子序列即可；当不同时，则求a[i+1...n]、b[j...m]和a[i...n]、b[j+1...m]两种情况中的较大数值即可，用公式表示如下：

代码如下：  

```
#include<stdio.h>  
#include<string.h>  
char a[100],b[100];  
int lena,lenb;  
int LCS(int,int);///两个参数分别表示数组a的下标和数组b的下标  
int main()  
{  
    strcpy(a,"ABCBDAB");  
    strcpy(b,"BDCABA");  
    lena=strlen(a);  
    lenb=strlen(b);  
    printf("%d\n",LCS(0,0));  
    return 0;  
}  
int LCS(int i,int j)  
{  
    if(i>=lena || j>=lenb)  
        return 0;  
    if(a[i]==b[j])  
        return 1+LCS(i+1,j+1);  
    else  
        return LCS(i+1,j)>LCS(i,j+1)? LCS(i+1,j):LCS(i,j+1);  
}  
```
这种处理方法优点是编程简单，非常容易理解。缺点是效率太低了，有大量的重复执行递归调用，一般情况下面试官是不会满意的。另一个致命的缺点是只能求出最大公共子序列的长度，求不出具体的最大公共子序列，而在大部分笔试或者面试时会要求我们求出具体的最大公共子序列。
#### 动态规划
这里依旧可以采用动态规划的方法来解决这个问题，可以借助一个二维数组来标识中间计算结果，避免重复的计算来提高效率，可能需要消耗一部分空间，但是时间复杂度大大降低。
如下图所示的两个串，求解最长公共子序列的过程很明了：

- 设有字符串a[0...n]，b[0...m]，字符串a对应的是二维数组num的行，字符串b对应的是二维数组num的列。我们有以下的递推公式：
- 我们在程序中，可以使用二维数组flag来记录下标i和j的走向。数字"1"表示，斜向下；数字"2"表示，水平向右；数字"3"表示，竖直向下。这样我们可以求解出行进的路径，从而得到最长公共子序列。代码如下：  
       
```
#include<stdio.h>  
#include<string.h>  
char a[500],b[500];  
char num[501][501]; ///记录中间结果的数组  
char flag[501][501];    ///标记数组，用于标识下标的走向，构造出公共子序列  
void LCS(); ///动态规划求解  
void getLCS();    ///采用倒推方式求最长公共子序列  
int main()  
{  
    int i;  
    strcpy(a,"ABCBDAB");  
    strcpy(b,"BDCABA");  
    memset(num,0,sizeof(num));  
    memset(flag,0,sizeof(flag));  
    LCS();  
    printf("%d\n",num[strlen(a)][strlen(b)]);  
    getLCS();  
    return 0;  
}  
void LCS()  
{  
    int i,j;  
    for(i=1;i<=strlen(a);i++)  
    {  
        for(j=1;j<=strlen(b);j++)  
        {  
            if(a[i-1]==b[j-1])   ///注意这里的下标是i-1与j-1  
            {  
                num[i][j]=num[i-1][j-1]+1;  
                flag[i][j]=1;  ///斜向下标记  
            }  
            else if(num[i][j-1]>num[i-1][j])  
            {  
                num[i][j]=num[i][j-1];  
                flag[i][j]=2;  ///向右标记  
            }  
            else  
            {  
                num[i][j]=num[i-1][j];  
                flag[i][j]=3;  ///向下标记  
            }  
        }  
    }  
}  
void getLCS()  
{  
    char res[500];  
    int i=strlen(a);  
    int j=strlen(b);  
    int k=0;    ///用于保存结果的数组标志位  
    while(i>0 && j>0)  
    {  
        if(flag[i][j]==1)   ///如果是斜向下标记  
        {  
            res[k]=a[i-1];  
            k++;  
            i--;  
            j--;  
        }  
        else if(flag[i][j]==2)  ///如果是斜向右标记  
            j--;  
        else if(flag[i][j]==3)  ///如果是斜向下标记  
            i--;  
    }  
    for(i=k-1;i>=0;i--)  
        printf("%c",res[i]);  
}  
```
### 5.字符串编辑距离
给定一个源字符串和目标字符串，能够对源串进行如下操作:  

- 在给定位置上插入一个字符
- 替换任意字符
- 删除任意字符  

求通过以上操作使得源字符串和目标字符串一致的最小操作步数。  

简单描述一下解该题的思想，源字符串和目标字符串分别为str_a、str_b，二者的长度分别为la、lb，定义f[i,j]为子串str_a[0...i]和str_b[0...j]的最小编辑距离，简单分析可知求得的str_a[0...i]和str_b[0...j]的最小编辑距离有一下三种可能：  
  
- 去掉str_a[0...i]的最后一个字符跟str_b[0...j]匹配，则f[i, j]的值等于f[i-1, j]+1；
- 去掉str_b[0...j]的最后一个字符跟str_a[0...i]匹配，则f[i, j]的值等于f[i, j-1]+1；
- 去掉str_a[0...i]和str_b[0...j]的最后一个字符，让二者匹配求得f[i-1, j-1]，计算f[i, j]时要考虑当前字符是否相等，如果str_a[i]==str_b[j]说明该字符不用编辑，所以f[i, j]的值等于f[i-1, j-1]，如果str_a[i]!=str_b[j]说明该字符需要编辑一次（任意修改str_a[i]或者str_b[j]即可），所以f[i, j]的值等于f[i-1, j-1]+1。
- 因为题目要求的是最小的编辑距离，所以去上面上中情况中的最小值即可，因此可以得到递推公式：
   ` f[i, j] = Min ( f[i-1, j]+1,   f[i, j-1]+1,   f[i-1, j-1]+(str_a[i]==str_b[j] ? 0 : 1) ) `
           
```
//求两个字符串的编辑距离问题  
//递归版本，备忘录C[i,j]表示strA[i]...strA[size_A-1]与strB[j]...strB[size_B-1]的编辑距离  
int editDistance_mem(char *strA,int size_A,char *strB,int size_B){  
 int **C=new int*[size_A+1];  
 for(int i=0;i<=size_A;i++){  
  C[i]=new int[size_B+1]();  
 }  
 //初始化  
 for(int i=0;i<=size_A;i++){  
  for(int j=0;j<=size_B;j++)  
   C[i][j]=INT_MAX;  
 }  
 int res=EDM(C,strA,0,size_A-1,strB,0,size_B-1);  
 //free mem  
 for(int i=0;i<=size_A;i++){  
  delete [] C[i];  
 }  
 delete [] C;  
 return res;  
}  
int EDM(int **C,char *strA,int i,int A_end,char *strB,int j,int B_end){  
 if(C[i][j]<INT_MAX)//做备忘  
  return C[i][j];  
 if(i>A_end){  
  if(j>B_end)  
   C[i][j]=0;  
  else  
   C[i][j]=B_end-j+1;  
 }else if(j>B_end){  
  if(i>A_end)  
   C[i][j]=0;  
  else  
   C[i][j]=A_end-i+1;  
 }  
 else if(strA[i]==strB[j])  
  C[i][j]=EDM(C,strA,i+1,A_end,strB,j+1,B_end);  
 else{  
  int a=EDM(C,strA,i+1,A_end,strB,j+1,B_end);  
  int b=EDM(C,strA,i,A_end,strB,j+1,B_end);  
  int c=EDM(C,strA,i+1,A_end,strB,j,B_end);  
  C[i][j]=min(a,b,c)+1;  
 }  
 return C[i][j];  
}  
```

#### 矩阵标记法  

递推方法（也可称为矩阵标记法），通过分析可知可以将f[i, j]的计算在一个二维矩阵中进行，上面的递推式实际上可以看做是矩阵单元的计算递推式，只要把矩阵填满了，f[la-1, lb-1]的值就是要求得最小编辑距离。代码如下：  
        
```
//求两个字符串的编辑距离问题  
//递推版本 C[i,j]表示strA[i]...strA[size_A-1]与strB[j]...strB[size_B-1]的编辑距离  
int editDistance_iter(char *strA,int size_A,char *strB,int size_B){  
 int **C=new int*[size_A+1];  
 for(int i=0;i<=size_A;i++){  
  C[i]=new int[size_B+1]();  
 }  
 for(int i=size_A;i>=0;i--){  
  for(int j=size_B;j>=0;j--){  
   if(i>size_A-1){  
    if(j>size_B-1)  
     C[i][j]=0;  
    else  
     C[i][j]=size_B-j;  
   }else if(j>size_B-1){  
    if(i>size_A-1)  
     C[i][j]=0;  
    else  
     C[i][j]=size_A-i;  
   }else if(strA[i]==strB[j])  
    C[i][j]=C[i+1][j+1];  
   else  
    C[i][j]=min(C[i+1][j+1],C[i+1][j],C[i][j+1])+1;  
  }  
 }  
 int res=C[0][0];  
 //free mem  
 for(int i=0;i<=size_A;i++){  
  delete [] C[i];  
 }  
 delete [] C;  
 return res;  
}  
```

### 6.最长不重复子串
很好理解，即求一个串内最长的不重复子串。  

#### 方法一:使用Hash  

要求子串中的字符不能重复，判重问题首先想到的就是hash，寻找满足要求的子串，最直接的方法就是遍历每个字符起始的子串，辅助hash，寻求最长的不重复子串，由于要遍历每个子串故复杂度为O(n^2)，n为字符串的长度，辅助的空间为常数hash[256]。代码如下：  

```
/* 最长不重复子串 我们记为 LNRS */  
int maxlen;  
int maxindex;  
void output(char * arr);  
/* LNRS 基本算法 hash */  
char visit[256];  
void LNRS_hash(char * arr, int size)  
{  
    for(int i = 0; i < size; ++i)  
    {  
        memset(visit,0,sizeof(visit));  
        visit[arr[i]] = 1;  
        for(int j = i+1; j < size; ++j)  
        {  
            if(visit[arr[j]] == 0)  
            {  
                visit[arr[j]] = 1;  
            }  
else  
            {  
                if(j-i > maxlen)  
                {  
                    maxlen = j - i;  
                    maxindex = i;  
                }  
                break;  
            }  
        }  
    }  
    output(arr);  
}  
```

#### 方法二: 动态规划法
字符串的问题，很多都可以用动态规划处理，比如这里求解最长不重复子串，和前面讨论过的最长递增子序列问题就有些类似，在LIS(最长递增子序列)问题中，对于当前的元素，要么是与前面的LIS构成新的最长递增子序列，要么就是与前面稍短的子序列构成新的子序列或单独构成新子序列。
这里我们采用类似的思路：某个当前的字符，如果它与前面的最长不重复子串中的字符没有重复，那么就可以以它为结尾构成新的最长子串；如果有重复，那么就与某个稍短的子串构成新的子串或者单独成一个新子串。  

我们来看看下面两个例子：  

- 字符串“abcdeab”，第二个a之前的最长不重复子串是“abcde”，a与最长子串中的字符有重复，但是它与稍短的“bcde”串没有重复，于是它可以与其构成一个新的子串，之前的最长不重复子串“abcde”结束；  

- 字符串“abcb”，跟前面类似，最长串“abc”结束，第二个字符b与稍短的串“c”构成新的串；  

我们貌似可以总结出一些东西：当一个最长子串结束时（即遇到重复的字符），新的子串的长度是与（第一个重复的字符）的下标有关的。
于是类似LIS，对于每个当前的元素，我们“回头”去查询是否有与之重复的，如没有，则最长不重复子串长度+1，如有，则是与第一个重复的字符之后的串构成新的最长不重复子串，新串的长度便是当前元素下标与重复元素下标之差。
可以看出这里的动态规划方法时间复杂度为O(N^2)，我们可以与最长递增子序列的动态规划方案进行对比，是一个道理的。代码如下：  
  
```
[cpp] view plain copy
/* LNRS 动态规划求解 */  
int dp[100];  
void LNRS_dp(char * arr, int size)  
{  
    int i, j;  
    maxlen = maxindex = 0;  
    dp[0] = 1;  
    for(i = 1; i < size; ++i)  
    {  
        for(j = i-1; j >= 0; --j)  
        {  
            if(arr[j] == arr[i])  
            {  
                dp[i] = i - j;  
                break;  
            }  
        }  
        if(j == -1)  
        {  
            dp[i] = dp[i-1] + 1;  
        }  
        if(dp[i] > maxlen)  
        {  
            maxlen = dp[i];  
            maxindex = i + 1 - maxlen;  
        }  
    }  
    output(arr);  
}  
```

### 7.最长回文子串
给出一个字符串S，找到一个最长的连续回文串。例如串 babcbabcbaccba 最长回文是:abcbabcba  

#### 方法一:自中心向两端寻找  

回文是一种特殊的字符串，我们可以以源字符串的每个字符为中心，依次寻找出最长回文子串P0, P1,...,Pn。这些最长回文子串中的最长串Pi = max(P1, P2,...,Pn)即为所求。核心代码如下：  

```
string find_lps_method1(const string &str)  
{  
    int center = 0, max_len = 0;  
    for(int i = 1; i < str.length()-1; ++i)  
    {  
        int j = 1;  
        //以str[i]为中心，依次向两边扩展，寻找最长回文Pi  
        while(i+j < str.length() && i-j >= 0 && str[i+j] == str[i-j])  
            ++j;  
        --j;  
        if(j > 1 && j > max_len)  
        {  
            center = i;  
            max_len = j;  
        }  
    }  
    return str.substr(center-max_len, (max_len << 1) + 1);  
}  
```

#### 方法二:利用最长公共字串的方法
这里用到了一个我们观察出来的结论：对于串S, 假设它反转后得到的串是S', 那么S的最长回文串是S和S'的最长公共字串。
例如 S = babcbabcbaccba,  S' = abccabcbabcbab，S和S'的最长公共字串是 abcbabcba也是S的最长回文字串。
代码这个地方就不写了，用首指针++，尾指针--很容易实现串的翻转，再结合前面写过的最长公共子串代码可得到最后结果。  
