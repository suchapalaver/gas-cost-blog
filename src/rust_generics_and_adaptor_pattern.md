# Rust Generics & Adapter Pattern in Blockchain Gas Calculation

## Adding Support for L1 and OP Stack Chains in Gas Cost Calculation

As part of recent work at [Semiotic](https://semiotic.ai/), I implemented a flexible,
type-safe system for calculating gas costs across multiple Ethereum-compatible chains.
This includes both L1 chains (like Ethereum Mainnet, where no L1 data fees are involved)
and [OP Stack](https://docs.optimism.io/stack/getting-started)-based L2s (like Optimism,
where gas fees include an additional L1 data component). This post walks through how
I used [Rust's generics](https://doc.rust-lang.org/book/ch10-01-syntax.html) and the
[adapter pattern](https://refactoring.guru/design-patterns/adapter) to support this
cleanly across network types, while maintaining compile-time guarantees and avoiding
runtime branching.

## 1. The Core Generic Architecture

### Network-Parameterized Gas Calculator

```rust
impl<N: Network> GasCostCalculator<N>
where
    N::TransactionResponse: TransactionTrait + Typed2718,
{
    // Generic implementation works for any network type N
}
```

**Key Learning**: This demonstrates **bounded generics** where `N` must implement the `Network` trait, and its associated type `TransactionResponse` must implement specific traits. This constraint ensures type safety while allowing generic behavior. The types used here, such as `U256` and `Network` (and others like `Address`, `Log`, `TransactionTrait`, `Typed2718`, `ReceiptResponse` appearing later), are core components provided by the [Alloy Project](https://alloy.rs/), a foundational library for Ethereum development in Rust.

## 2. The Adapter Pattern with Generic Traits

### The Adapter Trait

```rust
trait ReceiptAdapter<N: Network> {
    fn gas_used(&self, receipt: &N::ReceiptResponse) -> U256;
    fn effective_gas_price(&self, receipt: &N::ReceiptResponse) -> U256;
    fn l1_data_fee(&self, receipt: &N::ReceiptResponse) -> Option<U256>;
}
```

**Key Learning**: This trait is **parameterized by the network type** `N`, allowing the same interface to work with different network receipt types while maintaining compile-time type safety.

### Network-Specific Adapters

```rust
impl ReceiptAdapter<Ethereum> for EthereumReceiptAdapter {
    fn gas_used(&self, receipt: &<Ethereum as Network>::ReceiptResponse) -> U256 {
        U256::from(receipt.gas_used)
    }

    fn effective_gas_price(&self, receipt: &<Ethereum as Network>::ReceiptResponse) -> U256 {
        U256::from(receipt.effective_gas_price)
    }

    fn l1_data_fee(&self, _receipt: &<Ethereum as Network>::ReceiptResponse) -> Option<U256> {
        None // Ethereum L1 doesn't have L1 data fees
    }
}
```

```rust
impl ReceiptAdapter<Optimism> for OptimismReceiptAdapter {
    fn gas_used(&self, receipt: &<Optimism as Network>::ReceiptResponse) -> U256 {
        U256::from(receipt.inner.gas_used)
    }

    fn effective_gas_price(&self, receipt: &<Optimism as Network>::ReceiptResponse) -> U256 {
        U256::from(receipt.inner.effective_gas_price)
    }

    fn l1_data_fee(&self, receipt: &<Optimism as Network>::ReceiptResponse) -> Option<U256> {
        Some(U256::from(receipt.l1_block_info.l1_fee.unwrap_or_default()))
    }
}
```

**Key Learning**: Each adapter implements the same interface but handles network-specific receipt structures. Notice how Optimism receipts have an `inner` field and `l1_block_info`, while Ethereum receipts have direct access. For Optimism, we use `OptimismReceiptAdapter`, which leverages types and structures from the [op-alloy crate](https://github.com/alloy-rs/op-alloy), a specialized extension of Alloy for OP Stack chains.

## 3. Generic Functions with Multiple Type Parameters

### Processing Transfer Events

```rust
async fn process_transfer_event<A: ReceiptAdapter<N>>(
    &self,
    log: &Log,
    adapter: &A,
) -> anyhow::Result<Option<GasForTx>> {
```

**Key Learning**: This method demonstrates **multiple generic parameters**:

- `N` (the network type) from the impl block
- `A` (the adapter type) as a method-level generic

The constraint `A: ReceiptAdapter<N>` ensures the adapter matches the network type.

### Using the Adapter for Network-Agnostic Logic

```rust
let gas_used = adapter.gas_used(&receipt);
let receipt_effective_gas_price = adapter.effective_gas_price(&receipt);

let effective_gas_price = GasCalculationCore::calculate_effective_gas_price::<N>(
    &transaction,
    receipt_effective_gas_price,
);

// Create appropriate GasForTx based on network type
let gas_for_tx = match adapter.l1_data_fee(&receipt) {
    Some(l1_fee) => {
        // L2 network with L1 data fees
        GasForTx::from((gas_used, effective_gas_price, l1_fee))
    }
    None => {
        // L1 network or L2 without L1 data fees
        GasForTx::from((gas_used, effective_gas_price))
    }
};
```

**Key Learning**: The adapter pattern shines here - the same logic works for both networks, but the adapter handles the different receipt structures transparently.

## 4. Associated Types and Qualified Syntax

### Explicit Associated Type Usage

```rust
fn gas_used(&self, receipt: &<Ethereum as Network>::ReceiptResponse) -> U256 {
    U256::from(receipt.gas_used)
}
```

**Key Learning**: The syntax `<Ethereum as Network>::ReceiptResponse` is **qualified syntax** that explicitly refers to the `ReceiptResponse` associated type from the `Network` trait implementation for `Ethereum`.

## 5. Generic Core Logic

### Stateless Helper Functions

```rust
fn calculate_blob_gas_cost<N: Network>(transaction: &N::TransactionResponse) -> U256 {
    if !transaction.is_eip4844() {
        return U256::ZERO;
    }

    let blob_count = transaction
        .blob_versioned_hashes()
        .map(|hashes| hashes.len())
        .unwrap_or_default();

    let blob_gas_used = U256::from(blob_count * BLOB_GAS_PER_BLOB as usize);
    let blob_gas_price = U256::from(transaction.max_fee_per_blob_gas().unwrap_or_default());

    blob_gas_used.saturating_mul(blob_gas_price)
}
```

**Key Learning**: Generic functions can work with any type that satisfies the bounds. Here, any `N::TransactionResponse` that implements the required traits can use this blob gas calculation. The `is_eip4844()` method checks if the transaction conforms to [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844), which introduced blob transactions.

## 6. Network-Specific Implementations

### Thin Wrapper Pattern

```rust
impl GasCostCalculator<Ethereum> {
    pub async fn calculate_gas_cost_between_blocks(
        &self,
        chain_id: u64,
        from: Address,
        to: Address,
        token: Address,
        start_block: u64,
        end_block: u64,
    ) -> anyhow::Result<GasCostResult> {
        let adapter = EthereumReceiptAdapter;
        self.calculate_gas_cost_with_adapter(
            chain_id,
            from,
            to,
            token,
            start_block,
            end_block,
            &adapter,
        )
        .await
    }
}
```

**Key Learning**: The network-specific implementations become thin wrappers that just instantiate the appropriate adapter and delegate to the generic implementation.

## 7. Educational Benefits for Software Engineering Students

### The Adapter Pattern in Action

1. **Target Interface**: `ReceiptAdapter<N>` trait
2. **Adaptees**: Different network receipt types (`Ethereum::ReceiptResponse`, `Optimism::ReceiptResponse`)
3. **Adapters**: `EthereumReceiptAdapter`, `OptimismReceiptAdapter`
4. **Client**: Generic `GasCostCalculator<N>` implementation

### Rust-Specific Advantages

1. **Zero-Cost Abstractions**: Generics compile to specialized code
2. **Trait Bounds**: Ensure only compatible types can be used
3. **Associated Types**: Clean, ergonomic interfaces
4. **Memory Safety**: No runtime overhead for polymorphism

Rust's type system enables powerful abstractions without sacrificing performance or safety - a perfect example for students learning advanced Rust patterns and software engineering principles. These capabilities are further enhanced by robust libraries like the [Alloy Project](https://alloy.rs/) and its extensions like [op-alloy](https://github.com/alloy-rs/op-alloy) for OP Stack specifics.
