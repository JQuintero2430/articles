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

If we want to find our addends for a specific number, our first reasoning must be releated about how to travers our elements list, in a first instance the answer is simple, we must to do a for loop for every posible result, later we evaluate if we have the target into our results list, but this will generate an algorithm who have two nested loops, which means that every number must evaluate himself with every other number into the list, giving as result an algoritmic complexity of O(n.n). Therefore we have to abord the problem using another approach

Given that we want to travers the list only once to know the value for every position the best option is store this values into a key value structure, in our example the best option is a HashMap which store two integers.

class Solution {

    public static int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> numToIndex = new HashMap<>();
    }
}

Now then, after clarify this, we should solve the problem related with get the missing addend to obtain our target, for this we'll use a simple mathematic reasoning. 
I have the first addend which is the number into my list, and I have the result whish is my target, from this point I just need to clear a simple equation that looks like this

listNumber + X = target

Where listNumber and target are known parameter, so I move listNumber to the other side of equation and chenge his sign, giving us as result the follow equation

X = target - listNumber

Solved the topic of ¿how to find the second addend? we start the travers over our list using a unique for loop. Now we have to evaluate every single element and we have to question our self ¿when I use my formula for this number which is into my list, the result is into my hashMap as a key? if the answer is no I have to add the value and the index of my list in my HashMap, but if already exist that means we find the missing addend for our target, so we have to retrive the current index and the value of the key that make match in our first question.
This results in an algorithm with an algoritmic complexity of O(n).

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
    }
}

In our last step we take in consideration the posiblility that we don't have any combination of addends which generate our target, in this case we can add an exception or a message expressing this idea

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

Resume of the algorithm:

 - 1. Create a hash map to store the numbers and their indices.
 - 2. Iterate through the array, for each number calculate its complement.
 - 3. If the complement exists in the map, return the pair of indices.
 - 4. If no pair is found, throw an exception.
