# Top K Frequent Elements [#347](https://leetcode.com/problems/top-k-frequent-elements/description/)
## 문제
빈도 순으로 k개의 엘리먼트를 추출
* 입력: `nums = [1, 1, 1, 2, 2, 3, 4], k = 2`
* 출력: `[1, 2]`

## 전략
### 1. 해시맵
```java
// 빈도 계산
Map<Integer, Integer> freq = new HashMap<>();
for (int n : nums) {
  freq.put(n, freq.getOrDefault(n, 0) + 1);
}

// 최소 힙
PriorityQueue<Map.Entry<Integer, Integer>> pq = new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));

// 힙크기 유지
for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
  pq.offer(e);
  if (pq.size() > k) pq.poll();
}

// 결과 추출
int[] result = new int[k];
for (int i = k - 1; i>=0; i--) {
  result[i] = pq.poll().getKey();
}

return result;
```
