###Leetcode 记录2：two sum

 Given nums = [2, 7, 11, 15], target = 9,
 
 Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
先用target减掉数组里每个元素，最后再寻找剩下数组里有没有target减掉的那个数即可。
```
var twoSum = function(nums, target) {
    for (let index in nums) {
        let num = nums[index];
        let tmpReplacedValue = num;
        nums[index] = false;
        let endTarget = target - num;
        let endPos = nums.indexOf(endTarget);
        if (endPos >= 0) {
            return [index, endPos];
        }
        nums[index] = tmpReplacedValue;
    }
};
```
优化后，把减数存到一个空间里，nums每次遍历只需要查找目标减数在不在这个空间里，减少`nums.indexOf(endTarget)`这步查找的次数。
```
var twoSum = function(nums, target) {
    // 把所有需要寻找的减数存起来
    let res = {};
    for (let index in nums) {
        let num = nums[index];
        let endTarget = target - num;
        if (res.hasOwnProperty(num)) {
            return [res[num], index]
        }
        res[endTarget] = index;
    }
};
```
