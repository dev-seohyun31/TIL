# Implement Queue using Stacks [#232](https://leetcode.com/problems/implement-queue-using-stacks/description/)
## 문제
스택을 이용해서 다음 연산을 지원하는 큐를 구현하라
* 구현 매서드: `push(x), pop(), peek(), empty()`

## 전략
### 1. 스택을 2개 사용: `input` 스택에는 push만, `output` 스택에는 pop/peek 전용
```java
public class MyQueue {
  Deque<Integer> input = new ArrayDeque();
  Deque<Integer> output = new ArrayDeque();


  public void push(int x) {
    input.push(x);
  }

  public int peek() {
    move();
    return output.peek();
  }

  public int pop() {
    move();
    return output.pop();
  }

  public boolean empty() {
    return input.isEmpty() && output.isEmpty();
  }


  private void move() {
    if (output.isEmpty()) {
      while (!input.isEmpty()) {
        output.push(input.pop());
      }
    }
  }
}
```

* 시간 복잡도: 옮기는 비용이 분산되어 `O(1)`
* 공간 복잡도: 두 스택에 전체 원소를 저장하므로, `O(n)`

# Circular Queue [#622](https://leetcode.com/problems/design-circular-queue/description/)
## 문제
원형 큐를 디자인해라. 큐가 비어있다면 -1을 리턴한다. 
* 생성자: `new MyCircularQueue(5)`
* 매서드: `enQueue(10), deQueue(), Rear(), isFull(), Front()`
## 전략: 투포인터
```java
public class MyCircularQueue {
  int[] q;
  int head = 0, tail = -1, size = 0;

  public MyCircularQueue(int k) {
    this.q = new int[k]
  }

  public boolean enQueue(int x){ if(isFull()) return false; q[tail]=x; tail=(tail+1)%q.length; size++; return true; }
  public boolean deQueue(){ if(isEmpty()) return false; head=(head+1)%q.length; size--; return true; }
  public int Front(){ return isEmpty()? -1 : q[head]; }
  public int Rear(){ return isEmpty()? -1 : q[(tail-1+q.length)%q.length]; }
  public boolean isEmpty(){ return size==0; }
  public boolean isFull(){ return size==q.length; }
}
```
