# 1920 Build Array from Permutation
## 문제
주어진 배열 nums에 대해 다음 규칙을 만족하는 새로운 배열 ans를 생성한다:  `ans = nums[nums[i]]` (간단 배열 문제)
* 입력: `nums = [0, 2, 1]`
* 출력: `[0, 1, 2]`

## 전략
단순 반복문으로 해결할 수 있는 for문 O(n) 배열 문제이다.
### 포인트
* 배열 변수는 단순 포인터이므로, 배열을 생성할 떄 `int* arr = (int*)malloc(size);` 로 포인터 아래 배열 크기만큼 공간을 확보한다. 
* 자바에서 배열은 객체이기 떄문에 객체 헤더에 길이 정보를 담아 `arr.length`로 접근 가능하다. 하지만, C는 순수한 포인터이기 떄문에 길이 정보가 없어, 길이 변수까지 함께 전달해야 한다. 
### 코드 
```C
int* buildArray(int* nums, int numsSize, int* returnSize) {
    // 배열 생성
    int* arr = (int*)malloc(sizeof(int) * numsSize);
    // 배열 초기화
    for (int i = 0; i < numsSize; i++) {
        arr[i] = nums[nums[i]];
    }

    // 배열 반환
    *returnSize = numsSize; // returnSize 주소에 numsSize 값을 복사한다. 이걸 해줘야 비로소 arr가 완성!
    return arr;
}
```


# 1480 Running Sum of 1d Array
## 문제
주어진 배열 nums에 대해 합을 누적하여 새로운 배열 ans를 생성한다. 
* 입력: `nums = [1, 2, 3]`
* 출력: `[1, 3, 6]`

## 전략
배열을 오염시키지 않도록 sum 변수를 하나 추가하고, 반복문으로 누적합을 구한다.
### 코드
```C
int* runningSum(int* nums, int numsSize, int* returnSize) {
    // 배열 생성
    int* ans = (int*)malloc(sizeof(int) * numsSize);

    // 배열 초기화
    int sum = 0;
    for (int i = 0; i < numsSize; i++) {
        sum += nums[i];
        ans[i] = sum;
    }

    // 배열 반환
    *returnSize = numSize;
    return ans;
}
``` 
### 발전
`nums` 배열을 그대로 덮어씌워서 메모리 활용도를 높일 수도 있다. 
```C
int* runningSum(int* nums, int numsSize, int* returnSize) {
    // 배열 계산
    for (int i = 1; i < numsSize; i++) {
        nums[i] += nums[i-1];
    }

    // 배열 반환
    *returnSize = numsSize;
    return nums;

}
```

# 1108 Defanging an IP Address
## 문제
## 전략

# 2011 Final Value of Variable After Performing Operations
## 문제
## 전략
