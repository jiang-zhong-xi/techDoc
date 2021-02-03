### Object.is

ES6语法，目的时弥补“===”在“+0 -0 NaN”比较时的不足。

```javascript
function is(x: any, y: any) {
  // (x !== x && y !== y) 弥补NaN !== NaN的问题 如果和自身都不相等肯定是NaN
  // x === y && (1 / x === 1 / y) 弥补 +0 ！== -0 的问题 因为 1/+0 === 1/-0 是false
  // 其余情况用x === y && x !== 0
  // 相当于  (x === y && x !== 0) || (x === y && 1/x === 1/y) || (x!==x && y!=== y)
  return (
    (x === y && (x !== 0 || 1 / x === 1 / y)) || (x !== x && y !== y) // eslint-disable-line no-self-compare
  );
}
```

