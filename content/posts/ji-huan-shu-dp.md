---
title: "基环树DP"
date: 2017-12-25T19:18:23+08:00
draft: false
---

{{< admonition note "note" true >}}

从博客园迁移，复习一下以前学的内容

{{< /admonition >}}

一般的树形DP都是在树上进行，即有N个点和N-1条边。可是有一类问题是N个点和N条边，这样一定会有环，这样的图就叫基环树。

对于这类题，我们可以进行一次dfs，找出环，然后选择环上的任意一条边删除。这样就能变成N个点，N-1条边的树，
可以很容易地使用树形DP解决。

假设将要删去的环上的边为E，边上两端点为u，v。然后分别以u和v为树根做一次DP，去最优值就可以了。

Tips：在存图是，可将cnt的初始值设为1，即第一条边的编号设为2，然后用类似于网络流中存储反向边的方法来做。

模板题：

BZOJ 1040[ZJOI2008]骑士

```cpp
/*
    Author:Wdvxdr
    Problem:Bzoj 1040
    Time:2017/12/25
*/
#include<bits/stdc++.h>
using namespace std;
const int maxn = 1e6+10;

struct node{int v,nxt;}e[maxn<<1];
int head[maxn],cnt=1,a[maxn],n,Rtree,Ltree,DelEdge;
bool vis[maxn];
long long f[maxn][2];

inline void add(int u,int v)
{
    e[++cnt] = (node){v,head[u]};
    head[u]=cnt;
}

void dfs(int u,int fa)
{
    vis[u] = 1;
    for(int i=head[u];i;i=e[i].nxt)
    {
        int v = e[i].v;
        if(v!=fa)
            if(!vis[v])
            {
                dfs(v,u);
            }
            else
                Rtree = u,Ltree = v,DelEdge=i;
    }
}

void dp(int u,int fa)
{
    int v;
    f[u][0]=0;f[u][1]=a[u];
    for(int i=head[u];i;i=e[i].nxt)
    {
        if(i==DelEdge||i==(DelEdge^1))    continue;
        v = e[i].v;
        if(v!=fa)
        {
            dp(v,u);
            f[u][0] += max(f[v][0],f[v][1]);
            f[u][1] += f[v][0];
        }
    }
}

int main()
{
    int t;long long temp,ans=0;
    scanf("%d",&n);
    for(int i=1;i<=n;i++)
    {
        scanf("%d%d",&a[i],&t);
        add(t,i);add(i,t);
    }
    for(int i=1;i<=n;i++)
    {
        if(vis[i])    continue;
        dfs(i,0);
        dp(Rtree,0);
        temp = f[Rtree][0];
        dp(Ltree,0);
        temp = max(temp,f[Ltree][0]);
        ans += temp;
    }
    cout << ans;
    return 0;
}
```
