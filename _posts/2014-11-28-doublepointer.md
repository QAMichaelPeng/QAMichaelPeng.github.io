---
title: double pointer
layout: post
---
Double pointer is a great tool to simplify certain programming tasks.

Let's look into a simple example: merge two sorted linked list by spicing together the nodes of the orignal lists without creating new nodes.

Below is the define of linked node:
{% highlight c++ linenos tabsize=4 %}
struct ListNode {
    int val;
    ListNode *next;
    ListNode(int x) : val(x), next(NULL) {}
};
{% endhighlight %}

The simplest way is use a pointer h to record the head of merged list, and use another pointer p to record the tail of merged results;

{% highlight c++ linenos tabsize=4 %}
ListNode *mergeTwoLists(ListNode *l1, ListNode *l2) {
    ListNode* h = NULL, *p = NULL;
    while (l1 && l2) {
        if (l1->val <= l2->val) {
            if (!h) {
                h = p = l1;
            }
            else {
                p->next = l1;
                p = l1;
            }
            l1 = l1->next;
        }
        else {
            if (!h) {
                h = p = l2;
            }
            else {
                p->next = l2;
                p = l2;
            }
            l2 = l2->next;
        }
    }
    if (!h) h = l1 ? l1 : l2;
    else p->next = l1 ? l1 : l2;
    return h;
}
{% endhighlight %}

hmm, too many nested if else in the 28 line solution. How to elimate the detect of NULL of head? One trick is to use a dummy node.

{% highlight c++ linenos tabsize=4 %}
ListNode *mergeTwoLists(ListNode *l1, ListNode *l2) {
    ListNode dummy(-1), *p = &dummy;
    for (; l1 != nullptr && l2 != nullptr; p = p->next) {
        if (l1->val > l2->val) {
            p->next = l2;
            l2 = l2->next;
        }
        else { 
            p->next = l1;
            l1 = l1->next;
        }
    }
    p->next = l1 != nullptr ? l1 : l2;
    return dummy.next;
}
{% endhighlight %}

Here we got a 15 line solution, 13 lines are elimated. Here a dummy node elemated the check of NULL for head and prev pointer. But how can we elimate the construct of a new dummy load if the cost is heavy? Double pointer may help.
{% highlight c++ linenos tabsize=4 %}
ListNode *mergeTwoLists(ListNode *l1, ListNode *l2) {
    ListNode* h = NULL, **p = &h;
    for (;l1&&l2;p=&((*p)->next)) {
        if (l1->val <= l2->val) {
            *p = l1;
            l1 = l1->next;
        }
        else {
            *p = l2;
            l2 = l2->next;
        }
    }
    *p = l1 ? l1 : l2;
    return h;
}
{% endhighlight %}

Here're some quizzes you can try with double pointer.

* [Remove Nth Node From End of List](https://oj.leetcode.com/problems/remove-nth-node-from-end-of-list/)
* [Insertion Sort List ](https://oj.leetcode.com/problems/insertion-sort-list/)
* [Reverse Linked List II](https://oj.leetcode.com/problems/reverse-linked-list-ii/)
* [Partition List](https://oj.leetcode.com/problems/partition-list/)
* [Merge Two Sorted Lists](https://oj.leetcode.com/problems/merge-two-sorted-lists/)
