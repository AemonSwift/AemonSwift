---
title: 路径规划——Floyd
date: 2019-10-25 09:22:00
tags:
- 算法
- 路径规划
categories: 
- 算法
- 路径规划
toc: true
---

# 思想
$\color{red}{主要用来求解：}$任意两点的最短路径。该算法采用了动态规划的思想。
$\color{red}{思想如下：}$A到B，可以经历中转站得来降低成本；当考虑了所有的中转站的时候，则可以得到此图在A到B的最低成本。
$\color{red}{适用范围：}$解决多源路径问题。

# 算法步骤
1. 初始化边矩阵M；
2. 从U中选取顶点k加入S中，并将此元素从U中移除；
3. 以k为中转站，更新边矩阵M的信息；
4. 重复步骤2和3，直到所有顶点都包含在S中。
## 例子
![无向图](https://cdn.jsdelivr.net/gh/AemonSwift/AemonSwift@latest/AemonSwift.github.io/images/directpicture.jpg)

a. 初始化M矩阵,即经过A点的M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|6|3|$\infty$|$\infty$ |$\infty$
B|6|0|2|5| $\infty$|$\infty$
C|3|2|0|3|4| $\infty$
D|$\infty$|5|3|0|2 |3
E|$\infty$|$\infty$|4|2| 0|5
F|$\infty$|$\infty$|$\infty$|3| 5|0

b. 考虑经过B点，更新M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|6|3|11|$\infty$ |$\infty$
B|6|0|2|5| $\infty$|$\infty$
C|3|2|0|3|4| $\infty$
D|11|5|3|0|2 |3
E|$\infty$|$\infty$|4|2| 0|5
F|$\infty$|$\infty$|$\infty$|3| 5|0

c. 考虑经过C点，更新M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|5|3|6|7|$\infty$
B|5|0|2|5| 6|$\infty$
C|3|2|0|3|4| $\infty$
D|6|5|3|0|2 |3
E|7|6|4|2| 0|5
F|$\infty$|$\infty$|$\infty$|3| 5|0

d. 考虑经过D点，更新M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|5|3|6|7|9
B|5|0|2|5| 6|8
C|3|2|0|3|4|6
D|6|5|3|0|2 |3
E|7|6|4|2| 0|5
F|9|8|6|3| 5|0

e. 考虑经过E点，更新M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|5|3|6|7|9
B|5|0|2|5| 6|8
C|3|2|0|3|4|6
D|6|5|3|0|2 |3
E|7|6|4|2| 0|5
F|9|8|6|3| 5|0

f. 考虑经过F点，更新M矩阵

*|A|B|C|D|E|F
:-:|:-:|:-:|:-:|:-:|:-:|:-:
A|0|5|3|6|7|9
B|5|0|2|5| 6|8
C|3|2|0|3|4|6
D|6|5|3|0|2 |3
E|7|6|4|2| 0|5
F|9|8|6|3| 5|0

# 算法实现
## 头文件描述
```C++
pragma once

#include <iostream>
#include <string>
#include <stringstream>
using namespace std;

class GraphDG{
private:
	int pointSum;
	int edge;
	int **adjacentMat；
	int **dis;
	int **path;
	string intToString(int target)
	bool checkEdgeValue(int start,int end,int weight);
public:
	GraphDG(int pointSum,int edge);
	~GraphDG();
	void createGraph(int);
	void print();
	void Floyd();
	void printMinPath();
};
```
## 功能实现
```c++
# include "Floyd.h"

const INT_MAX=2^32-1;

GraphDG::GraphDG(int pointSum,int edge){
	this->pointSum=pointSum;
	this->edge=edge;
	this->adjacentMat=new int*[this->pointSum];
	this->dis=new int*[this->pointSum];
	this->path=new int*[this->pointSum];
	for(int i=0;i<this->pointSum;i++){
		adjacentMat[i]=new int[this->pointSum];
		dis[i]=new int[this->pointSum];
		path[i]=new int[this->pointSum];
		for (int j=0;j<this->pointSum;j++){
			adjacentMat[i][j]=INT_MAX;
		}
	}
}

GraphDG::~GraphDG(){
	for (int i=0;i<this->pointSum;i++){
		delete adjacentMat[i];
		delete dis[i];
		delete path[i];
	}
	delete adjacentMat;
	delete dis;
	delete path;
}

bool GraphDG::checkEdgeValue(int start,int end,int weight){
	if (start<1||end<1||start>end||end>pointSum||weight<0){
		return false;
	}
	return true;
}

void GraphDG::createGraph(int kind){
	cout << "请输入每条边的起点和终点（顶点编号从1开始）以及其权重" << endl;
	int start,end,weight;
	for (int i=0;i<this->edge;i++>){
		cin>>start>>end>>weight;
		while(!this->checkEdgeValue(start,end,weight)){
			cout << "输入的边的信息不合法，请重新输入" << endl;
            cin >> start >> end >> weight;
		}
		adjacentMat[start-1][end-1]=weight;
		//变成无向图
		if (kind==2){
			adjacentMat[end-1][start-1]=weight;
		}
	}

}

void GraphDG::print(){
	cout << "图的邻接矩阵为：" << endl;
	for (int row=0;row<this->pointSum;row++>){
		for (int col=0;col<this->pointSum;col++){
			if adjacentMat[row][col]==INT_MAX{
				cout<<"i ";
			}else{
				cout<<adjacentMat[row][col]<<" ";
			}
		}
		cout<<endl;
	}
	cout <<endl;
}
//思想在于：A到B，可以经历中转站得来降低成本；当考虑了所有的中转站的时候，则可以得到此图在A到B的最低成本。——动态规划思想
void GraphDG::Floyd(){
	for (int row=0;row<this->pointSum;row++){
		for (int col=0;col<this->pointSum;col++){
			this->dis[row][col]=this->adjacentMat[row][col];
			this->path[row][col]=col;
		}
	}
	for (int temp =0;temp<this->pointSum;temp++){ //temp为中转站
		for (int i=0;i<this->pointSum;i++){
			for(int j=0;j<this->pointSum;j++){
				select=(dis[row][temp] == INT_MAX || dis[temp][col] == INT_MAX) ? INT_MAX : (dis[row][temp] + dis[temp][col]);
				if(this->dis[i][j]>select){
					this->dis[i][j]=select;
					this->path[i][j]=this->path[i][temp];
				}
			}
		}
	}
}

void GraphDG::printMinPath(){
	cout << "各个顶点对的最短路径：" << endl;
	for (int row=0;row<this->pointSum;row++){
		for (int col=0;col<this->pointSum;col++){
			cout<<"v"<<intToString(row+1)<<"--"<<"v"<<intToString(col+1)<<"weight:"
			<<this->dis[row][col]<<"path:"<<"v"<<intToString(row+1);
			temp=path[row][col];
			while(temp!=col){
				cout<<"-->"<<"v"<<intToString(temp+1);
				temp=path[temp][col];
			}
			cout<<"-->"<<"v"<<intToString(col+1)<<endl;
		}
		cout<<endl;
	}
}

string GraphDG::intToString(int target){
	stringstream ss;
	ss<<target;
	return ss>>; 
}
```
## 测试函数
```c++
#include "Floyd.h"

bool check(int pointSum,int edge){
	if (pointSum<=1||edge<=0||(pointSum-1)*pointSum<edge){
		return false;
	}
	return true;
}

int main(){
	int pointSum,edge,kind;
	cout << "输入图的种类：1代表有向图，2代表无向图" << endl;
	cin>>kind;
	while(true){
		if (kind==1||kind==2){
			break;
		}else{
			cout << "输入的图的种类编号不合法，请重新输入：1代表有向图，2代表无向图" << endl;
            cin >> kind;
		}
	}
	cout << "输入图的顶点个数和边的条数：" << endl;
    cin >> pointSum >> edge;
    while (!check(pointSum, edge)) {
        cout << "输入的数值不合法，请重新输入" << endl;
        cin >> pointSum >> edge;
    }
    GraphDG graph(pointSum,edge);
    graph.createGraph(kind);
    graph.print();
    graph.Floy();
    graph.printMinPath();
    system("pause");
    return 0
}

//输入参数
// 2
// 7 12
// 1 2 12
// 1 6 16
// 1 7 14
// 2 3 10
// 2 6 7
// 3 4 3
// 3 5 5
// 3 6 6
// 4 5 4
// 5 6 2
// 5 7 8
// 6 7 9 
```
# 算法缺陷
可以求有负权值的边，但是不能有负回路。