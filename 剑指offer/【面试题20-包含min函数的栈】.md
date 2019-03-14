title: 【面试题20-包含min函数的栈】
tag: 剑指offer题解
---
剑指offer第二十题。
<!-- more -->

## 题目描述

定义栈的数据结构，请在该类型中实现一个能够得到栈中所含最小元素的min函数（时间复杂度应为O（1））。

## 解题思路

思路：利用一个辅助栈来存放最小值

栈 3，4，2，5，1

辅助栈 3，3，2，2，1

每入栈一次，就与辅助栈顶比较大小，如果小就入栈，如果大就入栈当前的辅助栈顶；

当出栈时，辅助栈也要出栈

这种做法可以保证辅助栈顶一定都当前栈的最小值

## 我的答案




```java
import java.util.Stack;

public class Solution {

    
    //存放元素
    Stack<Integer> stack1 = new Stack<Integer>();
    //存放当前stack1中的最小元素
    Stack<Integer> stack2 = new Stack<Integer>();
    
    //stack1直接塞，stack2要塞比栈顶小的元素，要不然就重新塞一下栈顶元素
    public void push(int node) {
        stack1.push(node);
        if(stack2.isEmpty() || stack2.peek() > node){
            stack2.push(node);
        }else{
            stack2.push(stack2.peek());
        }
        
    }
    
    //都要pop一下
    public void pop() throws Exception{
        if(stack1.isEmpty()){
           throw new Exception("no element valid"); 
        }
        stack1.pop();
        stack2.pop();
    }
    
    
    public int top(){
        if(stack1.isEmpty()){
           return 0;
        }
        return stack1.peek();
    }
    
    public int min(){
        if(stack2.isEmpty()){
           return 0;
        }
        return stack2.peek();
    }
}
```
