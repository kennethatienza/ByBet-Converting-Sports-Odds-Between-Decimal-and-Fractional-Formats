# ByBet: Converting Sports Odds Between Decimal and Fractional Formats
> Sports odds can look quite different across websites, even when they mean the same basic price. **Decimal odds show the full return, while fractional odds show only the profit above the stake.** A short JavaScript tool shows both forms, but makes no claim about the next game result.

<img width="1536" height="1024" alt="Decimal and fractional odds, visualized" src="https://github.com/user-attachments/assets/0fe4da8a-258a-4c56-bd5f-04d3a8496cf6" />
<br>
<br>

The **[ByBet](https://official.bybet.com/)** website lists basketball, football, tennis, and other sports on its public pages. This guide uses **sample odds only**, not live ByBet prices, user details, or real sports feeds. Its **open code** can help adults compare formats, but it cannot tell anyone what will win.

### Decimal Prices Show Full Returns, Not Just Profit

**Decimal odds of 2.50** mean the full return is two and a half times the stake. With a **sample stake of ₱100**, that means ₱250 back if the chosen result wins. **Of that sum, ₱100 is the full stake and only ₱150 would count as profit.**

For this sample, **full return equals stake times decimal odds**, with no fees added to that sum. To find profit alone, **take the stake away from the full return** shown by that rule. These are sample sums, not a claim that any real bet was made or paid by anyone.

**Even odds of 2.00** mean a win would give back twice the full stake. A shorter price, like **1.50**, means the possible profit is less than the starting stake. Neither price can say which team will win, since both only state how a return gets priced.

### Fractional Prices Put Profit Above The Stake

**Fractional odds of 3/2** mean three parts profit for each two parts used as the stake. For a ₱100 sample stake, that ratio gives **₱150 profit and ₱250 as the total return**. That is the same price as decimal 2.50, though the two quotes look different on screen.

A **fraction of 1/2** shows one part profit for every two parts set as a stake. That matches a price of **1.50**, since the full return also gives back the stake. **The top number is not the full payout, which is a common source of wrong comparisons.**

### One Ratio Converts Both Formats Without Guesswork

To turn decimal odds into a fraction, **first take one away from the decimal price** shown. At **2.50**, the part left is 1.50, the same ratio as three over two. **Write that as 3/2, and the amount above the stake stays the same in both formats.**

Going the other way, **divide the top by the bottom** of the fraction and add one. With **4/5**, the profit share is 0.80, so the matching price is exactly 1.80. This step works with positive ratios, even when the result has more than two decimal places.

The rule is **fractional odds equal decimal odds minus one**, with both sides showing the profit part. So decimal odds are one plus the fraction, not just the fraction on its own. The same odds still mean the same implied rate, though the two quotes look far apart.

### JavaScript Functions Can Check Two Decimal Places

A simple tool can **accept two place decimal odds** and turn their profit parts into fractions. The code below checks **bad values and invalid fractions**, then uses a common divisor to trim fractions. The sample runs in Node.js, with no site login, paid feeds, or calls to the web.

The **reverse function** returns an exact decimal string, with brackets around digits that repeat forever. Tests check **2.50, 1.80, and one third**, along with some inputs that should raise errors. **One third repeats forever, so the sample uses brackets to mark the part that repeats.**

```javascript
const assert = require('node:assert/strict');

const MAX_PART = 10_000n;

function gcd(a, b) {
  while (b !== 0n) [a, b] = [b, a % b];
  return a;
}

function decimalToFractional(input) {
  if (typeof input !== 'string' && typeof input !== 'number') {
    throw new RangeError('Enter decimal odds as a number or string');
  }
  const value = String(input);
  if (value.length > 16 || !/^(?:0|[1-9]\d*)(?:\.\d{1,2})?$/.test(value)) {
    throw new RangeError('Use decimal odds with at most two decimal places');
  }

  const [whole, places = ''] = value.split('.');
  const cents = BigInt(whole) * 100n + BigInt(places.padEnd(2, '0'));
  const profit = cents - 100n;
  if (profit <= 0n) throw new RangeError('Decimal odds must exceed 1.00');

  const divisor = gcd(profit, 100n);
  const top = profit / divisor;
  const bottom = 100n / divisor;
  if (top > MAX_PART || bottom > MAX_PART) {
    throw new RangeError('Reduced fraction parts must be at most 10000');
  }
  return `${top}/${bottom}`;
}

function fractionalToDecimal(input) {
  if (typeof input !== 'string' || input.length > 11 ||
      !/^[1-9]\d*\/[1-9]\d*$/.test(input)) {
    throw new RangeError('Use a positive fraction such as 3/2');
  }
  const [top, bottom] = input.split('/').map(BigInt);
  if (top > MAX_PART || bottom > MAX_PART) {
    throw new RangeError('Each fraction part must be at most 10000');
  }

  // Use exact integers instead of floating-point approximations.
  const numerator = top + bottom;
  const whole = numerator / bottom;
  let remainder = numerator % bottom;
  if (remainder === 0n) return whole.toString();

  const digits = [];
  const seen = new Map();
  while (remainder !== 0n && !seen.has(remainder)) {
    seen.set(remainder, digits.length);
    remainder *= 10n;
    digits.push((remainder / bottom).toString());
    remainder %= bottom;
  }
  if (remainder === 0n) return `${whole}.${digits.join('')}`;

  const repeatAt = seen.get(remainder);
  return `${whole}.${digits.slice(0, repeatAt).join('')}(${digits.slice(repeatAt).join('')})`;
}

// Exact conversions, including repeating-decimal notation.
assert.equal(decimalToFractional(2.5), '3/2');
assert.equal(decimalToFractional('2.50'), '3/2');
assert.equal(decimalToFractional(1.8), '4/5');
assert.equal(decimalToFractional(1.91), '91/100');
assert.equal(decimalToFractional(1.01), '1/100');
assert.equal(decimalToFractional(10001), '10000/1');

assert.equal(fractionalToDecimal('3/2'), '2.5');
assert.equal(fractionalToDecimal('4/5'), '1.8');
assert.equal(fractionalToDecimal('1/3'), '1.(3)');
assert.equal(fractionalToDecimal('1/6'), '1.1(6)');
assert.equal(fractionalToDecimal('2/7'), '1.(285714)');
assert.equal(fractionalToDecimal('1576/25'), '64.04');
assert.equal(fractionalToDecimal('10000/1'), '10001');

// Reject malformed, invalid, or unsupported inputs.
for (const bad of [
  1, 1.00000000001, '1.001', 0, -2, NaN,
  Infinity, Number.MAX_VALUE, 10002, 101.01,
  '2e0', ' 2.5', '02.5', null
]) {
  assert.throws(() => decimalToFractional(bad), RangeError);
}

for (const bad of [
  '1/0', '0/1', '1/-3', '1 / 3', '01/2',
  '10001/2', '2/10001', '9007199254740991/2',
  null, 1.5
]) {
  assert.throws(() => fractionalToDecimal(bad), RangeError);
}

// Test every two-place decimal price from 1.01 through 100.99.
for (let cents = 101; cents <= 10099; cents++) {
  const exact = (BigInt(cents) / 100n).toString()
    + '.' + String(cents % 100).padStart(2, '0');

  const result = fractionalToDecimal(decimalToFractional(exact));
  assert.equal(Number(result), cents / 100,
    `Round trip failed for ${exact}`);
}

console.log('All sample, invalid-input, and 9,999 round-trip tests passed');
```

### Test Cases Expose Rounding And Input Errors

A tool reading **1.91 returns 91/100**, since the profit part equals ninety one hundredths. That value does not become 10/11 just because some sites show close prices in those forms. A true odds swap is not the same as rounding the result to a common fraction.

The code **rejects odds of 1.00** because each input must leave room for some gain. It also rejects zero as a fraction base, which would make the maths fail at once. **The 10,000 cap applies to reduced fraction parts, not to all decimal values at once.**

If a source shows **more than two decimal places**, the small sample tool does not use it. That choice keeps the **fraction exact for two place inputs**, rather than making a fake neat ratio. A larger tool could use more decimal places to keep extra digits without rounding them away.

### Comparing Market Quotes Requires Matching Bet Terms

Two odds figures mean little unless they price the **same team, game, market, and point line**. A basketball win price and a **point spread** may show the same price but mean different outcomes. **The market terms must match first, before anyone can say one listed price is higher.**

Odds can **change before or during play**, while a saved screen may show an old quote. The tool has **no live feed or time stamp**, so it cannot show whether prices still apply. No price format can remove the chance of loss, even when the maths is quite clear.

### Public Sports Pages Are Not Live Odds Feeds

The site lists basketball and football pages, but **neither page sends live odds to this sample tool**. This example does not scrape those pages, send requests, or ask for real user login data. **No API link or live market test** has been used to check actual ByBet prices.

A fair code review should check **the input rules, the output format, and the tests** before reuse. It should also state that **implied chance from odds is not a proven real event chance**. The bookmaker’s cut and price shifts can change that rate, even when the sums are right.

The **[ByBet basketball odds](https://official.bybet.com/sports/basketball)** page gives sports context, not access to a working data feed. This sports odds converter is a stand alone math sample, built to keep formats clear for readers. **Its output checks numbers, not results, and neither the page nor the code can promise returns.**
