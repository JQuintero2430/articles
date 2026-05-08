# How To Solve The Two Sum Problem with Java

The first step is to understand the problem. If you visit the LeetCode problem section, you will find the statement of the problem, which gives the following description:

> **Given an array of integers `nums` and an integer `target`, return the indices of the two numbers such that they add up to `target`.**
>
> You may assume that each input would have exactly one solution, and you may not use the same element twice.  
> You can return the answer in any order.

### Examples

**Example 1:**  
Input: `nums = [2,7,11,15], target = 9`  
Output: `[0,1]`  
Explanation: Because `nums[0] + nums[1] == 9`, we return `[0,1]`.

**Example 2:**  
Input: `nums = [3,2,4], target = 6`  
Output: `[1,2]`

**Example 3:**  
Input: `nums = [3,3], target = 6`  
Output: `[0,1]`

### Constraints
- `2 <= nums.length <= 10⁴`  
- `-10⁹ <= nums[i] <= 10⁹`  
- `-10⁹ <= target <= 10⁹`  
- Only one valid answer exists.

**Follow-up:** Can you come up with an algorithm with time complexity less than O(n²)?

---

## Step 1: Initial Approach

If we want to find the addends for a specific number, our first reasoning must be related to how we traverse the list of elements. At first, the answer seems simple: we can use a nested loop to check all possible pairs. However, this generates an algorithm with two nested loops, meaning each number is compared with every other number in the list, leading to a time complexity of O(n²). Therefore, we need a different approach.

---

## Step 2: Using a HashMap

Since we want to traverse the list only once while keeping track of values, the best option is to store these values in a key-value structure. In this case, a `HashMap` works perfectly, storing numbers as keys and their indices as values.

```java
class Solution {
    public static int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> numToIndex = new HashMap<>();
    }
}
```

## Step 3: Finding the Missing Addend

Now, we need to determine how to find the missing addend that complements our current number to reach the target. The reasoning is simple:

listNumber + X = target


Rearranging the equation:

X = target - listNumber

## Step 4: Implementing the Algorithm

We traverse the list with a single loop. For each number, we calculate its complement and check if it already exists in the HashMap.

If it does, we have found the two indices.

If it doesn’t, we store the current number and its index in the HashMap.

This results in an algorithm with O(n) time complexity.

class Solution {
    public static int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> numToIndex = new HashMap<>();

        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];

            if (numToIndex.containsKey(complement)) {
                return new int[]{numToIndex.get(complement), i};
            }

            numToIndex.put(nums[i], i);
        }
        throw new IllegalArgumentException("No solution found");
    }
}

Summary of the Algorithm

1. Create a HashMap to store the numbers and their indices.

2. Iterate through the array and calculate the complement for each number.

3. If the complement exists in the map, return the pair of indices.

4. If no pair is found, throw an exception.
