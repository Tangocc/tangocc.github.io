---
layout:     post
title:      "面试之常用算法"
subtitle:   ""
date:       2017-10-07 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - 面试
    - 算法
---

## 最短路径算法

```

#include <queue>
#define MAX_NODE_NUM 10  //节点个数
#define MAXINT  65535
int Graph[MAX_NODE_NUM][MAX_NODE_NUM];

bool SPFA(int src, int dest)//求最短路径的SPFA算法  
{
	int distance[MAX_NODE_NUM];//距离
	bool inQueue[MAX_NODE_NUM];//判断是否在队列中
	queue<int> Q;//队列
	int u;
	for (int i = 0; i <= MAX_NODE_NUM; i++)//初始化  
	{
		distance[i] = MAXINT;
		inQueue[i] = false;
		path[i] = -1;
	}
	
	
		distance[src] = 0;
		inQueue[src] = true;
		path[src] = -1;
		Q.push(src);
	
	while (!Q.empty())
	{
		u = Q.front();
		Q.pop();
		inQueue[u] = 0;
		for (int i = 0; i < MAX_NODE_NUM; ++i)//更新u的邻接点的dist[], pre[], inq[]  
		{
			if (distance[i] > distance[u] + Grapg[u][i])//松弛操作  
			{
				distance[i] = distance[u] + maps[u][i].cost;
				path[i] = u;
				if (!inQueue[i])
				{
					Q.push(i);
					inQueue[i] = true;
				}
			}
		}
	}
	
	return distance[dest] != MAXINT;
}



const int MAX_NODE_NUM = 10;
int dist[MAX_NODE_NUM];
int prev[MAX_NODE_NUM];
int Graph[MAX_NODE_NUM][MAX_NODE_NUM];

void Dijkstra(int v0)
{
  　　bool S[MAX_NODE_NUM];                                  // 判断是否已存入该点到S集合中
      int n=MAX_NODE_NUM;
  　　for(int i=1; i<=n; ++i)
 　　 {
      　　dist[i] = Graph[v0][i];
      　　S[i] = false;                                // 初始都未用过该点
      　　if(dist[i] == MAXINT)    
            　　prev[i] = -1;
 　　     else 
            　　prev[i] = v0;
   　　}
   　 dist[v0] = 0;
   　 S[v0] = true; 　　
 　　 for(int i=2; i<=n; i++)
 　　 {
       　　int mindist = MAXINT;
       　　int u = v0; 　　                            // 找出当前未使用的点j的dist[j]最小值
      　　 for(int j=1; j<=n; ++j)
      　　    if((!S[j]) && dist[j]<mindist)
      　　    {
         　　       u = j;                             // u保存当前邻接点中距离最小的点的号码 
         　 　      mindist = dist[j];
       　　   }
       　　S[u] = true; 
       　　for(int j=1; j<=n; j++)
       　　    if((!S[j]) && Graph[u][j]<MAXINT)
       　　    {
           　    　if(dist[u] + Graph[u][j] < dist[j])     //在通过新加入的u点路径找到离v0点更短的路径  
           　    　{
                   　　dist[j] = dist[u] + Graph[u][j];    //更新dist 
                   　　prev[j] = u;                    //记录前驱顶点 
            　　    }
        　    　}
   　　}
}
```


## 最大数组和

```
/*
求子数组的最大和（数组）
题目：
输入一个整形数组，数组里有正数也有负数。
数组中连续的一个或多个整数组成一个子数组，每个子数组都有一个和。
求所有子数组的和的最大值。要求时间复杂度为 O(n)。
*/
int MaxSum(int* arr, int len){

    int max = arr[0];
    int sum = arr[0];

    for (int ii = 1; ii < len; ii++)
    {
        if (sum + arr[ii]<arr[ii]){

            sum = arr[ii];
            if (sum>max)
            {
                max = sum;
            }
        }
        else{
            sum = sum + arr[ii];
            if (sum > max)
            {
                max = sum;
            }
        }
    }
    return max;
}


```

## 翻转单链表

```
/*****************************************
翻转单链表
******************************************/
TreeNode* TranverseList(TreeNode* list)
{
    TreeNode* pFirst = NULL;
    TreeNode* pCurrent = NULL;
    TreeNode* pEnd = NULL;
    if (list == NULL || list->right == NULL)
    {
        return list;
    }
    

        pFirst = list;
        pCurrent = pFirst->right;
        pEnd = pCurrent->right;
        pFirst->right = NULL;

        while (pEnd != NULL)
        {
            pCurrent->right = pFirst;

            pFirst = pCurrent;
            pCurrent = pEnd;
            pEnd = pEnd->right;
        }
        pCurrent->right = pFirst;

        return pCurrent;
}

```

## 翻转单词顺序

```
/*******************************************************
第 10 题（字符串）
翻转句子中单词的顺序。
题目：输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。
句子中单词以空格符隔开。为简单起见，标点符号和普通字母一样处理。
例如输入“I am a student.”，则输出“student. a am I”。
*******************************************************/
#include "vector"
void InverseSentence(char* pString)
{
    string temp = "";
    vector<string> strDeque;
    int num = 0;

    while (*pString != '\0')
    {

        if (*pString != ' ')
        {

            temp = temp + *pString;
            pString++;
        }
        else
        {
            temp += '\0';
            strDeque.push_back(temp);
            num++;
            pString++;
            temp = "";
        }
    }
    strDeque.push_back(temp);
 
    while (num > 0 || num == 0)
    {
        num--;
    }
}
```

## 判断整数序列是不是二元查找树的后序遍历结果

```
/****************************************************************
判断整数序列是不是二元查找树的后序遍历结果
题目：输入一个整数数组，判断该数组是不是某二元查找树的后序遍历的结果。
如果是返回 true，否则返回 false。
例如输入 5、7、6、9、11、10、8，由于这一整数序列是如下树的后序遍历结果：
8
/      \
6     10
/    \      /    \
5   7    9   11
因此返回 true。
如果输入 7、4、6、5，没有哪棵树的后序遍历的结果是这个序列，因此返回 false。
************************************************************************/
bool IsBST(int arr[], int start, int end)
{
    int root = arr[end];

    if (start == end)
    {
        return true;
    }
    int ii = 0;
    for (ii = start; ii < end; ii++)
    {
        if (arr[ii]>root)
            break;
    }
    int jj = ii;

    for (jj = ii; jj < end; jj++)
    {
        if (arr[jj]<root)
        {
            return false;
        }
    }

    return IsBST(arr, start, ii - 1) && IsBST(arr, ii, end - 1);

}


```

## 二叉树中节点的最大距离

```

/****************************************************
求二叉树中节点的最大距离...
如果我们把二叉树看成一个图，父子节点之间的连线看成是双向的，
我们姑且定义"距离"为两节点之间边的个数。
写一个程序，
求一棵二叉树中相距最远的两个节点之间的距离。
*******************************************************/
#include "deque"
int TreeDepth(TreeNode *tree)
{
    deque<TreeNode* > deques;
    TreeNode *pNode;

    int depth = 0;
    int rear = 0;
    int front = 0;
    int level = 1;
    deques.push_back(tree);
    rear = 1;
    while (front < rear)
    {
        pNode = deques.front();
        deques.pop_front();
        front++;
        if (pNode->left != NULL)
        {
            deques.push_back(pNode->left);
            rear++;
        }
        if (pNode->right != NULL)
        {
            deques.push_back(pNode->right);
            rear++;
        }

        if (front == level)
        {
            level = rear;
            depth++;
        }
    }
    return depth;


    //方法二
    //if (tree==NULL)
    //{
    //    return 0;
    //}
    //else{
    //
    //    return TreeDepth(tree->left) > TreeDepth(tree->right) ? TreeDepth(tree->left) + 1 : TreeDepth(tree->right) + 1;
    //
    //}
}
int LongPath(TreeNode* tree)
{
    return TreeDepth(tree->left) + TreeDepth(tree->right);

}

```

## 输入一颗二元查找树，将该树转换为它的镜像

```
/**********************************************
第 15 题（树）：
题目：输入一颗二元查找树，将该树转换为它的镜像，
即在转换后的二元查找树中，左子树的结点都大于右子树的结点。
用递归和循环两种方法完成树的镜像转换。
例如输入：
8
/ \
6 10
/ \ /  \
5 7  9 11
输出：
8
/    \
10    6
/ \    /  \
11 9 7 5
***********************************************/
void MirrorImage(TreeNode *root)
{
    if (root->left != NULL || root->right != NULL)
    {
        TreeNode *pNode;

        pNode = root->left;
        root->left = root->right;
        root->right = pNode;
        MirrorImage(root->left);
        MirrorImage(root->right);
    }


}

```

## 字符串转整数

```
/*******************************************
第 20 题字符串转整数：
题目：输入一个表示整数的字符串，把该字符串转换成整数并输出。
例如输入字符串"345"，则输出整数 345。
*******************************************/
bool illegal = false;
long long String2Number(char *strNum)
{
    if (strlen(strNum) == 0 || strNum == NULL)
    {
        illegal = true;
    }
    bool positive = true;

    if (*strNum == '+')
    {
        positive = true;
    }
    if (*strNum == '-')
    {
        positive = false;
    }

    int len = strlen(strNum);
    long long result = 0;
    long weight = 1;
    if (positive == true)
    {
        for (int ii = len - 1; ii >= 0; ii--)
        {
            if ('0' <= *(strNum + ii) && *(strNum + ii) <= '9')
            {
                result += weight*(int)(*(strNum + ii) - '0');
                weight = weight * 10;
            }
            else
            {
                illegal = false;
                return -1;
            }

        }
    }
    else{

        for (int ii = len; ii > 0; ii--)
        {
            if ('0' <= *(strNum + ii) && *(strNum + ii) <= '9')
            {
                result += weight*(int)(*(strNum + ii) - '0');
                weight = weight * 10;
            }
            else
            {
                illegal = false;
                return -1;
            }

        }
        result = 0 - result;

    }

    return result;
}
```

## n个数之和等于m
```
/************************************
第 21 题（数组）
2010 年中兴面试题
编程求解：
输入两个整数  n  和  m，从数列 1，2，3.......n  中随意取几个数,
使其和等于  m ,要求将其中所有的可能组合列出来.
*****************************/
void FindNum(int n, int sum, vector<int> *result)
{
	if (n < 0 || sum < 0)
	{
		return;
	}

	if (sum == 0)
	{
		//cout
		vector<int>::iterator  iter = result->begin();
		while (iter != result->end())
		{
			cout << *iter << " ";
			iter++;
		}
		cout << endl;
	}
	else{
		result->push_back(n);
		FindNum(n - 1, sum - n, result);
		result->pop_back();
		FindNum(n - 1, sum, result);
	}
}
```

## 左旋转字符串

```

/******************************
26.左旋转字符串（字符串）
题目：
定义字符串的左旋转操作：把字符串前面的若干个字符移动到字符串的尾部。
如把字符串 abcdef 左旋转 2 位得到字符串 cdefab。请实现字符串左旋转的函数。
要求时间对长度为 n 的字符串操作的复杂度为 O(n)，辅助内存为 O(1)。
********************/
void Rotate(char* pChar, int start, int end)
{
    int ii = start;
    int jj = end;
    char charBuff;
    while (ii < jj)
    {
        charBuff = *(pChar + ii);
        *(pChar + ii) = *(pChar + jj);
        *(pChar + jj) = charBuff;
        ii++;
        jj--;
    }

}
void LeftRotate(char *pStr, int nTh, int nLen)
{
    Rotate(pStr, 0, nLen - 1);
    Rotate(pStr, 0, nLen - nTh - 1);
    Rotate(pStr, nLen - nTh, nLen - 1);

    while (*pStr != '\0')
    {
        cout << *(pStr);
        pStr++;
    }
    cout << endl;
}

```

## 输出比赛名次
```

/**************************************
n 支队伍比赛，分别编号为 0，1，2。。。。n-1，已知它们之间的实力对比关系，
存储在一个二维数组 w[n][n]中，w[i][j]  的值代表编号为 i，j 的队伍中更强的一支。
所以 w[i][j]=i  或者 j，现在给出它们的出场顺序，并存储在数组 order[n]中，
比如 order[n] = {4,3,5,8,1......}，那么第一轮比赛就是 4 对 3，  5 对 8。.......
胜者晋级，败者淘汰，同一轮淘汰的所有队伍排名不再细分，即可以随便排，
下一轮由上一轮的胜者按照顺序，再依次两两比，比如可能是 4 对 5,直至出现第一名
编程实现，给出二维数组 w，一维数组 order  和  用于输出比赛名次的数组 result[n]，
求出 result。
*******************************/
void Race(int w[][4], int* order, int *result, int numOfTeams, int lose)
{
    bool flag = false;
    if ((numOfTeams - lose) % 2)
    {
        flag = false;
    }
    else
    {
        flag = true;
    }

    if (lose == numOfTeams)
    {
        cout << result[0] << " " << result[1] << " " << result[2] << " " << result[3] << endl;
        return;
    }
    else{
        int win = 0;
        int ilose = lose;
        for (int ii = 0; ii < 2 * ((numOfTeams - ilose) / 2); ii = ii + 2)
        {

            if (w[order[ii]][order[ii + 1]] == order[ii])
            {
                order[win] = order[ii];
                result[numOfTeams - lose - 1] = order[ii + 1];
            }
            else
            {
                order[win] = order[ii + 1];
                result[numOfTeams - lose - 1] = order[ii];
            }
            win++;
            lose++;
        }
        if (!flag)
        {
            order[win] = order[numOfTeams - lose - 1];
        }
        if ((numOfTeams - lose) == 1)
        {

            result[numOfTeams - lose - 1] = order[0];
            lose++;
        }
        Race(w, order, result, numOfTeams, lose);

    }

}

```
## 非递归实现树的深度
```
//非递归实现树的深度
int FindLongestPath(int **G, int numOfV, int from)
{
    //求树的深度
    deque<int >   nodes;
    int level = 0;
    int front = 0;
    int rear = 0;
    int start = 0;
    int nDepth = 0;
    nodes.push_back(from);
    rear++;
    level = rear;
    while (front < level){

        start = nodes.at(front++);

        for (int ii = 0; ii < numOfV; ii++){
            if (G[start][ii] == 1)
            {
                nodes.push_back(ii);
                rear++;
            }
        }

        if (front == level)
        {
            nDepth++;
            level = rear;
        }
        if (nDepth>numOfV)
        {
            nDepth = -1;
            break;
        }

    }
    return nDepth;
}

```

## 8皇后问题

```
/**************************************
//8皇后问题：输出所有可能的解
*******************/

static int num = 0;
bool isLegal(int xth, int *path)
{
    for (int ii = 0; ii < xth; ii++)
    {
        if (path[ii] == path[xth] || abs(path[ii] - path[xth]) == abs(ii - xth))
        {
            return false;
        }

    }
    return true;
}
void Queue(int xTh, int numOfQueue, int *path)
{
    if (xTh > (numOfQueue - 1))
    {
        num++;
        for (int ii = 0; ii < numOfQueue; ii++)
        {
            cout << path[ii] << " ";

        }
        cout << endl;
    }
    else
    {
        for (int ii = 0; ii < numOfQueue; ii++)
        {
            path[xTh] = ii;
            if (isLegal(xTh, path))
            {
                Queue(xTh + 1, numOfQueue, path);
            }

        }
    }

}
```

## 二分查找

```
/*
48.微软（运算）二分查找：
一个数组是由一个递减数列左移若干位形成的，比如{4，3，2，1，6，5}
是由{6，5，4，3，2，1}左移两位形成的，在这种数组中查找某一个数。
*/

bool FindNumber(int *arr,int num,int len)
{
   
	int start = 0;
	int end = len - 1;
	int mid = 0;

	while (end >=start)
	{

		mid = (start + end) / 2;

		if (arr[mid]==num)
		{
			return true;
		}

		if (arr[mid]<=arr[start])
		{
			if (arr[mid]< num  && num <= arr[start])
			{
				end = mid-1;
			}
			else
			{
				start = mid+1;
			}
		}
		else
		{
		   if (arr[mid]>num  &&  num>=arr[end])
				   start = mid+1;
		   else
				   end = mid-1;
			   
		   }
	} 

	return false;
}
```

## 输入一个数N，输出0-N中7 的个数

```
/************************************************************************/
/* 输入一个数N，输出0-N中7 的个数。                             */
/************************************************************************/

int findNot7(int number)
{
		if (number <= 0)
		{
			return 0;
		}
		//计算出N的位数和最高位、次高位的数字
		int sum = 0;
		int length = 0;
		int val = 1;
		while (sum<number)
		{
			sum = sum * 10 + 9;
			++length;//位数
			val = val * 10;
		}
		val = val / 10;
		int Hnum =0;   //最高位
		int numberOf7 = 1;

		if (length==1)
		{
		   Hnum = number;   //最高位
		 
		}
		else
		{
			 Hnum = (number) / (val);   //最高位
		}
		int step = length;
		while (--step>0)
		{
			numberOf7 = numberOf7 * 9;
		}

		if (Hnum == 7 )
		{
			return numberOf7*(Hnum) - number % (val)+findNot7(number % (val));
		}
		else
	   if (Hnum>7)
		{
			return numberOf7*(Hnum-1) + findNot7(number % (val));
		}
		else
		{
			
			return numberOf7*(Hnum)+findNot7(number % (val));
		}
	 
}
int find7(int number)
{

	if (number <= 0)
	{
		return 0;
	}
	//计算出N的位数和最高位、次高位的数字
	int sum = 0;
	int length = 0;
	int val = 1;
	while (sum < number)
	{
		sum = sum * 10 + 9;
		++length;//位数
		val = val * 10;
	}
	val = val / 10;
	int Hnum = 0;   //最高位
	int numberOf7 = 1;

	if (length == 1)
	{
		Hnum = number;   //最高位

	}
	else
	{
		Hnum = (number) / (val);   //最高位
	}
	int step = length;
	while (--step > 0)
	{
		numberOf7 = numberOf7 * 10;
	}

	if (Hnum==7)
	{
		return number%val + 1 + Hnum * (numberOf7) / 10 * (length - 1) + find7(number%val);
	}else
	if (Hnum>7)
	{
		return Hnum * (numberOf7) / 10 * (length - 1) + numberOf7 + find7(number%val);
	}
	else
	{
		return Hnum * (numberOf7) / 10 * (length - 1)  + find7(number%val);
	}

}

```

## 字符串转换为数字
```
/******************************************
 字符串转换为数字
*************************************/

int  String2Int(char *pString)
{
	assert(pString!=NULL);

	if (strcmp(pString,"")==0)
	{
		return 0;
	}

	int result = 0;
	int  temp = 1;

	int ii = 0;
	bool  flag = false;//符号位标识符

	if (pString[0]=='-')
	{
		flag = true;  //负数
		ii = 1;
	}
	else if (pString[0] == '+')
	{
		flag = false;  //正数
		ii = 1;
	}

	while (pString[ii]!='\0')
	{

		//非法值判定
		if (pString[ii]<'0'  ||pString[ii]>'9')
		{
			return 0;
		}
		result =  result*temp +   (pString[ii++] - '0') ;
		temp *= 10;
	}

	if (flag)
	{
		result = -result;
	}

	return result;
}
```

## 任务调度算法
```
/*******************************
美团校园招聘：
   任务调度算法：将一个n个数的序列分到m份，使得每一份的和尽可能相等。
*******************************/
#include <algorithm>
void AssignWork(int *nums,int numOfWork,int numOfService)
{
	
	assert(nums != NULL);

	if (numOfService>=numOfWork)
	{
		//消耗时间就是任务中耗时最长的那个
		//找出最大的那个，时间复杂度n
		int maxTime = 0;
		for (int jj = 0; jj < numOfService; jj++)
		{
			if (nums[jj] > maxTime)
			{

				maxTime = nums[jj];
			}
		}
		cout << "------任务完成耗时最短时间为--------- " << endl;
		cout << maxTime << endl;
		return;
	}

	//快速排序
	//sort(nums,0,numOfWork-1);

	int *time = new int[numOfService];
	for (int ii = 0,jj=numOfWork-1; ii < numOfService;ii++,jj--)
	{
		time[ii] = nums[jj];
		cout << "任务  " << jj << " 分配给服务器" << ii << endl;
	}

	int minIndex;
	int minTime;
	for (int ii = numOfWork - 1-numOfService; ii >= 0;ii--)
	{
		//寻找time数组中最小值

		minIndex = 0;
		minTime = time[0];
		for (int jj = 1; jj < numOfService;jj++)
		{
			if (time[jj]<minTime)
			{
				minIndex = jj;
				minTime = time[jj];
			}
		}
		cout << "任务  " << ii << " 分配给服务器" << minIndex << endl;
		time[minIndex] += nums[ii];
	}
	cout << "------任务完成耗时最短时间为--------- " << endl;
	minTime = 0;
	for (int jj = 0; jj < numOfService; jj++)
	{
		if (time[jj] > minTime)
		{
			
			minTime = time[jj];
		}
	}
	cout << minTime << endl;
}
```

## 给定一个int 数组，找出所有的a[i],使得它左侧的数都小于等于它，右侧的数都大于等于它。
```
/**************************************
给定一个int 数组，找出所有的a[i],使得它左侧的数都小于等于它，右侧的数都大于等于它。
********************************************/
/***********************
思想：
    首先建立一个数组right，从右向左遍历给定数组，right[i]存储从size-1到nums[i]的最小值，
	然后从左向右依次遍历nums[i],设置一个变量maxInt保存nums[0]到nums[i]的最大值，比较  maxInt<=nums[i]<=right[i]，是，则nums[i]满足，否则不满足
	时间复杂度O(n), 实际是 2n
测试用例：
1.nums==NULL,size<1;
2.nums={1,3,6,4,7,9};   输出 1 3 7 9
3.nums = {9,7,6,4,3,1};  //无
4.nums = {9,9,9,9,9,9};  输出 9 9 9 9 9 9
**********************/
bool findNumbers(int *nums,int size)
{
	assert(nums != NULL &&size > 0);
	const int MAXINT = 65535;
	int *right = new int[size];
	right[size - 1] = MAXINT;
	for (int ii = size - 2; ii >= 0;ii--)
	{
		if (nums[ii+1] <= right[ii + 1])
		{
			right[ii] = nums[ii + 1];
		}
		else
		{
			right[ii] = right[ii + 1];
		}
	}
	int  maxInt = -65536;
	cout << "满足小于左边的数，大于右边的数的序列为：" << endl;
	for (int ii = 0; ii < size;ii++)
	{
		if (nums[ii]>=maxInt   && nums[ii]<=right[ii])
		{
			//找到了，加入结果集(直接输出)
			cout << nums[ii]<<"  ";
			result.push_back(nums[ii]);
			maxInt = nums[ii];
		}
		else
		{
		    if (nums[ii]>=maxInt)
		    {
				maxInt = nums[ii];
		    }
		}
	}
	cout << endl;
	delete []right;
	if (result.size()<1)
	{
		cout << "没有找到这样的数" << endl;
		return false;  //没有找到
	}
	else{
		return true; //找到了
	}
}

```

## 编程判定数字N是否是2的整数次幂

```
/*******************************
编程判定数字N是否是2的整数次幂
思路：
    判定N二进制表示的最后一位是否是1
测试用例：
    1. N= 0
	2. N = -99
	3. N = 1；
	4. N = 8
*********************************/
bool  is2times(int num)
{
	if (num<0)
	{
		return false;
	}
	return (num&(num - 1)) == 0 ? true : false;
}
```

## 路径查找

```
/********************************
一颗二叉树，每个节点包含一个整数，请设计一个算法输出所有满足条件的路径：路径上所有的整数和等于给定的值，不一定从根节点开始
******************************/
int  findSum(TreeNode *root,int sum,vector<int>  datas,int level)
{
	if (root==NULL)
	{
		return  -1;  //表示未找到 
	}

	datas.push_back(root->val);
	int tempSum = sum;
	for (int ii = level; ii >= 0;--ii)
	{
		tempSum -= datas[ii];
		if (tempSum==0)
		{
			//输出结果，说明找到了  level --ii
			for (int jj = level; jj >= ii;--jj)
			{
				cout << datas[jj] << "  " ;
			}
			cout << endl;

		}
	}
	findSum(root->left, sum, datas, level + 1);
//	datas.pop_back();
	//level -= 1;
	findSum(root->right, sum, datas, level + 1);


}

```

## 给定两个二进制字符串，输出两者的

```

/******************************
【Leetcode67】给定两个二进制字符串，输出两者的和
思路：
    例如 "1110101"+"100001"
   1.将num1与num2相加(不处理进位操作) 得到"1210102"
   2.处理进位，从后向前遍历，处理进位   得到 "0010110",flag =1
   3.如果最终flag=1,则说明需要在最高位之前插入'1',得到结果"10010110"
时间复杂度： 3*max(m,n)
******************************/

string  addBinaryString(string  num1,string num2)
{
	int len1 = num1.length();
	int len2 = num2.length();

	string addSum;
	string::iterator strIter = addSum.begin();
	
	int flag = 0;
	int diff = 0;
	if (len1>len2)
	{
		//字符串num1长
		diff = len1 - len2;

		for (int ii = 0; ii < diff;++ii)
		{
			addSum.insert(strIter, (num1[ii]));
			++strIter;
		}
		for (int ii = 0; ii < len2;++ii)
		{
			addSum.insert(strIter,( (num1[ii+diff]-'0')+(num2[ii]-'0')+'0'));
			++strIter;
		}

		cout << addSum.c_str() << endl;
	}
	else
	{
		//字符串num2长
		diff = len2 - len1;
		for (int ii = 0; ii < diff; ++ii)
		{
			addSum.insert(strIter, (num1[ii]));
			++strIter;
		}
		for (int ii = 0; ii < len1; ++ii)
		{
			addSum.insert(strIter, ( (num2[ii + diff] - '0') + (num1[ii]-'0')+'0' ));
			++strIter;
		}
		cout << addSum.c_str() << endl;
	}

	//再次遍历addSum，处理进位
	
	for (int ii = addSum.length() - 1; ii >= 0;--ii)
	{
		if ( (((addSum[ii]-'0')+flag)+'0')>'1')
		{
			addSum[ii] = '0';
			flag = 1;
		}
		else
		{
			flag = 0;
		}
	}
	//处理最高位的进位
	if (flag==1)
	{
		//最高位有进位
		addSum.insert(addSum.begin(), '1');
	}
	return addSum;
}

```
## 从N个数中寻找最大的K个数

```
/******************************************
LeetCode506 Relative Ranks
********************************/
/***********************
思想：从N个数中寻找最大的K个数，利用最小堆实现。
********************************/
void AdjustDown(vector<int> &scores, vector<int> &heap, int index, int size)
{
	int temp = heap[index];
	for (int ii = 2 * index+1; ii <= size;ii *= 2)
     {
		if (ii<size&& scores[heap[ii]]>scores[heap[ii + 1]])
		{
			ii++;
		}

		if ( scores[temp]>scores[heap[ii]])
		{
			heap[index] = heap[ii];
			index = ii;
		}
		else
		{
			break;
		}
	}
	heap[index] = temp;
}
void BuildMinHeap(vector<int> &scores, vector<int> &heap)
{
	int mid = (heap.size()) / 2;
	for (int ii = mid; ii >= 0;ii--)
	{
		AdjustDown(scores, heap, ii,heap.size()-1);
	}

}
void HeapSort(vector<int>  &scores, vector<int> &heap)
{
	int temp;
	for (int ii = 0; ii < heap.size();ii++)
     {
		temp = heap[0];
		heap[0] = heap[heap.size() - 1 - ii];
		heap[heap.size() - 1 - ii] = temp;
		AdjustDown(scores, heap, 0, heap.size() - ii-1);
     }
}
void TopK(vector<int>  scores,int topK)
{
	vector<int> heap;
	for (int ii = 0; ii < topK;ii++)
	{
		heap.push_back(ii);
	}
	BuildMinHeap(scores,heap);
	cout << "build";
	for (int ii = topK; ii < scores.size();ii++)
	{
		if (scores[ii]>scores[heap[0]])
		{
			heap[0] = ii;
			AdjustDown(scores, heap,0,heap.size()-1);
		}
	}

	HeapSort(scores,heap);

	for (int ii = 0; ii < topK;ii++)
	{
		cout << heap[ii] << "  ";
	}

}
```

## 整形数组重组，求最小值

```
/******************************
优酷面试题：整形数组重组，求最小值
*********************************/
/***********************
思想：
1.将整数数组转化为字符串数组(避免组合后超过最大整数范围)
2.对字符串数组进行冒泡排序(可以采用快速排序)
  比较原则是：  num[ii]+num[ii+1]>num[ii+1]+num[ii]（加号表示字符串的拼接）  则交换
              num[ii]+num[ii+1]<num[ii+1]+num[ii]                      不交换
3.最后排序的数组按照顺序组合后即是最小的组合数。
测试用例：
1.[321,3,32]
2.[12,1212,2,13]
************************/
string reArrangeInteger(vector<int> nums)
{
	if (nums.size()<0)
	{
		return "0";
	}
	vector<string> stringNums;
	//转换成字符串数组
	for (int ii = 0; ii < nums.size();ii++)
	{
		char buff[10];
		_itoa_s(nums[ii], buff, 10);
		stringNums.push_back(buff);
		
	}
	//冒泡排序
	for (int ii = 0; ii < stringNums.size();ii++)
	{
		for (int jj = stringNums.size() - 1; jj > ii;jj--)
		{
			string num1 = stringNums[jj-1] + stringNums[jj];
			string num2 = stringNums[jj] + stringNums[jj-1];

			if (num1.compare(num2)>0)
			{
				//交换
				string temp = stringNums[jj - 1];
				stringNums[jj - 1] = stringNums[jj];
				stringNums[jj] = temp;
			}

		}
	}
	string result;
	for (int ii = 0; ii < stringNums.size();ii++)
	{
		result += stringNums[ii];
	}
	return result;


}
```

## Reverse Nodes in K-Group

```
/****************************
LeetCode 25 Reverse Nodes in K-Group
****************************/
/********************
思想：
    1.遍历单链表，统计节点个数count
	2.count<k 则返回，否则 翻转次数为 count/k
	3.循环count/k次，翻转
	4.每次翻转要记录两个量 翻转链表的 头节点 和 尾节点
	5.判定翻转count/k次后，将后续未翻转节点连接到翻转后的链表后
**************************/

class ListNode{
public:
	int val;
	ListNode *next;
	ListNode(int val){
		this->val = val;
	};
};

ListNode *reverseNodes(ListNode *head)
{
	if (head==NULL)
	{
		return head;
	}
	ListNode *pre = NULL;
	ListNode *cur = head;
	ListNode *rear = cur->next;

	while (rear!=NULL)
	{
		cur->next = pre;
		pre = cur;
		cur = rear;
		rear = rear->next;
	}
	cur->next = pre;
	return cur;


}
ListNode *ReverseNodesInKGroup(ListNode *head, int k)
{
	if (k<=1 || head==NULL ||head->next==NULL)
	{
		return head;
	}
	//判定节点个数
	int count = 0;
	ListNode *preHead = head;
	while ( preHead != NULL)
	{
		count++;
		preHead = preHead->next;
	}
    if (count<k)
    {
		return head;
    }

	//进行count/k次翻转
	ListNode *tempHead = head;
	ListNode *firstTail = NULL;//前段翻转后的尾节点
	ListNode newHead(1);
	firstTail = &newHead;
	int num = count / k;  //翻转的次数
	for (int ii = 0; ii < num;ii++)
	{
		ListNode *secondTail = tempHead;//尾节点
		//进行翻转
		count = k-1;
		ListNode *pre = tempHead;
		ListNode *curr = pre->next;
		ListNode *rear = curr->next;
	
		while (count-- >0 &&rear!=NULL  )
		{
			curr->next = pre;
			pre = curr;
			curr = rear;
			rear = rear->next;
		}
		if (count!=-1 &&rear==NULL)
		{
			curr->next = pre;
			pre = curr;
			curr = rear;
		}
			secondTail->next = curr;
			firstTail->next = pre;
			firstTail = secondTail;
			tempHead = curr;
		
		
	}

	return newHead.next;
	

}
```

## 三个数最接近于目标值

```
/***********************************
LeetCode 16 threeSumClosest 三个数最接近于目标值
************************************/
int threeSumClosest(vector<int>  &nums,int target)
{
	if (nums.size()<3)
	{
		return 65535;
	}
	sort(nums.begin(), nums.end());
	int diff=0;
	int sum = 0;
	for (int ii = 0; ii < nums.size()-2;ii++)
	{
		int start = ii + 1;
		int end = nums.size() - 1;
		diff = nums[ii] + nums[start] + nums[end]-target;
		sum = nums[ii] + nums[start] + nums[end];
		while (start<end)
		{
			if (nums[ii]+nums[start]+nums[end]==target)
			{
				return target;
			}
			else if (nums[ii] + nums[start] + nums[end] > target)
			{
				if (abs(nums[ii] + nums[start] + nums[end] - target)<diff)
				{
					diff = abs(nums[ii] + nums[start] + nums[end] - target);
					sum = nums[ii] + nums[start] + nums[end];
				}
				
				end--;
			}
			else
			{
				if (abs(nums[ii] + nums[start] + nums[end] - target) < diff)
				{
					diff = abs(nums[ii] + nums[start] + nums[end] - target);
					sum = nums[ii] + nums[start] + nums[end];
				}

				start++;
			}

		}

	}


	return sum;

}
```

## 两个排序数组的中位数

```

/**************************
LeetCode 4  Median Of Sorted Array  两个排序数组的中位数
************************/
/*****************************
思想：
    1.定义两个指针索引  pointer1=0,pointer2=0  分别指向nums1 nums2 的首个元素
	2.数组长度和为偶数时  中位数取一个数 第(len1+len2)/2个数
	  数组长度为奇数时  中位数取两个数 第count个和第count+1个数的平均值
	3.比较pointer1 和 pointer2 指向的元素的大小，小的向后移动
	    nums1[pointer1]>nums2[pointer2] 则  pointer2++;
		nums1[pointer1]<=nums2[pointer2] 则  pointer1++;
	4.计数器count++,比较count是否等于(len1+len2)/2
	5.在比较过程中 注意判断 pointer1 和pointer2 是否到数组的末尾，处理越界问题
	
时间复杂度：
     O((m+n)/2)  m，n分别是数组长度 
	 空间复杂度 O(1)
	 各种越界处理 满屏幕都是 if....else.... 其实很简单
*************************/
double findMedianSortedArrays(vector<int> &nums1,vector<int> &nums2)
{
	if (nums1.size()<1  && nums2.size()<1)
	{
		return 0;
	}
	int pointer1 = 0;
	int pointer2 = 0;
	int median = (nums1.size() + nums2.size()-1)/2;
	int count = 0;  
	int  move = 1;  //指示哪个指针在移动  1 pointer1在移动  2 pointer2在移动  
	while (pointer1<nums1.size() && pointer2<nums2.size()  && count<median)
	{
		if (nums1[pointer1]>nums2[pointer2])
		{
			++pointer2;
			move = 2;
		}
		else
		{
			++pointer1;
			move = 1;
		}
		count++;
	}
	//分别判断结束的条件
	if (pointer1<nums1.size() && pointer2<nums2.size())
	{
		//说明count==median
		if ((nums1.size() + nums2.size())%2)//长度和为奇数，取1个数
		{
			if (move==1)  //上一次移动是 1
			{
				return nums1[pointer1];
			}
			else   //上一次移动是 2
			{
				return nums2[pointer2];
			}
		}
		else//长度和为偶数，取两个数
		{
			
			if (move == 1)  //上一次移动是 1
			{
				return (nums1[pointer1] + ((pointer1 + 1 )>=nums1.size()|| (nums1[pointer1 + 1] > nums2[pointer2]) ? nums2[pointer2] : nums1[pointer1 + 1])) / 2.0;//注意越界处理
				
			}
			else   //上一次移动是 2
			{
				return (nums2[pointer2] + (((pointer2 + 1) >= nums2.size())  || (nums2[pointer2 + 1] > nums1[pointer1]) ? nums1[pointer1] : nums2[pointer2 + 1])) / 2.0;
			}
		}
	}
	else
	{
		if (pointer1<nums1.size()) //说明数组2到末尾
		{
			while (count<median)
			{
				++pointer1;
			}
			if ((nums1.size() + nums2.size()) % 2)//长度和为奇数，取1个数
			{
				return nums1[pointer1];
			}
			else
			{
				return (nums1[pointer1] + nums1[pointer1 + 1]) / 2.0;
			}
		}
		else  //说明数组1到末尾
		{
			while (count < median)
			{
				++pointer2;
			}
			if ((nums1.size() + nums2.size()) % 2)//长度和为奇数，取1个数
			{
				return nums2[pointer2];
			}
			else
			{
				return (nums2[pointer2] + nums2[pointer2 + 1]) / 2.0;
			}
		}
			
	}
}

```


## 换钱方法

```
/*****************************
换钱方法  动态规划
***********************************/
/*****************************
思路：
   1.建立一张二维表dp,m*n 其中m表示货币的种类，行n表示 target+1(即是目标钱数+1)
      即是：(下表以目标钱数为5，钱的种类为3表示)
	  | 0 1 2 3 4 5
	-----------------
	2 | 1 0 1 0 1 0
	3 | 1 0 0 0 0 0
	4 | 1 0 0 0 0 0
   2.初始化如上表
      dp[ii][jj]  表示 用面值为 0，1，。。ii(即是当前面值包括前面的所有面值) 的货币对换 jj 元的方法数
   3.对硬币面值按从小到大排序
   4.填充表
    
	 (1)使用 0 枚面值 ii的货币     dp[ii-1][jj-0*value]
	 (2)使用 1 枚面值 ii的货币     dp[ii][jj-1*value]
	 (3)使用 2 枚面值 ii的货币     dp[ii][jj-2*value]
	 (4)....类推  直至 jj-count*value<=0为止
	 (5)将上述结果累加即是最终  dp[ii][jj]的结果。
	 dp[ii][jj] =dp[ii-1][jj]+ dp[ii][jj-1*value]+ dp[ii][jj-2*value]。。。。
时间复杂度：
     m*(n+1)  m货币种类  n 目标钱数
空间复杂度：  
     m*(n+1)  m货币种类  n 目标钱数
****************************/
int  changeMoneyWays(vector<int>  money,int target)
{
	sort(money.begin(), money.end());
    //定义二维数组
	int **dp = new int*[money.size()];

	for (int ii = 0; ii < money.size();ii++)
	{
		dp[ii] = new int[target + 1];
	}
	//填充第一列
	for (int ii = 0; ii < money.size();ii++)
	{
		dp[ii][0] = 1;
	}
	//填充第一行
	for (int ii = 1; ii < target + 1;ii++)
	{
		if (ii%money[0]==1)
		{
			dp[0][ii] = 0;
		}
		else
		{
			dp[0][ii] = 1;
		}
	}

	for (int ii = 1; ii < money.size();ii++)
	{
		for (int jj = 1; jj < target + 1;jj++)
		{
			int value = money[ii];//这个硬币的面值
			int count = 0;//使用这个面值的硬币的个数

			while (jj - count*value >= 0)//分别使用1.2.3.4...张该种面值的货币
			{
				dp[ii][jj] += dp[ii-1][jj - count*value];
				count++;
			}
		}
	}
	
	for (int ii = 0; ii < money.size(); ii++)
	{
		for (int jj = 0; jj < target + 1;jj++)
		{
			cout << dp[ii][jj] << "   ";
		}
		cout << endl;
		cout << endl;
	}
	cout << endl;
	//删除建立的数组
	int  result = dp[money.size()-1][target];
	for (int ii = 0; ii < money.size();ii++)
	{
		delete[]  dp[ii];
	}
	return result;
}


```

## 最少硬币换取目标钱数

```
/***********************************
最少硬币换取目标钱数
***************************************/
int  changeMoneyWithLeastNumber(vector<int>  money, int target)
{
	//定义二维数组
	int **dp = new int*[money.size()];

	for (int ii = 0; ii < money.size(); ii++)
	{
		dp[ii] = new int[target + 1];
	}
	//填充第一列
	for (int ii = 0; ii < money.size(); ii++)
	{
		dp[ii][0] = 0;
	}
	//填充第一行
	for (int ii = 1; ii < target + 1; ii++)
	{
		if (ii%money[0] == 0)
		{
			dp[0][ii] = ii/money[0];
		}
		else
		{
			dp[0][ii] = 65535;
		}
	}
	//填充表格
	for (int ii = 1; ii < money.size(); ii++)
	{
		for (int jj = 1; jj < target + 1; jj++)
		{
			int value = money[ii];//这个硬币的面值
			int count = 1;//使用这个面值的硬币的个数
			dp[ii][jj] = dp[ii - 1][jj];
			while (jj - count*value >= 0)//分别使用1.2.3.4...张该种面值的货币
			{
				if (dp[ii][jj]>dp[ii][jj-count*value]+count)
				{
					dp[ii][jj] = dp[ii][jj - count*value] + count;
				}
				count++;
			}
		}
	}
	//输出表格
	for (int ii = 0; ii < money.size(); ii++)
	{
		for (int jj = 0; jj < target + 1; jj++)
		{
			cout << dp[ii][jj] << "   ";
		}
		cout << endl;
		cout << endl;
	}
	cout << endl;
	//删除建立的数组
	int  result = dp[money.size() - 1][target];
	for (int ii = 0; ii < money.size(); ii++)
	{
		delete[]  dp[ii];
	}
	return result;
}

```

## 需要排序的最短子数组长度
```
/**************************************
需要排序的最短子数组长度
给定一个无需数组nums,求出需要排序的最短子数组长度
********************************************/
/***********************
思想：
方法一：
     1.对数组进行排序，得到数组sortedNums
	 2.遍历数组sortedNums和数组nums
	    如果

方法二：
   1.首先建立一个数组right，从右向左遍历给定数组，right[i]存储从size-1到nums[i]的最小值，
   2.定义flag[]数组，记录leftMax<=nums[i]<=right[i]是否满足
   3.然后从左向右依次遍历nums[i],设置一个变量leftMax保存nums[0]到nums[i]的最大值，比较  leftMax<=nums[i]<=right[i]，满足则flag[i]=true，，否则flag[i]=false
   4.遍历flag[]数组，统计连续false的个数最小的长度
时间复杂度O(n), 实际是 2n
测试用例：
1.nums==NULL,size<1;
2.nums={1,3,6,4,7,9};   输出 1 3 7 9
3.nums = {9,7,6,4,3,1};  //无
4.nums = {9,9,9,9,9,9};  输出 9 9 9 9 9 9
**********************/
int shortestRearrangeSubArray(vector<int> nums)
{
	int size = nums.size();
	int *right = new int[size];
	int rightMin = 65535;
	int leftMax = -65536;
	for (int ii = size - 1; ii >= 0;ii--)
	{
		right[ii] = rightMin;
		if (nums[ii]<rightMin)
		{
		    rightMin = nums[ii];
		}
	}
	int minLen = 65535;
	int countLen = 0;
	//bool flag = false;
	bool *flag = new bool[size];

	for (int ii = 0; ii < size;ii++)
	{
		if ( (leftMax<=nums[ii] && nums[ii]<= right[ii]))
		{
			if (leftMax <= nums[ii])
			{
				leftMax = nums[ii];
			}
		/*	if (flag &&(countLen < minLen))
			{
				minLen = countLen+1;
				countLen = 0;
			}
			flag = false;*/
			flag[ii] = true;
		}
		else
		{
			flag[ii] = false;
		 
		}
	
	}

	for (int ii = 0; ii < size;ii++)
	{
		if (flag[ii]==false)
		{
			countLen = 0;
			while (ii<size && !flag[ii])
			{
				countLen++;
				ii++;
			}
			if (minLen>countLen)
			{
				minLen = countLen;
			}
		}

	}
	/*if (flag[size-1]==false)
	{
		if (minLen > countLen)
		{
			minLen = countLen;
		}
	}*/
	delete[] right;

	return minLen+1;
}
```

## Rearrange String k Distance Apart
```
/******************************
LeetCode 358 Rearrange String k Distance Apart
******************************/
/******************************
思想：
    1.建立map,遍历字符串，统计字符出现的次数
	2.定义一个数组nextPosition[26]，存储字符下次可以出现的位置
	3.字符串长度的循环体
	    每次寻找剩下字符中，个数最多的字符，且要满足的条件是 此处的位置大于nextPosition[i]
		存在这样的字符，则加入结果字符
		不存在的话，则返回""，说明不存在这样的字符串
		注意维护nextPosition数组
时间复杂度：
   n+26*n   n是字符串的长度
空间复杂度
   O(1)
测试用例：
*******************************/
string rearrangeString(string str, int k)
{
	map<char, int> hashmap;
	int size = str.size();
	//建立hash表，统计每个字符出现的次数
	for (int ii = 0; ii < size;++ii)
	{
		hashmap[str[ii]] += 1;
	}
	int nextPosition[26];//存储字符下次出现的位置
	memset(nextPosition, 0, sizeof(nextPosition));
	string result;
	int count = 0;//字符串长度
	for (int ii = 0; ii < size;++ii)
	{
		//寻找剩下字符中个数最多的
		char ch='\0';
		int max = 0;
		for (map<char, int>::iterator iter = hashmap.begin(); iter != hashmap.end();++iter)
		{
			if (count >= nextPosition[iter->first - 'a']&&iter->second>max )
			{
				max = iter->second;
				ch = iter->first;
			}
		}

		//判断是否找到
		if (max>0 &&ch!='\0')
		{
			//找到了
			result += ch;
			hashmap[ch] -= 1;
			nextPosition[ch - 'a'] = count + k;//下次可以放的位置
			count++;
		}
		else
		{
			return "";
		}

	}
	return result;

}
```

## LeetCode 30  

```

/**************************
LeetCode 30  
*******************************/
/*********************************
思想：
1.建立一个hash表，map实现
2.定义两个指针left,right，遍历字符串s,找到第一个words中字符串，即找到left起始位置
3.const strLen = words[0].size()即是words中字符串的长度
3.定义一个变量count记录words中字符串已经出现的个数，count = 0时表示words中所有字符串都出现，即此时left~right即是要找的结果
4.hash[S.substr(left,strLen)]>0  说明S.substr(left,strLen)存在，且出现次数仍然不够
  hash[S.substr(left,strLen)]<=0   说明S.substr(left,strLen)存在，但是前面已经出现过了，则恢复现场，再将left右移，并维护count和hash表
 
5.遍历结束，即得出结果
时间复杂度：
O(n)
空间复杂度
O(1)
*******************************/
vector<int> findSubstring(string s,vector<string> &words)
{
	vector<int> result;
	if (words.size()<1 || s.size()<words[0].size())
	{
		return result;
	}
	
	map<string, int> hash;
	for (int ii = 0; ii < words.size();ii++)
	{
		if (hash.find(words[ii])==hash.end())
		{
			hash[words[ii]]=1;
		}
		else
		{
			hash[words[ii]]++;
		}

	}

	const int strLen = words[0].size();
	int sLen = s.size();
	int left = 0;
	while (left<sLen && hash.find(s.substr(left, strLen)) == hash.end())//找到第一个符合条件的left
	{
		left += 1;
	}
	if (left>=sLen)
	{
		return result;
	}
	int right = left;
	int count = words.size();
	while (right<sLen-strLen)
	{
		if (hash.find(s.substr(right, strLen)) != hash.end() && hash[s.substr(right, strLen)]>0)//存在
		{
			hash[s.substr(right, strLen)]--;
			count--;
			if (count==0)//找到了
			{
				result.push_back(left);
				while (left <= right)
				{
					if (hash[s.substr(left, strLen)] >= 0)
					{
						hash[s.substr(left, strLen)]++;
					}
					left += strLen;
				}
				left += 1;
				right = left;
				count = words.size();
				
			}
			else

			right += strLen;
		}
		else  //说明不存在，则left直接跳到right+1,且恢复字符串出现的次数，维护hash表
		{
			while (left<right)
			{
				if (hash[s.substr(left, strLen)]>=0)
				{
					hash[s.substr(left, strLen)]++;
				}
				left += strLen;
			}
			count = words.size();
			left += 1;
			right = left;
		}

		
	}

	return result;
}
```

## 易序词

```
/*************************
LeteCode 438  易序词
***********************/
/*********************************
思想：
   1.建立一个hash表，因为此处只有小写字母，所以用数组实现hash[26],在计算过程中既要记录字符是否出现，又要记录出现几次，所以初始化为-1
     则：  hash[i]>=0时，表示i字符出现且出现次数是hash[i]次
	      hash[i]==-1时，表示字符未出现
   2.定义两个指针left,right，遍历字符串s,找到第一个p字符，即找到left起始位置
   3.定义一个变量count记录p中字符已经出现的个数，count = 0时表示p中所有字符都出现，即此时left~right即是易序词
   4.hash[s[right] - 'a']>0  说明s[right]存在，且出现次数仍然不够
    hash[s[right] - 'a']==0  说明s[right]存在，但是前面已经出现过了，则将left右移，并维护count和hash表
	hash[s[right] - 'a']<0   说明s[right]不存在，则直接跳到right+1,在这之前要恢复left~right之间的字符个数,维护hash表
   5.遍历结束，即得出结果
时间复杂度：
   O(n)
空间复杂度
   O(1)
*******************************/

vector<int>  findAllAnagrams(string s,string p)
{
	//构建hash表
	int hash[26];//小写字母
	memset(hash, -1, sizeof(hash));
	int pLen = p.size();
	for (int ii = 0; ii < pLen; ++ii)
	{
		if (hash[p[ii] - 'a']==-1)
		{
			hash[p[ii] - 'a'] = 1;
		}
		else
		{
			hash[p[ii] - 'a'] += 1;
		}
	}
	//定义两个指针
	int left = 0;
	int sLen = s.size();
	int count = p.size();
	vector<int>  result;
	while (left<s.size()&& hash[s[left]-'a']==-1)//找到第一个字符
	{
		++left;
	}
	if (left==sLen)
	{
		return result;
	}
	int right = left;
	while (right<sLen)
	{
		if (hash[s[right] - 'a']>0)//说明s[right]存在，且出现次数仍然不够
		{
			hash[s[right]-'a']--;
			count--;
			if (count==0 )//找到了
			{
				result.push_back(left);
				hash[s[left]-'a']++;
				count++;
				left++;
			}
		}
		else if (hash[s[right] - 'a']==0)  //说明s[right]存在，但是前面已经出现过了
		{
			if (hash[s[left]-'a']>=0)
			{
				hash[s[left]-'a']++;
				count++;
				left++;
			}
			right--;
		}
		else  //说明s[right]不存在，则直接跳到right+1,在这之前要恢复left~right之间的字符个数
		{
			while (left <= right)//恢复left~right之间的字符个数
			{
				if (hash[s[left]-'a']>=0)
				{
					hash[s[left]-'a']++;
				}
				left++;
			}
			count = pLen;
		}
		right++;
	}
	return result;
}
//归并排序
#define MAX_SIZE  10
//将SR中SR[start...mid]和SR[mid+1...end]归并到TR[start...end]中
void Merge(int *SR,int *TR,int start,int mid,int end)
{
    if (SR==NULL || TR==NULL)
    {
        return;
    }
    int pre = start;
    int lat = mid+1;
    int s=0;
    while (pre<=mid &&lat<=end)
    {
        if (SR[pre]>SR[lat])
        {
            TR[s] = SR[lat++];
        }
        else
        {
            TR[s] = SR[pre++];
        }
        s++;
    }
    //判定哪个数组到达边界
    if(pre<=mid)
    {
        while (pre <= mid)
        {
            TR[s++] = SR[pre++];
        }
    }
    else
    {
        while (lat <= end)
        {
            TR[s++] = SR[lat++];
        }
    }

}

```

## 有n个数，两两组成二元组，差最小的有多少对呢

```
/****************************************
小Q今天在上厕所时想到了这个问题：有n个数，两两组成二元组，差最小的有多少对呢？差最大呢？


输入描述:

输入包含多组测试数据。

对于每组测试数据：

N - 本组测试数据有n个数

a1,a2...an - 需要计算的数据

保证:

1<=N<=100000,0<=ai<=INT_MAX.



输出描述:

对于每组数据，输出两个数，第一个数表示差最小的对数，第二个数表示差最大的对数。

输入例子:
6
45 12 45 32 5 6

输出例子:
1 2
******************************/

int jiecheng(int num)
{
    int sum = 0;
    while (num-- > 0)
    {
        sum *= num;
    }
    return sum;
}
int getPairs()
{
    int n;//整数个数
    cin >> n;
    int N = n;
    if (n < 2)
    {
        cout << 0 << " " << 0 << endl;
        return 0;
    }
    if (n == 2)
    {
        cout << 1 << " " << 1 << endl;
        return 0;
    }

    map<int, int>  nums;
    int num = 0;
    bool  flag = false;
    while (n-- > 0)
    {
        cin >> num;
        if (nums.find(num) == nums.end())
            nums[num] = 1;
        else
        {
            nums[num]++;
            flag = true;
        }
    }
    int maxDiff = 1;
    int minDiff = 0;
    if (nums.size() < 2)//说明所有的元素都相等，则最大差值对数==最小差值对数 等于个数的阶乘/2
    {
        int num = nums.begin()->second;

        maxDiff = jiecheng(num) / 2;
        minDiff = maxDiff;
        cout << minDiff << " " << maxDiff << endl;
        return 0;
    }
    else
    {
        maxDiff = nums.rbegin()->second*nums.begin()->second;


        if (flag)  //存在重复元素，则最小差值为 0 ，接下来统计各个相同字符的组合
        {
            map<int, int>::iterator  iter = nums.begin();

            while (iter != nums.end())
            {
                minDiff += jiecheng(iter->second) / 2;
                iter++;
            }
        }
        else
        { //不存在重复元素，则最小差值产生于相邻节点的差值，接下来统计相邻节点差值

            map<int, int>  hmap;
            minDiff = 65535;
            //find minDiff pairs
            map<int, int>::iterator iter = nums.begin();
            map<int, int>::iterator next = iter++;
            iter = nums.begin();
            while (  next != nums.end())
            {
             //   next = iter++;
                if (minDiff >= next->first - iter->first)
                {
                    minDiff = next->first - iter->first;
                    if (hmap.find(minDiff) == hmap.end())
                        hmap[minDiff] = 1;
                    else
                        hmap[minDiff]++;
                }
                iter = next;
                next++;
            }

            minDiff = hmap.begin()->second;
        }

        cout << minDiff << " " << maxDiff << endl;

    }
       

    return 0;


}



void compute_overlay(const string& pattern)
{
    const int pattern_length = pattern.size();
    int *overlay_function = new int[pattern_length];
    int index;
    overlay_function[0] = -1;
    for (int i = 1; i < pattern_length; ++i)
    {
        index = overlay_function[i - 1];
        //store previous fail position k to index;

        while(index >= 0 && pattern[i] != pattern[index + 1])
        {
            index = overlay_function[index];
        }
        if (pattern[i] == pattern[index + 1])
        {
            overlay_function[i] = index + 1;
        }
        else
        {
            overlay_function[i] = -1;
        }
    }
    for (int i = 0; i < pattern_length; ++i)
    {
        cout << overlay_function[i]<<" ";
    }
    cout << endl;
    delete[] overlay_function;
}



void getNext(string pat)
{
    int next[100];
    memset(next, 0, 100);
    int i = 1;
    int j = 0;
    next[0] = 0;
    while (i<pat.size())
    {
        if (j==0||pat[i] == pat[j])
        {
            i++;
            j++;
            if (pat[i]!=pat[j])
            {
                next[i] = j;
            }
            else
            {
                next[i] = next[j];
            }

        }
        else
        {
            j = next[j];
        }


    }
    for (int ii = 0; ii < pat.size();ii++)
    {
        cout << next[ii] << " ";
    }
    cout << endl;
}

```

## KMP
```
//KMP
void get_nextval(char const* ptrn, int plen, int* nextval)    
{    
    int i = 0;     
    nextval[i] = -1;    
    int j = -1;    
    while( i < plen-1 )    
    {    
        if( j == -1 || ptrn[i] == ptrn[j] )   //循环的if部分    
        {    
            ++i;    
            ++j;    
            //修正的地方就发生下面这4行    
            if( ptrn[i] != ptrn[j] ) //++i，++j之后，再次判断ptrn[i]与ptrn[j]的关系    
                nextval[i] = j;      //之前的错误解法就在于整个判断只有这一句。    
            else    
                nextval[i] = nextval[j];    
        }    
        else                                 //循环的else部分    
            j = nextval[j];    
    }    
}    
int kmp_search(char const* src, int slen, char const* patn, int plen, int const* nextval, int pos)    
{    
    int i = pos;    
    int j = 0;    
    while ( i < slen && j < plen )    
    {    
        if( j == -1 || src[i] == patn[j] )    
        {    
            ++i;    
            ++j;    
        }    
        else    
        {    
            j = nextval[j];              
            //当匹配失败的时候直接用p[j_next]与s[i]比较，    
            //下面阐述怎么求这个值，即匹配失效后下一次匹配的位置    
        }    
    }    
    if( j >= plen )    
        return i-plen;    
    else    
        return -1;    
}    



```
## 二叉树中任意两个节点间的最大距离

```


/********************************************
39.（树、图、算法）
网易有道笔试：
(1).
求一个二叉树中任意两个节点间的最大距离，
两个节点的距离的定义是  这两个节点间边的个数，
比如某个孩子节点和父节点间的距离是 1，和相邻兄弟节点间的距离是 2，优化时间空间复
杂度。
***********************************************/
struct BinaryTreeNode  
{  
    char m_chValue;  
    BinaryTreeNode* m_pLeft;  
    BinaryTreeNode* m_pRight;  
};  
  
struct Result  
{  
    int nMaxDistance;  
    int nMaxDepth;  
};  
  
Result GetMaxDistance(BinaryTreeNode *& pRoot)  
{  
    if (!pRoot)  
    {  
        Result empty={-1,-1};  
        return empty;  
    }  
    Result leftResult = GetMaxDistance(pRoot->m_pLeft);  
    Result rightResult = GetMaxDistance(pRoot->m_pRight);  
<span style="white-space:pre">  </span>  
    Result result;  
    result.nMaxDepth = 1+ max(leftResult.nMaxDepth, rightResult.nMaxDepth);  
    result.nMaxDistance =max( max(leftResult.nMaxDistance, rightResult.nMaxDistance), (leftResult.nMaxDepth+rightResult.nMaxDepth+2) );  
  
    return result;  
}  

//二叉树任意两个节点公共祖先
　struct BinTreeNode;

　　typedef struct BinTreeNode * PBinTreeNode;

　　struct BinTreeNode

　　{

　　　　char info;

　　　　PBinTreeNode llink;

　　　　PBinTreeNode rlink;

　　};

　　typedef struct BinTreeNode *BinTree;

　　

　　typedef struct

　　{

　　　　PBinTreeNode ptr;

　　　　int tag;

　　}Elem;

　　struct SeqStack

　　{

　　　　Elem s[MAXNUM];

　　　　int t;

　　};

　　typedef struct SeqStack * PSeqStack;


　　PBinTreeNode ancestor(BinTree t,PBinTreeNode p,PBinTreeNode q)

　　{

　　　　Elem stnode;

　　　　PSeqStack st;

　　　　PBinTreeNode pbnode,pstring[MAXNUM],qstring[MAXNUM];　　//分别用于存放p和q的祖先

　　　　char continueflag;

　　　　int i,findflag = 0;

　　　　if(t == NULL)

　　　　　　return NULL;

　　　　st = createEmptyStack();

　　　　pbnode = t;

　　　　do

　　　　{

　　　　　　while(pbnode != NULL)

　　　　　　{

　　　　　　　　stnode.ptr = pbnode;

　　　　　　　　stnode.tag = 1;

　　　　　　　　push(st,stnode);

　　　　　　　　pbnode = pbnode->llink;

　　　　　　}

　　　　　　while(!isEmptyStack(st))

　　　　　　{

　　　　　　　　stnode = pop(st);

　　　　　　　　pbnode = stnode.ptr;

　　　　　　　　if(stnode.tag == 1)

　　　　　　　　{

　　　　　　　　　　stnode.ptr = 2;

　　　　　　　　　　push(st,stnode);

　　　　　　　　　　pbnode = pbnode->rlink;

　　　　　　　　　　break;

　　　　　　　　}

　　　　　　　　else

　　　　　　　　{

　　　　　　　　　　if(pbnode == p)

　　　　　　　　　　{

　　　　　　　　　　　　for(i = 0;i < st->s[i];i++)

　　　　　　　　　　　　　　pstring[i] = st->s[i].ptr;　　　　//保存p的祖先

　　　　　　　　　　　　ptstring[i] =NULL;

　　　　　　　　　　　　findflag++;　　　　　　　　　　　　 //找到一个祖先

　　　　　　　　　　}

　　　　　　　　　　if(pbnode == q)

　　　　　　　　　　{

　　　　　　　　　　　　for(i = 0;i < st->s[i];i++)

　　　　　　　　　　　　　　qstring[i] = st->s[i].ptr;　　　　//保存p的祖先

　　　　　　　　　　　　qtstring[i] =NULL;

　　　　　　　　　　　　findflag++;　　　　　　　　　　　　 //找到一个祖先

　　　　　　　　　　}

　　　　　　　　　　if(findflag == 2)

　　　　　　　　　　{

　　　　　　　　　　　　i = 0;

　　　　　　　　　　　　while((pstring[i] == qstring[i]) && (pstring[i] != NULL))

　　　　　　　　　　　　　　i++;

　　　　　　　　　　　　if(pstring[i] == NULL)　　

　　　　　　　　　　　　{

　　　　　　　　　　　　　　free(st);

　　　　　　　　　　　　　　return pstring[i-1];

　　　　　　　　　　　　}

　　　　　　　　　　　　else

　　　　　　　　　　　　{

　　　　　　　　　　　　　　free(st);

　　　　　　　　　　　　　　return qstring[i-1];

　　　　　　　　　　　　}

　　　　　　　　　　}

　　　　　　　　}

　　　　　　}

　　　　}while(!isEmptyStack(st));

　　　　return NULL;

　　}
```
