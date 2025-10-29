# Two Sum [#1](https://leetcode.com/problems/two-sum/)
## 문제
더해서 target 값을 만들 수 있는 배열의 두 숫자 인덱스를 리턴
* 입력: `nums = [2, 6, 11, 15], target = 8`
* 출력: `[0, 1]`

## 전략
배열에서 두 수의 합이 target되는 인덱스 쌍을 찾기 위해 다음 두가지 방법이 존재합니다. 
### 1. 브루트포스: 모든 쌍을 탐색한다
```java
for (int i = 0; i < nums.length; i++) {
 for (int j = i+1; j < nums.length; j++) {
   if (nums[i] + nums[j] == target) {
     return new int[]{i, j};
   }
 }
}
```
* 시간복잡도:
  * 외부 루프: `i`가 `0 ~ n-1`까지 반복하여, n번 반복
  * 내부 루프: `j`가 `i+1 ~ n-1`까지 반복하여, 평균적으로 `n/2`번 반복
  * 따라서, `1 + 2 + ... + (n-1) = n(n-1)/2`번 if문에서 비교하므로, 시간복잡도는 `O(n^2)`
* 공간복잡도:
  * 추가로 사용하는 배열이나 자료구조가 없기 때문에, `O(1)`
### 2. 해시맵: `target - num` 존재 여부를 키로 빠르게 탐색
```java
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < nums.length; i++){
 map.put(nums[i], i);
}

for (int i = 0; i < nums.length; i++) {
  int complement = target - nums[i];
  if (map.containsKey(complement) && i != map.get(complement){
    return new int[]{i, map.get(complement)};
 }
}
```
* 시간 복잡도
  * 배열을 한 번 순회하므로, `n`번 반복
  * 각 반복에서,
    * `target - nums[i]` 계산은 `O(1)`
    * `map.containsKey(complement)`에서, HashMap 조회는 평균 `O(1)`
  * 따라서, `O(n)`
* 공간 복잡도
  * HashMap을 사용하여, 최대 n개의 key-value 쌍을 저장하므로, `O(n)`
  * 반환되는 `int[2]`는 상수크기이므로, `O(1)`
  * 따라서, `O(n) + O(1)`이므로 `O(n)`이다. 
* 코드 품질
  * `map`의 저장과 조회를 하나의 for문으로 처리할 수 있습니다.  
    ```java
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        map.put(nums[i], i);
    }
    ```



# 3Sum [#15](https://leetcode.com/problems/3sum/description/)
## 문제
배열을 입력받아서 합으로 0을 만들 수 있는 3개의 원소를 리턴
* 입력: `nums = [-1, 0, 1, 2, -1, -5]`
* 출력: `[ [-1, 0, 1], [-1, -1, 2] ]`

## 전략
인덱스를 출력하는 문제가 아니므로, 먼저 정렬(`O(n log n)`)을 진행한다. 

### 1. 브루트포스: 모든 쌍을 (중복을 주의하며) 탐색한다.
```java
List<List<Integer>> results = new LinkedList<>();
Arrays.sort(nums);

for (int i = 0; i < nums.length - 2; i++) {
  if (i > 0 && nums[i] == nums[i - 1]) continue; // 결과 중복 피하기
  for (int j = i + 1; j < n - 1; j++) {
    if (j > i + 1; nums[j] == nums[j - 1]) continue;
    for (int k = j + 1; k < n; k++) {
      if (k > j + 1; nums[k] == nums[k - 1]) continue;

      int sum = nums[i] + nums[j] + nums[k];
      if (sum == 0) {
        results.add(List.of(nums[i], nums[j], nums[k]));
      }
    }
  }
}
return results;
```
* 시간 복잡도
  * 정렬은 `O(n logn)`
  * 반복은 i는 대략 n번, j는 대략 n/2번, k는 대략 n/3번으로, `nC3 = n(n-1)(n-2)/6`인 `O(n^3)`
  * 따라서, `O(n logn) + O(n^3)`이므로 `O(n^3)`
* 공간 복잡도
  * 알고리즘에서는 추가 구조가 없기 때문에 `O(1)`이지만, 결과 저장은 각 음수마다 2가지의 양수 조합이 가능하므로 `O(n^2)`까지 늘 수 있습니다.
* 참고
  * 리스트에 추가 작업이 많고, 임의 접근이 없는 구조이므로 `LinkedList`를 선택했습니다. 
### 2. 투포인터 
```java
int left, right, sum;
List<List<Integer>> results = new ArrayList<>();
Arrays.sort(nums);

for (int i = 0; i < nums.length - 2; i++) {
  if (i > 0; nums[i] == nums[i - 1]) continue; // 피벗 중복 제거

  left = i + 1;
  right = nums.length - 1;
  while (left < right) {
    sum = nums[i] + nums[left] + nums[right];
    if (sum < 0) {
      left += 1;
    } else if (sum > 0) {
      right -= 1;
    } else {
      results.add(Arrays.asList(nums[i], nums[left], nums[right]));
      while (left < right && nums[left] == nums[left + 1]) left += 1; // 좌 중복 제거
      while (left < right && nums[right] == nums[right - 1]) right -= 1; // 우 중복 제거
      left += 1;
      right -= 1;
    }
  }
}
return results;
```
* 시간 복잡도
  * 정렬은 `O(n logn)`
  * 반복은 바깥 루프에서 n번, 내부 포인터 이동은 left, right가 서로를 향해 한번씩만 이동하므로 `O(n)`이다.
  * 따라서, `O(n logn) + n * O(n)`이므로, `O(n^2)`
* 공간 복잡도
  * 알고리즘에서는 추가 구조가 없기 때문에 `O(1)`이지만, 결과 저장은 `O(n^2)`까지 늘 수 있습니다.
