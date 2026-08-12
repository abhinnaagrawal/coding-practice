[← back to index](README.md)

# Stacks & Queues (incl. Monotonic Stack)

## When to recognize it
Nested/matching structure (brackets, expressions), or "next greater/smaller element," or you need last-in-first-out processing of history while scanning forward.

## Core idea
A stack lets you defer decisions — push elements while a condition holds, pop and resolve them once you see the element that "closes" or "breaks" that condition. A monotonic stack keeps its contents strictly increasing or decreasing, so each pop tells you "this popped element's next greater/smaller element is the current one."

## Gotchas
- Monotonic stack: decide increasing vs decreasing *before* coding — driven by whether you want next-greater or next-smaller.
- Store indices on the stack, not values, when you need distance/span (Daily Temperatures).
- Balanced brackets: mismatched closing bracket with empty stack → check `if not stack` before popping.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Easy | 87 | Given a string of brackets, determine if every bracket is properly closed and nested. | Push opening brackets; on a closing bracket, pop and verify it's the matching type — any mismatch or pop-on-empty means invalid. |
| [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) | Medium | — | For each day, find the number of days until a warmer temperature (monotonic stack of indices). | Keep a stack of indices with decreasing temperatures; when the current temperature is higher than the stack's top, pop it and record the day-distance as the answer for that popped index. |
| [Min Stack](https://leetcode.com/problems/min-stack/) | Medium | — | Design a stack supporting push/pop/top/getMin, all in O(1). | Store each element alongside the minimum-so-far at push time (or maintain a parallel min-stack) — the current min is always at the top. |
| [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) | Medium | — | Evaluate an arithmetic expression given in Reverse Polish (postfix) notation. | Push numbers onto a stack; when an operator appears, pop the top two operands, apply the operator, push the result back. |
| [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | Medium | 41 | Generate all combinations of well-formed parentheses for n pairs. | Backtrack while tracking open/close counts used so far; only add a closing bracket if fewer closes than opens have been placed, only add an opening bracket if opens used < n. |
| [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | Hard | — | Given bar heights, find the area of the largest rectangle that fits within the histogram. | Monotonic increasing stack of bar indices; when a shorter bar appears, pop taller bars and compute the rectangle area using the popped bar as height and the span between the new stack top and current index as width. |
| [Car Fleet](https://leetcode.com/problems/car-fleet/) | Medium | — | Simulate cars on a road with positions/speeds, count how many fleets arrive at the destination. | Sort cars by starting position (closest to target first), compute each car's time-to-target; process from closest to farthest — a car "joins" the fleet ahead if its time is ≤ the fleet ahead's time (it catches up, never passes). |

## Solutions

### Valid Parentheses
```python
def is_valid_parentheses(s):
    pairs = {')': '(', ']': '[', '}': '{'}
    stack = []
    for ch in s:
        if ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
        else:
            stack.append(ch)
    return not stack
```

### Daily Temperatures
```python
def daily_temperatures(temperatures):
    res = [0] * len(temperatures)
    stack = []  # indices with decreasing temperatures
    for i, t in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < t:
            j = stack.pop()
            res[j] = i - j  # distance to the day that's warmer
        stack.append(i)
    return res
```

### Min Stack
```python
class MinStack:
    def __init__(self):
        self.stack = []  # each entry: (value, min_so_far_including_this_value)

    def push(self, val):
        min_so_far = min(val, self.stack[-1][1]) if self.stack else val
        self.stack.append((val, min_so_far))

    def pop(self):
        self.stack.pop()

    def top(self):
        return self.stack[-1][0]

    def getMin(self):
        return self.stack[-1][1]
```

### Evaluate Reverse Polish Notation
```python
def eval_rpn(tokens):
    stack = []
    ops = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b),  # truncate toward zero, like the problem expects
    }
    for tok in tokens:
        if tok in ops:
            b, a = stack.pop(), stack.pop()  # b popped first: it's the more recent operand
            stack.append(ops[tok](a, b))
        else:
            stack.append(int(tok))
    return stack[0]
```

### Generate Parentheses
```python
def generate_parenthesis(n):
    res = []

    def backtrack(path, open_count, close_count):
        if len(path) == 2 * n:
            res.append(''.join(path))
            return
        if open_count < n:
            path.append('(')
            backtrack(path, open_count + 1, close_count)
            path.pop()
        if close_count < open_count:  # only close if there's an unmatched open
            path.append(')')
            backtrack(path, open_count, close_count + 1)
            path.pop()

    backtrack([], 0, 0)
    return res
```

### Largest Rectangle in Histogram
```python
def largest_rectangle_area(heights):
    stack = []  # indices with increasing heights
    max_area = 0
    for i, h in enumerate(heights + [0]):  # sentinel 0 forces a final flush of the stack
        while stack and heights[stack[-1]] >= h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)
    return max_area
```

### Car Fleet
```python
def car_fleet(target, position, speed):
    cars = sorted(zip(position, speed), reverse=True)  # closest to target first
    fleets = 0
    curr_time = 0
    for pos, spd in cars:
        time = (target - pos) / spd
        if time > curr_time:  # this car can't catch the fleet ahead -> starts its own
            fleets += 1
            curr_time = time
    return fleets
```
