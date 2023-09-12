# circuit-tools
A lightweight SDK for Halo2 frontend circuit.
## Features
### Logic: Constraint Builder
Leverage Rust's macro to construct conditional constraints based on execution branching. Build all constraints in on step with Halo2's gate.
```rust
let mut cb: ConstraintBuilder<F, TestCellType> =  ConstraintBuilder::new(4,  None, None);

meta.create_gate("Test", |meta| {
    circuit!([meta, cb], {
        ifx!(f!(q_enable) => {
            ifx!{a!(a) => {
                require!(a!(res) => a!(b) + f!(c)); 
            } elsex {
                require!(a!(res) => a!(b) + c!(r));  
            }};
        });
    });
    cb.build_constraints()
});
```
### Data: Cell Manager
Manage and allocate cells on demand based on user specified `CellType`. Decrease amount of columns space wasted, enable degree reduction and lookup automation.
```rust
let mut cm = CellManager::new(5, 0);
cm.add_columns(meta, &mut cb, TestCellType::StoragePhase1, 1, false, 1);
cm.add_columns(meta, &mut cb, TestCellType::Lookup, 2, false, 1);

// Allocation
let a = cb.query_default();
let b = cb.query_default();
let c = cb.query_cell_with_type(TestCellType::StoragePhase2);

// Computation
require!(c.expr() => a.expr() + b.expr() * challenge.expr());
// Lookup
require!((a.expr(), b.expr()) =>> @TestCellType::Lookup);

```
### Memory: Dynamic Lookup
Abstraction over using dynamic lookup to prove RW access in RAM. Stores to dynamic table to resemble WRITE and add lookups to represent READ. Lookup operation `(idx, r0, r1) => (rdx, w0, w1)` returns true if and only if the prover access the writen data at anticipated position correctly.
```rust
let mut memory = Memory::new();
memory.add_rw(meta, &mut cb.base, &mut state_cm, MyCellType::Mem1, 1);
memory.add_rw(meta, &mut cb.base, &mut state_cm, MyCellType::Mem2, 1);

let register1 = memory[MyCellType::Mem1];
let register2 = memory[MyCellType::Mem2];

register1.store(cb, &[a0, b0]);
register2.store(cb, &[c0, d9, 123.expr()]);

// ... long time later ...
register2.load(cb, &[c1, d1, 123.expr()]);
register1.load(cb, &[a0, b1]);

```
### Assignment: Cached Region
Used to backtrack the intermediate cells queried in by Constrinat Builder during degree reduction. After `a * b * (c + d)` being split into `x = a * b` and `y = c + d`, the system need to account for the assignment of intermediate cells `x` and `y` while the prover only need to assign `a, b, c, d` based on the execution trace. Hence, Cached Region iterates over all stored expressions and recursively find the prover's assignment to calculate the intermediate values.
```rust
layouter.get_challenge(self.rand).map(|r| r1 = r);
layouter.assign_region(
    || "Test", 
    |mut region| {
        let mut region = CachedRegion::new(&mut region, 0.scalar());
        region.push_region(0, 0);

        let (a, b, c, d,  e) = &self.cells;
        assign!(&mut region, a, 0 => 1.scalar())?;
        assign!(&mut region, b, 0 => 2.scalar())?;
        assign!(&mut region, c, 0 => 3.scalar())?;
        assign!(&mut region, d, 0 => 4.scalar())?;
        region.assign_stored_expressions(&self.cb, &[r0])?;
        Ok(())
    }
)
```
## Workflow
Must specify cell type that implement trait `CellType` to satisfy the generic argument of `ConstraintBuilder<F, C: CellType>`. Can also declear a `TableType` to tag corresponding table for column-to-table lookups.
```rust
pub enum TableTag {
    Fixed,
    Dyn
}
pub enum TestCellType {
    StoragePhase1,
    Lookup,
}
impl CellType for TestCellType{
    type TableType = TableTag;

    fn lookup_table_type(&self) -> Option<Self::TableType> {
        match self {
            TestCellType::Lookup => Some(TableTag::Fixed),
            _ => None,
        }
    }
    fn byte_type() -> Option<Self> {None}
    fn create_type(_id: usize) -> Self {unreachable!()}
    fn storage_for_phase(phase: u8) -> Self {
        match phase {
            1 => Self::StoragePhase1,
            _ => unreachable!()
        }
    }
}
impl Default for TestCellType {
    fn default() -> Self {Self::StoragePhase1}
}
```
Initialize the Constraint Builder and Cell Manager with optional challenge that's used in RLC to conbime multi-columns lookups. The Cell Manager needs a max height and this is usually the height of your Halo2 gate; a offset is also needed to query cell from columns. Offset should be set to 0 in usual case. Load fixed table in to Constraint Builder with corresponding tag and initialized columns with the cell manager with `(cell_type: MyCellType, phase: u8, permuation: bool, num: usize)`.
```rust
let mut cm = CellManager::new(5, 0);
let mut cb: ConstraintBuilder<F, TestCellType> =  ConstraintBuilder::new(4,  Some(cm), Some(challenge));
cb.load_table(meta, TableTag::Fixed, &fixed_table);
cm.add_columns(meta, &mut cb, TestCellType::StoragePhase1, 1, true, 1);
cm.add_columns(meta, &mut cb, TestCellType::Lookup, 2, false, 1);
```
In Halo2's gate API, use macro to config your circuit! Remember to call `build_constraints()` to return the constraints expression for the gate, finally calling `build_lookups(meta)` that turned into `meta.lookup_any(..) in Halo2.
```rust
meta.create_gate("Test", |meta| {
    circuit!([meta, cb], {
        // ... circuit ...
    });
    cb.build_constraints()
});
cb.build_lookups(meta);
```
Finally in assingment, initialize `CachedRegion` as demonstrated above and use it to assign values.
