# Valid Parentheses [#20](https://leetcode.com/problems/valid-parentheses/description/)
## 문제
대중소 세 종류 괄호로 된 입력값이 유효한지 판별
* 입력: `[]{}()`
* 출력: true

## 전략
### 1. 스택: 여는 괄호는 push, 닫는 괄호는 pop하여 마지막에 스택이 비어있으면 유효
```java
Deque<Character> stack = new ArrayDeque<>();
for (char c : s.toCharArray()) {
  if (c == '(' || c == '{' || c == '[') {
    stack.push(c);
  } else {
    if (stack.isEmpty()) return false;
    char top = stack.pop();
    if ((c == ')' && top != '(') || (c == '}' && top != '{') || (c == ']' && top != '[')) return false;
  }
}
return stack.isEmpty();
```
* 시간 복잡도
  * 순회하는 동안 한 번씩만 검사하므로 `O(n)`
  * `push`, `pop`, `peek` 연산 모두 `O(1)`
  * 따라서, `O(n)`
* 공간 복잡도
  * 스택에는 n개의 문자가 쌓이므로, `O(n)` 
* 개선사항
  * 고정 리터럴들은 `Map`에 넣어서 관리 
    ```java
    Map<Character, Character table = new HashMap<>() {{
      put(')', '(');
      put('}', '{');
      put(']', '[');
    }}
    
    Deque<Character> stack = new ArrayDeque<>();
    for (int i = 0; i < s.length(); i++) {
      if (!table.containsKey(s.charAt(i))) stack.push(s.charAt(i));
      else if (stack.isEmpty() || table.get(s.charAt(i)) != stack.pop()) return false;
    }
    return stack.size() == 0;
    ```

# Daily Temperature (단조 감소) [#739](https://leetcode.com/problems/daily-temperatures/description/)
## 문제
하루마다 측정된 온도를 배열로 입력받아, 더 따뜻한 날씨를 위해서 며칠을 더 기다려야 하는지 출력
* 입력: `temperatures = [23, 24, 25, 21, 19, 22, 26, 23]`
* 출력: `[1, 1, 4, 2, 1, 1, 0, 0]`

## 전략
### 1. 스택: 스택에는 아직 더 따뜻한 날을 찾지 못한 인덱스를 저장하여 내림차순 온도만 남도록 관리
```java
int result = new int[temperatures.length];
Deque<Integer> stack = new ArrayDeque<>();
for (int i = 0; i < temperatures.length; i++) {
  while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
    int last = stack.pop();
    result[last] = i - last;
  }
  stack.push(i);
}
return result;
```
* 시간 복잡도
  * push, pop은 한번씩만 수행되므로, O(n) ???
* 공간 복잡도
  * 스택에 최대 n개가 들어갈 수 있으므로, `O(n)`
