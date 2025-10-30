# Sort List
## 전략
### 1. Bubble Sort: 인접한 두 원소를 비교해서 큰 값을 뒤로 보내기
```java
for (int i = 0; i < n - 1; i++) {
  for (int j = 0; j < i - 1; j++) {
    if (arr[j] > arr[j+1]) {
      swap(arr, j, j+1);
    }
  }
}
```
* 시간 복잡도: `O(n^2)`
* 공간 복잡도: `O(1)`

### 2. Selection Sort: 남은 구간 중 최소값을 찾아서 맨 앞과 교환
```java
for (int i = 0; i < n - 1; i++) {
  int min = i;
  for (int j = i+1; j < n; j++) {
    if (arr[j] < arr[min]) {
      min = j;
    }
    swap(arr, i, min);
  }
}
```
* 시간 복잡도: `O(n^2)`
* 공간 복잡도: `O(1)`

### 3. Insertion Sort: 이미 정렬된 구간에서 현재 원소를 삽입
```java
for (int i = 1; i < n; i++) {
  int key = arr[i];
  int j = i - 1;
  while (j>=0 && arr[j] > key) {
    arr[j+1] = arr[j--];
  }
  arr[j+1] = key;
}
```
* 시간 복잡도: `O(n^2)`
* 공간 복잡도: `O(1)`

### 4. Merge Sort: 분할 정복
```java
void mergeSort(int[] a, int l, int r) {
  if (l >= r) return;
  int m = (1+r)/2;
  mergeSort(a, l, m);
  mergeSort(a, m+1, r);
  merge(a, l, m, r);
}
```
* 시간 복잡도: `O(n log n)`
* 공간 복잡도: `O(n)`

### 5. Quick Sort: 피봇을 기준으로 좌우를 나누고 재귀 정렬
```java
void quickSort(int[] a, int l, int r) {
  if (l>=r) return;
  int p = partition(a, l, r);
  quickSort(a, l, p-1);
  quickSort(a, p+1, r);
}
```
* 시간 복잡도: 
* 공간 복잡도:

### 6. Heap Sort: 힙을 이용해서 최댓값을 추출하여 뒤에서부터 채움
```java
void heapSort(int[] a) {
  buildMaxHeap(a);
  for (int i = a.length - 1; i>0; i--) {
    swap(a, 0, i);
    heapify(a, o, i);
  }
}
```
* 시간 복잡도: `O(n logn)`
* 공간 복잡도: `O(1)`


