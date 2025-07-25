`TypeLayoutCompatibleWith<AddressSpace>` is a `TypeLayoutRecipe` with the additional
guarantee that the `TypeLayout` it produces is compatible with the specified `AddressSpace`.

The address spaces are language specific. For example, for WGSL there are two address spaces:
[`WgslStorage`] and [`WgslUniform`].

For a `TypeLayoutRecipe` to be `TypeLayoutCompatibleWith<AddressSpace>` requires that
- the recipe is a **valid** (for example the recipe of a struct with `Repr::Packed` and custom field align attribute is not valid, because custom field align attributes are not supported by `Repr::Packed`)
- the type layout it produces **satisfies the layout requirements** of the address space
- the recipe is **representable** in the target language

To be representable in a language means that the type layout recipe can be expressed in the
language's type system:
1. all types in the recipe can be expressed in the target language (for example `bool` or `PackedVector` can't be expressed in wgsl)
2. the available layout algorithms in the target language can produce the same layout as the one produced by the recipe
3. support for the custom attributes the recipe uses, such as `#[align(N)]` and `#[size(N)]`.
   Custom attributes may be rejected by the target language itself (NotRepresentable error)
   or by the layout algorithms specified in the recipe (InvalidRecipe error).

For example for wgsl we have
1. PackedVector can be part of a recipe, but can not be expressed in wgsl,
   so a recipe containing a PackedVector is not representable in wgsl.
2. Wgsl has only one layout algorithm (`Repr::Wgsl`) - there is no choice between std140 and std430
   like in glsl - so to be representable in wgsl the type layout produced by the recipe
   has to be the same as the one produced by the same recipe but using exclusively the `Repr::Wgsl` layout algorithm instead of the layout algorithms specified in the recipe.
3. Wgsl only supports custom struct field attributes `#[align(N)]` and `#[size(N)]` currently.

Currently the errors reflect the above description
```rust
pub enum AddressSpaceError {
    // error that can occur when cooking a `TypeLayoutRecipe`
    InvalidRecipe(InvalidRecipe),
    RequirementsNotSatisfied(RequirementsNotSatisfied),
    NotRepresentable(NotRepresentable),
}

pub enum RequirementsNotSatisfied {
    // wgsl doesn't support unsized uniform buffers
    MustBeSized(TypeLayoutRecipe, AddressSpaceEnum),
    LayoutError(LayoutError),
    UnknownLayoutError(TypeLayoutRecipe, AddressSpaceEnum),
}

pub enum NotRepresentable {
    // enum RecipeContains { CustomAlign, CustomSize, PackedVector }
    MayNotContain(TypeLayoutRecipe, AddressSpaceEnum, RecipeContains),
    LayoutError(LayoutError),
    UnknownLayoutError(TypeLayoutRecipe, AddressSpaceEnum),
}

pub struct LayoutError {
    recipe: TypeLayoutRecipe,
    address_space: AddressSpaceEnum,
    // LayoutMismatch is from `type_layout/eq.rs`
    mismatch: LayoutMismatch,
    colored: bool,
}

```
