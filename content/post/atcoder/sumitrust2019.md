---
title: "Atcoder Sumitomo Mitsui Trust Bank Programming Contest 2019"
date: 2019-12-04T17:14:56+08:00
draft: false
lastmod: 2019-12-04T17:14:56+08:00
tags: ["math", "knapsack", "enumeration"]
categories: ["Atcoder"]
---

[題目](https://atcoder.jp/contests/sumitrust2019/tasks)

<!--more-->

# [C - 100 to 105](https://atcoder.jp/contests/sumitrust2019/tasks/sumitb2019_c)

無限背包

## AC Code

```c++
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
typedef pair<int, int> ii;

const int N = 1000001;
bool dp[N];

int main()
{
    memset(dp, 0, sizeof(N));
    dp[0] = true;

    for (int i = 100; i <= 105; i++) {
        for (int j = 0; j < N; j++) {
            if (dp[j] && j + i < N)
                dp[j + i] = true;
        }
    }

    int x;
    scanf("%d", &x);
    printf("%d\n", dp[x]);

    return 0;
}
```

# [D - Lucky PIN](https://atcoder.jp/contests/sumitrust2019/tasks/sumitb2019_d)

如果想要 $O(n^3)$ 枚舉所有可能組合會超時。

但是，答案只有 1000 種，那我們就改成枚舉所有答案，並檢查是不是能組合出來就好啦！

## AC Code

```c++
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
typedef pair<int, int> ii;

int main()
{
    int n;
    scanf("%d", &n);

    char inp[n + 3];
    scanf("%s", inp);

    int ans = 0;
    for (int i = 0; i < 10; i++)
        for (int j = 0; j < 10; j++)
            for (int k = 0; k < 10; k++) {
                char num[3];
                num[0] = i + '0';
                num[1] = j + '0';
                num[2] = k + '0';

                int cnt = 0;
                for (int p = 0; p < n; p++) {
                    if (inp[p] == num[cnt]) {
                        cnt++;
                        if (cnt == 3)
                            break;
                    }
                }

                if (cnt == 3)
                    ans++;
            }
    printf("%d\n", ans);

    return 0;
}

```

# [E - Colorful Hats 2](https://atcoder.jp/contests/sumitrust2019/tasks/sumitb2019_e)

我們對於目前有幾個人有相同顏色這件事情，進行計數。

假設我們有個計數數列 $0, 1, 2$，那表示說，在現在這個人之前，分別可能有 0, 1, 2 個人，帽子的顏色可能跟他一樣。

## AC Code

```c++
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
typedef pair<int, int> ii;

const ll MOD = 1000000007;

int main()
{
    int n;
    scanf("%d", &n);

    int cnt[3] = {0}; // print the cnt out to see how it works
    ll ans = 1;
    for (int i = 0; i < n; i++) {
        int num;
        scanf("%d", &num);

        int tmp = 0;
        for (int j = 0; j < 3; j++) {
            if (cnt[j] == num)
                tmp++;
        }

        ans = (ans * tmp) % MOD;

        for (int j = 0; j < 3; j++)
            if (cnt[j] == num) {
                cnt[j]++;
                break;
            }
    }

    printf("%lld\n", ans);

    return 0;
}

```