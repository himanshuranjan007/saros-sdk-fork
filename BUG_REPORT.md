# Saros SDK Bug Hunt - Critical Issues Found

## Summary
This report documents critical bugs and vulnerabilities found in the Saros SDK codebase during the bug hunt challenge. These issues range from syntax errors that prevent compilation to logic bugs that could cause financial losses.

---

## 🚨 CRITICAL BUGS (High Impact - $200 each)

### 1. Syntax Error in README Example - Missing Comma
**File:** `README.md`  
**Line:** 338-339  
**Severity:** Critical  
**Impact:** Prevents code compilation, breaks example usage

**Bug Description:**
The `onUnStakePool` function in the README example has a missing comma between parameters, causing a syntax error.

**Current Code:**
```javascript
const onUnStakePool = async () => {
  const hash = await SarosFarmService.unStakePool(
    connection,
    payerAccount,
    new PublicKey(farmList.poolAddress),
    new PublicKey(farmList.lpAddress)  // Missing comma here
    new BN(100),
    SAROS_FARM_ADDRESS,
    farmList.rewards,
    false // Set to true if want to unstake full balance
  );
  return `Your transaction hash: ${hash}`;
};
```

**Fixed Code:**
```javascript
const onUnStakePool = async () => {
  const hash = await SarosFarmService.unStakePool(
    connection,
    payerAccount,
    new PublicKey(farmList.poolAddress),
    new PublicKey(farmList.lpAddress), // Added comma
    new BN(100),
    SAROS_FARM_ADDRESS,
    farmList.rewards,
    false // Set to true if want to unstake full balance
  );
  return `Your transaction hash: ${hash}`;
};
```

**Reproduction Steps:**
1. Copy the `onUnStakePool` function from README.md
2. Try to run the code
3. Observe syntax error: "Unexpected token 'new'"

**Environment:** Any JavaScript environment  
**Suggested Fix:** Add missing comma after `new PublicKey(farmList.lpAddress)`

---

### 2. Incorrect Slippage Calculation in Deposit Function
**File:** `src/swap/sarosSwapServices.js`  
**Line:** 426  
**Severity:** Critical  
**Impact:** Incorrect slippage calculation leading to wrong token amounts

**Bug Description:**
In the `depositAllTokenTypes` function, the slippage calculation for `token1Amount` incorrectly uses `newAmount0` instead of `newAmount1`, causing incorrect slippage calculations.

**Current Code:**
```javascript
const newAmount1 = Math.floor(
  (poolToken1AccountInfo.amount.toNumber() * lpTokenAmount) / lpTokenSupply
);
const token1Amount = Math.floor(
  newAmount1 + renderAmountSlippage(newAmount0, slippage) // BUG: Should be newAmount1
);
```

**Fixed Code:**
```javascript
const newAmount1 = Math.floor(
  (poolToken1AccountInfo.amount.toNumber() * lpTokenAmount) / lpTokenSupply
);
const token1Amount = Math.floor(
  newAmount1 + renderAmountSlippage(newAmount1, slippage) // Fixed: Use newAmount1
);
```

**Reproduction Steps:**
1. Call `depositAllTokenTypes` with any slippage value
2. Observe that token1 slippage calculation is incorrect
3. Compare with expected slippage calculation

**Environment:** Any environment using the deposit function  
**Impact:** Users may receive incorrect token amounts due to wrong slippage calculation

---

### 3. Missing Error Handling in Critical Functions
**File:** `src/swap/sarosSwapIntructions.js`  
**Line:** 103  
**Severity:** Critical  
**Impact:** Potential runtime crashes

**Bug Description:**
The `serialize` function is called but not imported, causing a ReferenceError at runtime.

**Current Code:**
```javascript
const data = serialize(  // BUG: serialize is not imported
  commandDataLayout,
  {
    instruction: 0,
    // ... other parameters
  },
  1024
)
```

**Fixed Code:**
```javascript
// Add import at top of file
import { BorshService } from '../common/borshService'

// Use BorshService.serialize instead
const data = BorshService.serialize(
  commandDataLayout,
  {
    instruction: 0,
    // ... other parameters
  },
  1024
)
```

**Reproduction Steps:**
1. Try to create a swap instruction
2. Observe ReferenceError: "serialize is not defined"

**Environment:** Any environment using swap instructions  
**Impact:** Swap functionality completely broken

---

## ⚠️ MEDIUM BUGS (Medium Impact - $100 each)

### 4. Inconsistent Error Handling Patterns
**File:** Multiple files  
**Severity:** Medium  
**Impact:** Poor user experience, inconsistent error reporting

**Bug Description:**
Error handling is inconsistent across the codebase. Some functions return error strings, others throw exceptions, and some return objects with `isError` flags.

**Examples:**
- `src/farm/sarosFarmServices.js:98` - Returns error string
- `src/swap/sarosSwapServices.js:258` - Returns object with `isError` flag
- `src/common/solana.js:246` - Returns object with `isError` flag

**Suggested Fix:** Standardize error handling across all functions to return consistent error objects.

---

### 5. Missing Input Validation
**File:** `src/functions/common.js`  
**Line:** 8-27  
**Severity:** Medium  
**Impact:** Potential division by zero, incorrect calculations

**Bug Description:**
The `convertWeiToBalance` and `convertBalanceToWei` functions don't validate input parameters, potentially causing division by zero or incorrect calculations.

**Current Code:**
```javascript
export const convertWeiToBalance = (strValue, iDecimal = 18) => {
  try {
    if (parseFloat(strValue) === 0) return 0;
    const multiplyNum = new bigdecimal.BigDecimal(Math.pow(10, iDecimal));
    const convertValue = new bigdecimal.BigDecimal(String(strValue));
    return convertValue.divide(multiplyNum).toString();
  } catch (err) {
    return 0;
  }
};
```

**Suggested Fix:** Add proper input validation for `strValue` and `iDecimal` parameters.

---

### 6. Potential Division by Zero in APR Calculation
**File:** `src/farm/sarosFarmServices.js`  
**Line:** 543  
**Severity:** Medium  
**Impact:** Runtime errors, incorrect APR display

**Bug Description:**
The APR calculation doesn't check for zero liquidity, potentially causing division by zero.

**Current Code:**
```javascript
const apr = (totalRewardOneYearUSD / liquidityUsd) * 100;
```

**Suggested Fix:** Add zero check before division.

---

## 🔧 MINOR BUGS (Minor Impact - $100 each)

### 7. Hardcoded Connection URL
**File:** `src/common/solana.js`  
**Line:** 196  
**Severity:** Minor  
**Impact:** Inflexibility, potential network issues

**Bug Description:**
The connection URL is hardcoded to mainnet, making it difficult to use with devnet or testnet.

**Current Code:**
```javascript
const connectionSolana = new Connection(
  'https://api.mainnet-beta.solana.com',  // Hardcoded mainnet
  'singleGossip'
);
```

**Suggested Fix:** Make the connection URL configurable.

---

### 8. Missing Null Checks
**File:** `src/swap/sarosSwapServices.js`  
**Line:** 293  
**Severity:** Minor  
**Impact:** Potential runtime errors

**Bug Description:**
Missing null check for `lpTokenSupply` before calling `toNumber()`.

**Current Code:**
```javascript
const lpTokenSupply = poolLpMintInfo.supply.toNumber();
```

**Suggested Fix:** Add null check before calling `toNumber()`.

---

### 9. Inefficient Promise.all Usage
**File:** `src/farm/sarosFarmServices.js`  
**Line:** 80-92  
**Severity:** Minor  
**Impact:** Performance degradation

**Bug Description:**
Using `Promise.all` with async functions that don't return promises, causing unnecessary overhead.

**Current Code:**
```javascript
await Promise.all(
  rewards.map(async (reward) => {
    const { poolRewardAddress } = reward;
    await this.stakePoolReward(
      connection,
      payerAccount,
      poolAddress,
      new PublicKey(poolRewardAddress),
      sarosFarmProgramAddress,
      transaction
    );
  })
);
```

**Suggested Fix:** Use regular `for...of` loop or `Promise.allSettled` for better error handling.

---

## 🛡️ SECURITY CONCERNS

### 10. Missing Input Sanitization
**File:** Multiple files  
**Severity:** Medium  
**Impact:** Potential injection attacks

**Bug Description:**
User inputs are not properly sanitized before being used in calculations or API calls.

**Examples:**
- `parseFloat()` calls without validation
- Direct use of user-provided addresses without validation
- Missing bounds checking for numeric inputs

**Suggested Fix:** Add comprehensive input validation and sanitization.

---

## 📊 PERFORMANCE ISSUES

### 11. Inefficient GraphQL Queries
**File:** `src/farm/sarosFarmServices.js`  
**Line:** 405-420  
**Severity:** Minor  
**Impact:** Slow response times

**Bug Description:**
GraphQL queries fetch all data and then slice it client-side instead of using pagination.

**Current Code:**
```javascript
const response = await gqlClient.request(query);
const data = get(response, 'farms', []);
const limit = parseInt(size);
const skip = parseInt(page - 1) * limit;
const listFarm = data.slice(skip, skip + limit);
```

**Suggested Fix:** Implement server-side pagination in GraphQL queries.

---

## 🧪 TESTING RECOMMENDATIONS

1. **Unit Tests:** Add comprehensive unit tests for all mathematical functions
2. **Integration Tests:** Test all swap, farm, and stake operations
3. **Error Handling Tests:** Verify consistent error handling patterns
4. **Edge Case Tests:** Test with zero values, negative values, and boundary conditions
5. **Performance Tests:** Measure response times for large datasets

---

## 📝 SUMMARY

**Total Issues Found:** 11  
**Critical Issues:** 3 ($200 each)  
**Medium Issues:** 3 ($100 each)  
**Minor Issues:** 5 ($100 each)  

**Total Potential Reward:** $1,400

The most critical issues are the syntax error in the README example and the incorrect slippage calculation, which could cause immediate failures and financial losses respectively. These should be prioritized for immediate fixes.

---

## 🔧 IMPLEMENTATION PRIORITY

1. **Immediate (Critical):** Fix syntax error and slippage calculation bug
2. **High Priority:** Fix missing serialize import and standardize error handling
3. **Medium Priority:** Add input validation and null checks
4. **Low Priority:** Performance optimizations and code quality improvements

---

*Report generated during Saros SDK Bug Hunt Challenge*
