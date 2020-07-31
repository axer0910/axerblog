---
title: Leetcode121. Best Time to Buy and Sell Stock最佳买卖股票时机
tag: 算法
date: 2019-04-23
updated: 2019-04-23
---

Say you have an array for which the ith element is the price of a given stock on day i.

If you were only permitted to complete at most one transaction (i.e., buy one and sell one share of the stock), design an algorithm to find the maximum profit.

Note that you cannot sell a stock before you buy one.

Input: [7,1,5,3,6,4]
Output: 5
Explanation: Buy on day 2 (price = 1) and sell on day 5 (price = 6), profit = 6-1 = 5.
             Not 7-1 = 6, as selling price needs to be larger than buying price.

Input: [7,6,4,3,1]
Output: 0
Explanation: In this case, no transaction is done, i.e. max profit = 0.

解法是一次遍历数组，第一个元素先作为最小值，后面每次遍历均更新最小值和当前元素减掉最小值的最大差值，遍历结束返回最大差值即可。
```Javascript
var maxProfit = function(prices) {
    let min = prices[0];
    let max = 0;
    for (let i = 0; i < prices.length; i ++) {
        if (prices[i] < min) min = prices[i];
        let val = prices[i] - min;
        if (val > 0) {
            if (val > max) max = val;
        }
    }
    return max;
};
```