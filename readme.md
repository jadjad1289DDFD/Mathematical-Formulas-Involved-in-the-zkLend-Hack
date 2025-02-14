# Mathematical Formulas Involved in the zkLend Hack



---

### 1. lending_accumulator Update Formula
The `lending_accumulator` is recalculated after flash loan repayments to distribute extra reserves to depositors. The modification was proposed by the auditor and has been updated here https://github.com/zkLend/zklend-v1-core/pull/33 

The formula is:

```markdown
lending_accumulator_new = ((reserve_balance + total_debt - treasury_fee) * 10^27) / ztoken_supply
```

**Key Variables**:

- **reserve_balance**: Total reserves in the market (includes flash loan repayments).
- **total_debt**: Outstanding borrows (0 in an empty market).
- **treasury_fee**: Protocol fee (e.g., 14.9% of extra reserves).
- **ztoken_supply**: Total supply of zTokens (initially 1 wei).

---

### 2. **zToken Minting (Deposit)**

When depositing assets, the user receives zTokens based on the current `lending_accumulator`:

```markdown
zToken_minted = floor(deposit_amount * 10^27 / lending_accumulator)
```

**Truncation**: The `floor()` operator discards decimals (integer division).

---

### 3. **Asset Withdrawal (Burn zTokens)**

When burning zTokens to withdraw assets, the amount received is:

```markdown
withdrawn_amount = floor(zToken_burned * lending_accumulator / 10^27)
```

**Truncation**: Again, fractional results are discarded.

---

### Attack Workflow with Formulas

#### **Step 1: Manipulate lending_accumulator via Flash Loans**

**Initial Deposit:**

- Deposit 1 wei of wstETH.
- `ztoken_supply = 1 wei`.
- Initial `lending_accumulator = 10^27`.

**Flash Loan Amplification:**

- Borrow 1 wei via flash loan and repay 1,000 wei.
- Update `lending_accumulator`:

```markdown
lending_accumulator_new = (1000 - 149) * 10^27 = 851 * 10^27
```

Repeat this to exponentially inflate `lending_accumulator` (e.g., to `4.069 * 10^45`).

#### **Step 2: Exploit Rounding Errors**

**Deposit Phase:**

- Deposit `4.069 * 10^18` wei (4.069 wstETH).
- Calculate zTokens minted:

```markdown
zToken_minted = floor(4.069 * 10^45 / 4.069 * 10^18) = 1 zToken
```

**Withdrawal Phase:**

- Request withdrawal of `6.1039 * 10^18` wei (6.1039 wstETH).
- Calculate zTokens required to burn:

```markdown
zToken_burned = floor(6.1039 * 10^45 / 4.069 * 10^18) = 1 zToken
```

- Withdraw `6.1039 * 10^18` wei by burning 1 zToken:

```markdown
withdrawn_amount = floor(1 * 4.069 * 10^45 / 10^27) = 4.069 * 10^18 wei
```

**Discrepancy**: The protocol allows withdrawing `6.1039 * 10^18` wei (requested) but only deducts 1 zToken (worth `4.069 * 10^18` wei). This mismatch is due to separate truncation errors in minting and burning logic.

---

### Key Exploited Flaws

1. **Empty Market Vulnerability**:  
   Low `ztoken_supply` magnifies `lending_accumulator` changes.

2. **Truncation in Integer Division**:  
   Minting rounds down `zToken_minted`, allowing attackers to burn fewer zTokens than required for large withdrawals.

3. **Dynamic lending_accumulator**:  
   Manipulated via flash loans to create arbitrage between deposit/withdrawal phases.

---

### Mitigation Formulas

1. **Minimum Liquidity Requirement**:  
   Enforce `ztoken_supply >= min_threshold` to prevent empty-market exploits.


2. **Bounds on lending_accumulator**:  
   Limit growth rate:

```markdown
lending_accumulator_new <= lending_accumulator_old * (1 + max_growth_rate)
```

---

### Conclusion

The attacker exploited the protocol’s reliance on truncation in integer division and the ability to manipulate `lending_accumulator` via flash loans. By inflating `lending_accumulator` and exploiting rounding discrepancies, they withdrew more assets than deposited. Fixes involve stricter parameter validation, improved rounding logic, and isolating flash loan reserves.

---



