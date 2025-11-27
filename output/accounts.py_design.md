**

---  

## `accounts.py` – Complete Self‑Contained Design  

```python
"""
accounts.py
-----------

A lightweight, self‑contained account management system for a trading
simulation platform.

Features
~~~~~~~~
* Create an account with an optional initial deposit.
* Deposit and withdraw cash (withdrawal blocked if it would cause a negative cash balance).
* Record buy / sell transactions for any ticker symbol.
* Enforce buying power limits and prevent selling more shares than owned.
* Compute current portfolio market value using a supplied ``get_share_price`` function.
* Calculate profit / loss relative to the initial cash deposit.
* Provide a snapshot of holdings, profit/loss, and a chronological transaction log.

The module is deliberately single‑file so it can be imported directly in tests
or wired up to a minimal UI.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal, ROUND_HALF_UP
from typing import Dict, List, Tuple, Callable

# ----------------------------------------------------------------------
# Helper: price lookup (stub for production – can be replaced by real API)
# ----------------------------------------------------------------------
def get_share_price(symbol: str) -> Decimal:
    """
    Return the current price for *symbol*.

    In the test environment a fixed price table is used:
        AAPL  -> 150.00
        TSLA  -> 750.00
        GOOGL -> 2800.00

    Raises:
        ValueError: if the symbol is unknown.
    """
    price_table = {
        "AAPL": Decimal("150.00"),
        "TSLA": Decimal("750.00"),
        "GOOGL": Decimal("2800.00"),
    }
    try:
        return price_table[symbol.upper()]
    except KeyError:
        raise ValueError(f"Unknown symbol '{symbol}'. No price available.")


# ----------------------------------------------------------------------
# Data structures
# ----------------------------------------------------------------------
@dataclass(frozen=True)
class Transaction:
    """
    Immutable record of a single account operation.

    Attributes
    ----------
    timestamp: datetime
        When the transaction occurred (UTC).
    type: str
        One of: 'deposit', 'withdraw', 'buy', 'sell'.
    symbol: str | None
        Ticker symbol for buy/sell, otherwise ``None``.
    quantity: Decimal
        Number of shares for buy/sell; zero for cash‑only ops.
    price: Decimal
        Price per share for buy/sell; zero for cash‑only ops.
    amount: Decimal
        Cash amount moved (positive for deposit/buy, negative for withdraw/sell).
    """
    timestamp: datetime
    type: str
    symbol: str | None
    quantity: Decimal
    price: Decimal
    amount: Decimal

    def __repr__(self) -> str:
        if self.type in {"buy", "sell"}:
            return (f"<Transaction {self.type.upper()} {self.quantity} {self.symbol} @ "
                    f"${self.price} on {self.timestamp.isoformat()}>")
        else:
            return (f"<Transaction {self.type.upper()} ${self.amount} on "
                    f"{self.timestamp.isoformat()}>")


# ----------------------------------------------------------------------
# Core class
# ----------------------------------------------------------------------
class Account:
    """
    Represents a single user's trading account.

    The class holds cash balance, a ledger of share positions, and a list of
    all transactions performed. All monetary values use :class:`Decimal`
    for exact financial arithmetic.
    """

    # ------------------------------------------------------------------
    # Construction / basic properties
    # ------------------------------------------------------------------
    def __init__(self, user_id: str, initial_deposit: Decimal | float | int = Decimal("0")):
        """
        Create a new account.

        Parameters
        ----------
        user_id: str
            Unique identifier for the account owner.
        initial_deposit: Decimal|float|int, optional
            Starting cash balance. Defaults to 0.
        """
        self.user_id: str = user_id
        self._cash_balance: Decimal = self._to_decimal(initial_deposit)
        self._initial_deposit: Decimal = self._cash_balance  # immutable reference
        self._positions: Dict[str, Decimal] = {}              # symbol → shares owned
        self._transactions: List[Transaction] = []

        if self._cash_balance > 0:
            self._record_transaction(
                type="deposit",
                symbol=None,
                quantity=Decimal("0"),
                price=Decimal("0"),
                amount=self._cash_balance,
            )

    # ------------------------------------------------------------------
    # Public API – cash management
    # ------------------------------------------------------------------
    def deposit(self, amount: Decimal | float | int) -> None:
        """
        Add cash to the account.

        Raises
        ------
        ValueError: if *amount* is non‑positive.
        """
        amount = self._to_decimal(amount)
        if amount <= 0:
            raise ValueError("Deposit amount must be positive.")
        self._cash_balance += amount
        self._record_transaction(
            type="deposit",
            symbol=None,
            quantity=Decimal("0"),
            price=Decimal("0"),
            amount=amount,
        )

    def withdraw(self, amount: Decimal | float | int) -> None:
        """
        Remove cash from the account.

        Raises
        ------
        ValueError: if *amount* is non‑positive or would cause a negative cash balance.
        """
        amount = self._to_decimal(amount)
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive.")
        if amount > self._cash_balance:
            raise ValueError("Insufficient cash for withdrawal.")
        self._cash_balance -= amount
        self._record_transaction(
            type="withdraw",
            symbol=None,
            quantity=Decimal("0"),
            price=Decimal("0"),
            amount=-amount,
        )

    # ------------------------------------------------------------------
    # Public API – trading
    # ------------------------------------------------------------------
    def buy(self, symbol: str, quantity: Decimal | float | int) -> None:
        """
        Record a purchase of *quantity* shares of *symbol*.

        The total cost (price × quantity) must be ≤ current cash balance.

        Raises
        ------
        ValueError: for invalid quantity, unknown symbol, or insufficient cash.
        """
        symbol = symbol.upper()
        qty = self._to_decimal(quantity, allow_zero=False)

        price = get_share_price(symbol)
        total_cost = (price * qty).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

        if total_cost > self._cash_balance:
            raise ValueError(
                f"Not enough cash to buy {qty} shares of {symbol} "
                f"(@ ${price} each = ${total_cost})."
            )

        # Update cash & position
        self._cash_balance -= total_cost
        self._positions[symbol] = self._positions.get(symbol, Decimal("0")) + qty

        self._record_transaction(
            type="buy",
            symbol=symbol,
            quantity=qty,
            price=price,
            amount=-total_cost,
        )

    def sell(self, symbol: str, quantity: Decimal | float | int) -> None:
        """
        Record a sale of *quantity* shares of *symbol*.

        The account must own at least *quantity* shares.

        Raises
        ------
        ValueError: for invalid quantity, unknown symbol, or insufficient shares.
        """
        symbol = symbol.upper()
        qty = self._to_decimal(quantity, allow_zero=False)

        owned = self._positions.get(symbol, Decimal("0"))
        if qty > owned:
            raise ValueError(
                f"Cannot sell {qty} shares of {symbol}; only {owned} owned."
            )

        price = get_share_price(symbol)
        proceeds = (price * qty).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

        # Update cash & position
        self._cash_balance += proceeds
        remaining = owned - qty
        if remaining == 0:
            self._positions.pop(symbol)
        else:
            self._positions[symbol] = remaining

        self._record_transaction(
            type="sell",
            symbol=symbol,
            quantity=qty,
            price=price,
            amount=proceeds,
        )

    # ------------------------------------------------------------------
    # Query helpers
    # ------------------------------------------------------------------
    @property
    def cash_balance(self) -> Decimal:
        """Current cash (uninvested) balance."""
        return self._cash_balance

    def get_holdings(self) -> Dict[str, Tuple[Decimal, Decimal]]:
        """
        Return a snapshot of share holdings.

        Returns
        -------
        dict
            Mapping ``symbol → (shares_owned, market_value)``.
            ``market_value`` = current_price × shares_owned.
        """
        holdings: Dict[str, Tuple[Decimal, Decimal]] = {}
        for symbol, shares in self._positions.items():
            price = get_share_price(symbol)
            market_val = (price * shares).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            holdings[symbol] = (shares, market_val)
        return holdings

    def portfolio_market_value(self) -> Decimal:
        """
        Total market value of all share positions (cash excluded).
        """
        total = Decimal("0")
        for symbol, shares in self._positions.items():
            price = get_share_price(symbol)
            total += price * shares
        return total.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def total_account_value(self) -> Decimal:
        """
        Cash balance + current market value of holdings.
        """
        total = self._cash_balance + self.portfolio_market_value()
        return total.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def profit_loss(self) -> Decimal:
        """
        Profit (positive) or loss (negative) relative to the initial cash deposit.

        Formula:
            (current total account value) – (initial deposit)
        """
        pl = self.total_account_value() - self._initial_deposit
        return pl.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def get_transactions(self) -> List[Transaction]:
        """
        Return a shallow copy of the transaction log ordered chronologically.
        """
        return list(self._transactions)

    # ------------------------------------------------------------------
    # Internal utilities
    # ------------------------------------------------------------------
    @staticmethod
    def _to_decimal(
        value: Decimal | float | int,
        allow_zero: bool = True,
    ) -> Decimal:
        """Convert *value* to ``Decimal`` with two‑decimal quantisation."""
        dec = Decimal(str(value)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
        if not allow_zero and dec <= 0:
            raise ValueError("Value must be positive and non‑zero.")
        return dec

    def _record_transaction(
        self,
        *,
        type: str,
        symbol: str | None,
        quantity: Decimal,
        price: Decimal,
        amount: Decimal,
    ) -> None:
        """Append a new :class:`Transaction` to the internal log."""
        txn = Transaction(
            timestamp=datetime.utcnow(),
            type=type,
            symbol=symbol,
            quantity=quantity,
            price=price,
            amount=amount,
        )
        self._transactions.append(txn)

    # ------------------------------------------------------------------
    # Pretty printing helpers (optional)
    # ------------------------------------------------------------------
    def __repr__(self) -> str:
        return (
            f"<Account user_id={self.user_id!r} cash=${self.cash_balance} "
            f"positions={len(self._positions)} txns={len(self._transactions)}>"
        )
```

### Design Overview  

| Component | Responsibility |
|-----------|-----------------|
| **`get_share_price`** | Stub that returns deterministic prices for a few tickers; can be replaced by a real market data provider. |
| **`Transaction` (dataclass)** | Immutable record of every deposit, withdrawal, buy, or sell. Stores timestamp, type, symbol (if applicable), quantity, price, and cash amount (positive for inflow, negative for outflow). |
| **`Account`** | Core class encapsulating: <br>• Cash balance & initial deposit <br>• Share positions (`symbol → shares`) <br>• Full transaction history <br>Provides methods for all required operations and queries, with rigorous validation and Decimal‑based arithmetic. |

### Key Implementation Decisions  

* **Decimal arithmetic** – Guarantees financial precision; all amounts are quantised to two decimal places.  
* **Immutable `Transaction`** – Makes the audit trail tamper‑proof.  
* **Validation** – Each public method raises `ValueError` on illegal actions (negative balances, insufficient shares, unknown symbols).  
* **Separation of concerns** – Helper methods (`_to_decimal`, `_record_transaction`) keep the public API clean.  
* **Extensibility** – `get_share_price` can be monkey‑patched or injected at runtime for real‑time pricing.  

With this single, self‑contained module the backend developer can directly import `Account`, instantiate accounts, and exercise the full feature set required by the trading simulation platform.