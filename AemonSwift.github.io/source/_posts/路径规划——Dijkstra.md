---
title: 路径规划——Dijkstra
date: 2019-10-24 15:03:06
tags: 
- 算法
- 路径规划
categories:
- 算法
- 路径规划
toc: true
---
# 思想
主要用来求解：从起始点到其他所有点的最短路径。该算法采用了贪心的思想。
思想如下：A到B可以有多个中转站集合U，如何选择中转站？本算法选择最低的成本的中转站，即将C1加入到需要走的中转站集合S中，在目前集合S情况下，得到了A到所有各站的成本，在此成本基础上选择最低的成本加入到需要走的中转站集合S中，重复上述操作。
# 算法步骤
1. 初始化时，S只含有源节点；
2. 从U中选取一个距离v最小的顶点k加入S中（该选定的距离就是v到k的最短路径长度）；
3. 以k为新考虑的中间点，修改U中各顶点的距离；若从源节点v到顶点u的距离（经过顶点k）比原来距离（不经过顶点k）短，则修改顶点u的距离值，修改后的距离值是顶点k的距离加上k到u的距离；
4. 重复步骤2和3，直到所有顶点都包含在S中。
## 例子
![无向图](https://cdn.jsdelivr.net/gh/AemonSwift/AemonSwift@latest/AemonSwift.github.io/images/directpicture.jpg)
从A开始出发，到其他所有点的最短距离和路径
步骤|描述
:-:|:-:
1|初始化距离$dis=[0,6,3,\infty,\infty,\infty,\infty]$，此时S={A},U={B,C,D,E,F}
2|排除S中的点，寻找dis中最小距离的点C,此时距离为$dis=[0,5,3,6,7,\infty]$，此时S={A,C},U={B,D,E,F}
3|排除S中的点，寻找dis中最小距离的点B，此时距离变为$dis=[0,5,3,6,7,\infty]$，此时S={A,C,B},U={D,E,F}
4|排除S中的点，寻找dis中最小距离的点D，此时距离变为$dis=[0,5,3,6,7,9]$，此时S={A,C,B,D},U={E,F}
5|排除S中的点，寻找dis中最小距离的点E，此时距离变为$dis=[0,5,3,6,7,9]$，此时S={A,C,B,D,E},U={F}
6|排除S中的点，寻找dis中最小距离的点F，此时距离变为$dis=[0,5,3,6,7,9]$，此时S={A,C,B,D,F},U={}

# 算法实现
## 头文件描述
```c++
// Dijkstra.h
# pragma once //pragma once是一个比较常用的C/C++声明，只要在头文件的最开始加入这条杂注，就能够保证头文件只被编译一次。
# include <iostream>
# include <string>
#include <sstream>
using namespace std;


struct Dis{
	string path;
	int value;
	bool visit;
	int prePoint; //记录到当前节点的上一个节点是谁
	Dis(){
		visit=false;
		value=0;
		path="";
		minPath=nullptr;
	}
};

class GraphDG{
private:
	int pointNum;
	int edge;
	int **adjacentMat；
	Dis*dis;
	string intToString(int target);
	bool checkEdgeValue(int start,int end,int weight);
public:
	GraphDG(int pointNum,int edge);
	~GraphDG();
	void createGraph();
	void print();
	void resloveMinPath(int begin);
	void printSearchPath(int begin);
	void printMinPath(int begin,int end);
};
```
## 功能实现
```c++
# include "Dijkstra.h"

const INT_MAX=2^31-1;
GraphDG::GraphDG(int pointNum,int edge){
	this->pointNum=pointNum;
	this->edge=edge;
	adjacentMat=new int* [this->pointNum];
	dis=new Dis[this->pointNum];
	for(int i=0;i<this->pointNum;i++){
		adjacentMat=new int[this->pointNum];
		for int(j=0;j<this->pointNum;j++){
			adjacentMat[i][j]=INT_MAX; //开始赋值无穷大
		}
	}
}

GraphDG::~GraphDG() {
	delete dis;
	for (int i=0;i<this->pointNum;i++){
		delete this->adjacentMat[i];
	}
	delete this->adjacentMat;
}

bool GraphDG::checkEdgeValue(int start,int end,int weight){
	if (start<1||end<1||start>this->pointNum||end>this->pointNum||weight<0){
		return false;
	}
	return true;
}

void GraphDG::createGraph(){
	cout << "请输入每条边的起点和终点（顶点编号从1开始）以及其权重" << endl;
	int start,end,weight,count=0;
	while(count!=this->edge){
		cin>>start>>end>>weight;
		while(!this->checkEdgeValue(start,end,weight)){
			cout << "输入的边的信息不合法，请重新输入" << endl;
            cin >> start >> end >> weight;
		}
		adjacentMat[start-1][end-1]=weight;
		// adjacentMat[end-1][start-1]=weight 加上这句为无向边
		count++;
	}
}

void GraphDG::print(int begin){
	cout << "图的邻接矩阵为：" << endl;
	for (int row=0;row<this->pointNum;col++){
		for(int col=0;col<this->pointNum;i++;col++){
			if (adjacentMat[row][col]==INT_MAX){
				cout<<"infinity"；
			}else{
				cout<<adjacentMat[row][col];
			}
		}
		cout<<endl;
	}
	cout<<endl;
}

void GraphDG::printSearchPath(int begin){
	string str;
	str="v"+intToString(begin);
	cout << "以"<<str<<"为起点的图的最短路径为：" << endl;
	for (int i=0;i!=this->pointNum;i++){
		if(dis[i].value!=INT_MAX){
			cout << dis[i].path << "=" << dis[i].value << endl;
		}else{
			cout << dis[i].path << "是无最短路径的" << endl;
		}
	}
}

void GraphDG::printMinpath(int begin,int end){
	string str;
	int prePoint=dis[end].prePoint;
	while(prePoint>=0){
		proPoint=dis[prePoint].prePoint;
		if (prePoint==proPoint){
			break;
		}
		str=intToString(prePoint+1)+" "+str;
		prePoint=proPoint;
	}
	cout<<str;
}

void GraphDG::Dijkstra(int begin){
	for (int i=0;i<this->pointNum;i++){
		dis[i].path="v"+intToString(begin)+"-->v"+intToString(i+1);
		dis[i].value=adjacentMat[begin-1][i];
		dis[i].prePoint=begin;
	}
	dis[begin-1].value=0;
	dis[begin-1].visit=true;
	for (int count=1;count<this->pointNum;count++){
		//找加入的最小值对应的下标
		int temp=0,min=INT_MAX;
		for (int i=0;i<this->pointNum;i++){
			if(!dis[i].visit&&dis[i].value<INT_MAX){
				min=dis[i].value;
				temp=i
			}
		}
		dis[temp].visit=true;
		// 计算剩余点的最短路径
		for (int i=0;i<this->pointNum;i++>){
			//注意这里的条件adjacentMat[temp][i]!=INT_MAX必须加，不然会出现溢出，从而造成程序异常
			if(!dis[i].visit&&adjacentMat[temp][i]!=INT_MAX&&dis[temp].value+adjacentMat[temp][i]<dis[i].value){
				dis[i].value=dis[temp].value+adjacentMat[temp][i];
				dis[i].path=dis[temp].path+"-->v"+intToString(i+1);
				dis[i].prePoint=temp;
			}
		}

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
#include "Dijkstra.h"

bool check(int pointNum,int edge){
	if(pointNum<=1||edge<=0||(pointNum-1)*pointNum/2<edge){
		return false;
	}
	return true;
}

int main(){
	int pointNum,edge;
	cout << "输入图的顶点个数和边的条数：" << endl;
    cin >> pointNum >> edge;
    while (!check(vexnum, edge)) {
        cout << "输入的数值不合法，请重新输入" << endl;
        cin >> pointNum >> edge;
    }
    GraphDG grpah(pointNum,edge);
    graph.createGraph();
    graph.print();
    graph.Dijkstra();
    graph.printSearchPath();
    graph.printMinPath();
    system("pause");
    return 0;
}
//输入参数
// 6 8
// 1 3 10
// 1 5 30
// 1 6 100
// 2 3 5
// 3 4 50
// 4 6 10
// 5 6 60
// 5 4 20
```

# 算法缺陷
若权重为负边的时候，此算法失效。