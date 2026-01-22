Definition for singly-linked list.
```java
public class ListNode {
    int val;
    ListNode next;
    ListNode() {}
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 }
```
# Reverse Liked List [#206](https://leetcode.com/problems/reverse-linked-list/description/)
## 문제 
연결 리스트를 뒤집기
* 입력: 1 -> 2 -> 3 -> 4 -> 5
* 출력: 5 -> 4 -> 3 -> 2 -> 1

## 전략
### 1. 반복: `prev, curr, next` 세 포인터로 링크를 한 칸씩 뒤집는다
```java
ListNode prev = null;
ListNode curr = head;

while (curr != null) {
  ListNode next = curr.next;
  curr.next = prev;
  prev = curr;
  curr = next;
}
return prev;
```
* 시간 복잡도
  * 반복문 내에서는 노드마다 포인터를 1번 재지정해야 하기 떄문에, 총 `O(n)`
* 공간 복잡도
  * 보조 포인터 3개이므로, `O(1)`
* 참고
  * `curr.next = prev; prev = curr; curr = next;` 순서가 중요합니다.
 
# Merge Two Sorted Lists [#21](https://leetcode.com/problems/merge-two-sorted-lists/description/)
## 문제
정렬되어 있는 두 연결 리스트를 합치기
* 입력: 1 -> 2 -> 5, 1 -> 3 -> 4
* 출력: 1 -> 1 -> 2 -> 3 -> 4 -> 5
## 전략
### 1. 반복: 더미 노드를 사용하여 tail 잇기
// 참조가 어떻게 되는건지 디버깅??
```java
ListNode dummy = new ListNode(0);
ListNode tail = dummy;

while (list1 != null && list2 != null) {
  if (list1.val <= list2.val) {
    tail.next = list1;
    list1 = list1.next;
  } else {
  tail.next = list2;
  list2 = list2.next;
  }
  tail = tail.next;
}

tail.next = (list1 != null) ? list1 : list2;
return dummy.next;
```
* 시간 복잡도
  * 두 리스트의 총 길이 m+n을 한 번씩 방문하여, `O(m + n)`
* 공간 복잡도
  * 기존의 노드들로 링크만 바꾸고 있기 때문에 `O(1)`
### 2. 재귀: 가장 작은 노드부터 하나씩 선택하고, 나머지 병합은 재귀로 위임
```java
ListNode mergeRecursive(ListNode list1, ListNode list2) {
  if (list1 == null) return list2;
  if (list2 == null) return list1;

  if (list1.val <= list2.val) {
    list1.next = mergeRecursive(list1.next, list2);
    return list1;
  } else {
    list2.next = mergeRecursive(list1, list2.next);
    return list2;
  }
}
```
* 시간 복잡도
  * 두 리스트를 한 번씩 방문하므로, `O(m + n)`
* 공간 복잡도
  * 추가로 사용하는 구조는 없지만, 호출 스택 공간이 입력 길이와 비례하여 `O(m + n)`
  * 노드 수가 매우 많으면 `StackOverflowError` 위험이 있음

# Palindrom Linked List [#234](https://leetcode.com/problems/palindrome-linked-list/description/)
## 문제
연결 리스트가 팰린드롬 구조인지 판별
* 입력: 1 -> 2 -> 3 -> 2 -> 1
* 출력: true

## 전략
### 1. 러너 기법 // 보완 필요 
```java
if (head == null || head.next == null) return true;

ListNode slow = head;
ListNode fast = head;
while (fast.next != null && fast.next.next != null) {
  slow = slow.next;
  fast = fast.next.next;
}

ListNode second = reverse(slow.next);
```

### 2. 스택
```java
Deque<Integer> stack = new ArrayDeque<>();
for (ListNode p = head; p != null; p = p.next) {
  stack.push(p.val);
}

for (ListNode p = head; p != null; p = p.next) {
  if (p.val != stack.pop()) {
    return false;
  }
}
return true;
```
* 시간 복잡도: `O(n)`
* 공간 복잡도
  * 스택에 새로 담아야 하기 때문에 `O(n)`
