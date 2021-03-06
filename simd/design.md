



This is the design document for SIMD support in GHC that resulted from the October 11, 2011 meeting at GHC HQ. Please see the [top-level GHC SIMD](simd) page for further details.


## Introduction



We are interested in the SIMD vector instructions on current and future generations of CPUs. This includes SSE and AVX on x86/x86-64 and NEON on ARM chips (targets like GPUs or FPGAs are out of scope for this project). These SIMD vector instruction sets are broadly similar in the sense of having relatively short vector registers and operations for various sizes of integer and/or floating point operation. In the details however they have different capabilities and different vector register sizes.



We therefore want a design for SIMD support in GHC that will let us efficiently exploit current vector instructions but a design that is not tied too tightly to one CPU architecture or generation. In particular, it should be possible to write portable Haskell programs that use SIMD vectors.



On the other hand, we want to be able to write programs for maximum efficiency that exploit the native vector sizes, preferably while remaining portable. For example, algorithms on large variable length vectors are in principle agnostic about the size of the primitive vector operations.



Finally, we want a design that is not too difficult or time consuming to implement.


### Use cases



We are mainly interested in scientific / numerical use cases with large arrays / vectors. These are the kinds of use cases that DPH already targets.



In the interests of limiting implementation difficulty, we are prepared initially to sacrifice performance in use cases with small vectors. Examples with lots of small vectors include 3D work where there are lots of 4-element vectors and 4x4 matrices. These tradeoffs show up in our choices about calling conventions and vector memory alignment which are discussed below.



Note: we will need to be clear with users that initially this SIMD work is not suitable for small vectors, just big arrays.


### Existing SIMD instruction sets



Intel and AMD CPUs use the [
SSE family](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) of extensions and, more recently (since Q1 2011), the [
AVX](http://en.wikipedia.org/wiki/Advanced_Vector_Extensions) extensions.  ARM CPUs (Cortex A series) use the [
NEON](http://www.arm.com/products/processors/technologies/neon.php) extensions. PowerPC and SPARC have similar vector extensions. Variations between different families of SIMD extensions and between different family members in one family of extensions include the following:


<table><tr><th>**Register width**</th>
<td>
SSE registers are 128 bits, whereas AVX registers are 256 bits, but they can also still be used as 128 bit registers with old SSE instructions. NEON registers can be used as 64-bit or 128-bit register.
</td></tr>
<tr><th>**Register number**</th>
<td>
SSE sports 8 SIMD registers in the 32-bit i386 instruction set and 16 SIMD registers in the 64-bit x84\_64 instruction set. (AVX still has 16 SIMD registers.) NEON's SIMD registers can be used as 32 64-bit registers or 16 128-bit registers.
</td></tr>
<tr><th>**Register types**</th>
<td>
In the original SSE extension, SIMD registers could only hold 32-bit single-precision floats, whereas SSE2 extend that to include 64-bit double precision floats as well as 8 to 64 bit integral types. The extension from 128 bits to 256 bits in register size only applies to floating-point types in AVX. This is expected to be extended to integer types in AVX2, but in AVX, SIMD operations on integral types can only use the lower 128 bits of the SIMD registers. NEON registers can hold 8 to 64 bit integral types and 32-bit single-precision floats.
</td></tr>
<tr><th>**Alignment requirements**</th>
<td>
SSE requires alignment on 16 byte boundaries. With AVX, it seems that operations on 128 bit SIMD vectors may be unaligned, but operations on 256 bit SIMD vectors needs to be aligned to 32 byte boundaries. NEON suggests to align SIMD vectors with *n*-bit elements to *n*-bit boundaries.
</td></tr></table>


### SIMD/vector support in other compilers



Both GCC and LLVM provide some low-level yet portable support for SIMD vector types and operations.



GCC provides [
vector extensions](http://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html) to C where the programmer may define vector types of a fixed size. The standard C `+`, `-`, `*` etc operators then work on these vector types. GCC implements these operations using whatever hardware support is available. Depending on the requested vector size GCC uses native vector registers and instructions, or synthesises large requested vectors using smaller hardware vectors. For example it can generate code for operating on vectors of 4 doubles by using SSE2 registers and operations which only handle vectors of doubles of size 2.



The LLVM compiler tools targeted by GHC's [LLVM backend](commentary/compiler/backends/llvm) support a generic [
vector type](http://llvm.org/docs/LangRef.html#t_vector) of arbitrary, but fixed length whose elements may be any LLVM scalar type. In addition to three [
vector operations](http://llvm.org/docs/LangRef.html#vectorops), LLVM's operations on scalars are overloaded to work on vector types as well. LLVM compiles operations on vector types to target-specific SIMD instructions, such as those of the SSE, AVX, and NEON instruction set extensions. As the capabilities of the various versions of SSE, AVX, and NEON vary widely, LLVM's code generator maps operations on LLVM's generic vector type to the more limited capabilities of the various hardware targets.


## General plan



We need to implement support for vectors in several layers of GHC + Libraries, from bottom to top:


- code generators (NCG, LLVM)
- Cmm
- Haskell/Core primops
- Some strategy for making use of vector primops, e.g. DPH or Vector lib

### Vector types



We intend to provide vectors of the following basic types:


>
> <table><tr><th> Int8  </th>
> <th> Int16  </th>
> <th> Int32  </th>
> <th> Int64  
> </th></tr>
> <tr><th> Word8 </th>
> <th> Word16 </th>
> <th> Word32 </th>
> <th> Word64 
> </th></tr>
> <tr><th>       </th>
> <th>        </th>
> <th> Float  </th>
> <th> Double 
> </th></tr></table>
>
>

### Fixed and variable sized vectors



The hardware supports only small fixed sized vectors. High level libraries would like to be able to use arbitrary sized vectors. Similar to the design in GCC and LLVM we will provide primitive Haskell types and operations for fixed-size vectors. The task of implementing variable sized vectors in terms of fixed-size vector types and primops is left to the next layer up (DPH, vector lib).



That is, in the core primop layer and down, vector support is only for fixed-size vectors. The fixed sizes will be only powers of 2 and only up to some maximum size. The choice of maximum size should reflect the largest vector size supported by the current range of CPUs (256bit with AVX):


>
> <table><tr><th> types </th>
> <th>        </th>
> <th>        </th>
> <th> vector sizes    
> </th></tr>
> <tr><th> Int8  </th>
> <th> Word8  </th>
> <th>        </th>
> <th> 2, 4, 8, 16, 32 
> </th></tr>
> <tr><th> Int16 </th>
> <th> Word16 </th>
> <th>        </th>
> <th> 2, 4, 8, 16     
> </th></tr>
> <tr><th> Int32 </th>
> <th> Word32 </th>
> <th> Float  </th>
> <th> 2, 4, 8         
> </th></tr>
> <tr><th> Int64 </th>
> <th> Word64 </th>
> <th> Double </th>
> <th> 2, 4            
> </th></tr>
> <tr><th> Int   </th>
> <th> Word   </th>
> <th>        </th>
> <th> 2, 4            
> </th></tr></table>
>
>


In addition, we will support vector types with fixed but architecture-dependent sizes (see below).



We could choose to support larger fixed sizes, or the same maximum size for all types, but there is no strict need to do so.


### Portability and fallbacks



To enable portable Haskell code we will provide the same set of vector types and operations on all architectures. Again this follows the approach taken by GCC and LLVM.



We will rely on fallbacks for the cases where certain types or operations are not supported directly in hardware. In particular we can implement large vectors on machines with only small vector registers. Where there is no vector hardware support at all for a type (e.g. arch with no vectors or 64bit doubles on ARM's NEON) we can implement it using scalar code.



The obvious approach is a transformation to synthesize larger vector types and operations using smaller vector operations or scalar operations. This synthesisation could plausibly be done at the core, Cmm or code generator layers, however the most natural choice would be as a Cmm -\> Cmm transformation. This approach would reduce or eliminate the burden on code generators by allowing them to support only their architecture's native vector sizes and types, or none at all.



Using fallbacks does pose some challenges for a stable/portable ABI, in particular how vector registers should be used in the GHC calling convention. This is discussed in a later section.


### GHC command line flags



We will add machine flags such as `-msse2` and `-mavx`. These tell GHC that it is allowed to make use of the corresponding instruction sets.



For compatibility, the default will remain targeting the base instruction set of the architecture. This is the behaviour of most other compilers. We may also want to add a `-mnative` / `-mdetect` flag that is equivalent to the `-m` flag corresponding to the host machine.


## Code generators



We will not extend the portable C backend to emit vector instructions. It will rely on the higher layers transforming vector operations into scalar operations. The portable C backend is not ABI compatible with the other code generators so there is no concern about vector registers in the calling convention.



The LLVM C library supports vector types and instructions directly. The GHC LLVM backend could be extended to translate vector ops at the Cmm level into LLVM vector ops.



The NCG (native code generator) may need at least minimal support for vector types if vector registers are to be used in the calling convention (see below). If we choose a common calling convention where vectors are passed in registers rather than on the stack then minimal support in the NCG would be necessary if ABI compatibility is to be preserved with the LLVM backend. It is optional whether vector instructions are used to improve performance.


## Cmm layer



The Cmm layer will be extended to represent vector types and operations.



The `CmmType` describes the machine-level type of data. It consists of the "category" of data, along with the `Width` in bits.


```wiki
data CmmType = CmmType CmmCat Width
data Width = ...
data CmmCat     -- "Category" (not exported)
   = GcPtrCat   -- GC pointer
   | BitsCat    -- Non-pointer
   | FloatCat   -- Float
```


The current code distinguishes floats, pointer and non-pointer data. These are distinguished primarily because either they need to be tracked separately (GC pointers) or because they live in special registers on many architectures (floats).



For vectors we add two new categories


```wiki
   | VBitsCat  Multiplicity   -- Non-pointer
   | VFloatCat Multiplicity   -- Float

type Multiplicty = Int
```


We keep vector types separate from scalars, rather than representing scalars as having multiplicty 1. This is to limit disruption to existing code paths and also because it is expected that vectors will often need to be treated differently from scalars. Again we distinguish float from integral types as these may use different classes of registers. There is no need to support vectors of GC pointers.



Vector operations on these machine vector types will be added to the Cmm `MachOp` type, e.g.


```wiki
data MachOp = 
  ...
  | MO_VF_Add Width Multiplicity
```


For example `MO_VF_Add W64 4` represents vector addition on a length-4 vector of 64bit floats.


## Core layer



We need Haskell data types and Haskell primitive operations for fixed size vectors. In some ways this is a harder problem than representing the vector types and opertions at the Cmm level. In particular, at the Haskell type level we cannot easily parametrise on the vector length.



Our design is to provide a family of fixed size vector types and primitive operations, but not to provide any facility to parametrise this family on the vector length.



for width {w} in 8, 16, 32, 64 and "", (empty for native Int\#/Word\# width)

for multiplicity {m} in 2, 4, 8, 16, 32



`type Int`*{w}*`Vec`*{m}*`#`

`type Word`*{w}*`Vec`*{m}*`#`

`type FloatVec`*{m}*`#`

`type DoubleVec`*{m}*`#`



Syntax note: here {m} is meta-syntax, not concrete syntax



Hence we have individual type names with the following naming convention:


>
> <table><tr><th>              </th>
> <th> length 2     </th>
> <th> length 4     </th>
> <th> length 8     </th>
> <th> etc 
> </th></tr>
> <tr><th> native `Int` </th>
> <th> `IntVec2#`   </th>
> <th> `IntVec4#`   </th>
> <th> `IntVec8#`   </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int8`       </th>
> <th> `Int8Vec2#`  </th>
> <th> `Int8Vec4#`  </th>
> <th> `Int8Vec8#`  </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int16`      </th>
> <th> `Int16Vec2#` </th>
> <th> `Int16Vec4#` </th>
> <th> `Int16Vec8#` </th>
> <th> ... 
> </th></tr>
> <tr><th> etc          </th>
> <th> ...          </th>
> <th> ...          </th>
> <th> ...          </th>
> <th> ... 
> </th></tr></table>
>
>


Similarly there will be families of primops:


```wiki
extractInt{w}Vec{m}#  :: Int{w}Vec{m}# -> Int# -> Int{w}#
addInt{w}Vec{m}#      :: Int{w}Vec{m}# -> Int{w}Vec{m}# -> Int{w}Vec{m}#
```


From the point of view of the Haskell namespace for values and types, each member of each of these families is distinct. It is just a naming convention that suggests the relationship.


### Optional extension: extra syntax



We could add a new concrete syntax using `<...>` to suggest a paramater, but have it really still part of the name:


>
> <table><tr><th>              </th>
> <th> length 2       </th>
> <th> length 4       </th>
> <th> length 8       </th>
> <th> etc 
> </th></tr>
> <tr><th> native `Int` </th>
> <th> `IntVec<2>#`   </th>
> <th> `IntVec<4>#`   </th>
> <th> `IntVec<8>#`   </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int8`       </th>
> <th> `Int8Vec<2>#`  </th>
> <th> `Int8Vec<4>#`  </th>
> <th> `Int8Vec<8>#`  </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int16`      </th>
> <th> `Int16Vec<2>#` </th>
> <th> `Int16Vec<4>#` </th>
> <th> `Int16Vec<8>#` </th>
> <th> ... 
> </th></tr>
> <tr><th> etc          </th>
> <th> ...            </th>
> <th> ...            </th>
> <th> ...            </th>
> <th> ... 
> </th></tr></table>
>
>

### Primop generation and representation



Internally in GHC we can take advantage of the obvious parametrisation within the families of primitive types and operations. In particular we extend GHC's `primop.txt.pp` machinery to enable us to describe the family as a whole and to generate the members.



For example, here is some plausible concrete syntax for `primop.txt.pp`:


```wiki
parameter <w, m> Width Multiplicity
  with <w, m> in <8, 2>,<8, 4>,<8, 8>,<8, 16>,<8, 32>,
                 <16,2>,<16,4>,<16,8>,<16,16>,
                 <32,2>,<32,4>,<32,8>,
                 <64,2>,<64,4>
```


Note that we allow non-rectangular combinations of values for the parameters. We declare the range of values along with the parameter so that we do not have to repeat it for every primtype and primop.


```wiki
primtype <w,m> Int<w>Vec<m>#

primop VIntAddOp <w,m> "addInt<w>Vec<m>#" Dyadic
  Int<w>Vec<m># -> Int<w>Vec<m># -> Int<w>Vec<m>#
  {Vector addition}
```


This would generate a family of primops, and an internal representation using the type names declared for the parameters:


```wiki
data PrimOp = ...
   | IntAddOp
   | VIntQuotOp Width Multiplicity
```


It is not yet clear what syntax to achieve the names of the native sized types `Int` and `Word`. Perhaps we should use "", e.g.


```wiki
parameter <w, m> Width Multiplicity
  with <w, m> in <8, 2>,<8, 4>,<8, 8>,<8, 16>,<8, 32>,
                 <16,2>,<16,4>,<16,8>,<16,16>,
                 <32,2>,<32,4>,<32,8>,
                 <64,2>,<64,4>
                 <"",2>,<"",4> 
```

### Optional extension: primitive int sizes



The above mechanism could be used to handle parametrisation between Int8\#, Int16\# etc. Currently these do not exist as primitive types. The types Int8, Int16 etc are implemented as a boxed native-sized Int\# plus narrowing.



Note that while this change is possible and would make things more uniform it is not essential for vector support.



That is we might have:


```wiki
parameter <w> Width
  with <w> in <8>, <16>, <32>, <64>, <"">

primtype Int<w>#

primop   IntAddOp <w>    "addInt<w>#"    Dyadic
   Int<w># -> Int<w># -> Int<w>#
   with commutable = True
```


generating


```wiki
data PrimOp = ...
   | IntAddOp Width
```


We might want some other solution so we can use `+#` as well as `addInt#` since `+8#` as an infix operator doesn't really work.


## Native vector sizes



In addition to various portable fixed size vector types, we will have a portable vector type that is tuned for the hardware vector register size. This is analogous to the existing integer types that GHC supports. We have Int8, Int16, Int32 etc and in addition we have Int, the size of which is machine dependent (either 32 or 64bit).



As with Int, the rationale is efficiency. For algorithms that could work with a variety of primitive vector sizes it will almost always be fastest to use the vector size that matches the hardware vector register size. Clearly it is suboptimal to use a vector size that is smaller than the native size. Using a larger vector is not nearly as bad as using as smaller one, though it does contribute to register pressure.



Without a native sized vector, libraries would be forced to use CPP to pick a good vector size based on the architecture, or to pick a fixed register size that is always at least as big as the native size on all platforms that are likely to be used. The former is annoying and the latter makes less sense as vector sizes on some architectures increase.



Note that the actual size of the native vector size will be fixed per architecture and will not vary based on "sub-architecture" features like SSE vs AVX. We will pick the size to be the maximum of all the sub-architectures. That is we would pick the AVX size for x86-64. The rationale for this is ABI compatibility which is discussed below. In this respect, the IntVec\# is like Int\#, the size of both is crucial for the ABI and is determined by the target platform/architecture.



So we extend our family of vector types with:


>
> <table><tr><th>              </th>
> <th> native length </th>
> <th> length 2     </th>
> <th> length 4     </th>
> <th> length 8     </th>
> <th> etc 
> </th></tr>
> <tr><th> native `Int` </th>
> <th> `IntVec#`     </th>
> <th> `IntVec2#`   </th>
> <th> `IntVec4#`   </th>
> <th> `IntVec8#`   </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int8`       </th>
> <th> `Int8Vec#`    </th>
> <th> `Int8Vec2#`  </th>
> <th> `Int8Vec4#`  </th>
> <th> `Int8Vec8#`  </th>
> <th> ... 
> </th></tr>
> <tr><th> `Int16`      </th>
> <th> `Int16Vec#`   </th>
> <th> `Int16Vec2#` </th>
> <th> `Int16Vec4#` </th>
> <th> `Int16Vec8#` </th>
> <th> ... 
> </th></tr>
> <tr><th> etc          </th>
> <th> ...           </th>
> <th> ...          </th>
> <th> ...          </th>
> <th> ...          </th>
> <th> ... 
> </th></tr></table>
>
>


and there are some top level constants describing the vector size so as to enable their portable use


```wiki
intVecSize, int8VecSize, int16VecSize, int32VecSize, int64VecSize :: Int
wordVecSize, word8VecSize, word16VecSize, word32VecSize, word64VecSize :: Int
floatVecSize, doubleVecSize :: Int
```


Note that these constants are of type Int since top level values of type Int\# are not currently supported. This should not be a problem as they should always get inlined and unboxed where it matters.



The native-sized vector types are distinct types from the explicit-sized vector types, not type aliases for the corresponding explicit-sized vector. This is to support and encourage portable code.


## Vector operations



The following operations on vectors will be supported. They will need to be implemented at the Haskell/core primop layer, Cmm `MachOp` layer and optional support in the code generators.



In the following, `<t>` ranges over `Int<w>`, `Word<w>`, `Float`, `Double`.



Loading and storing vectors in arrays, `ByteArray#` and raw `Addr#`


```wiki
indexInt<w>Vec<m>Array#  :: ByteArray# -> Int# -> Int<w>Vec<m>#
indexWord<w>Vec<m>Array# :: ByteArray# -> Int# -> Word<w>Vec<m>#
indexFloatVec<m>Array#   :: ByteArray# -> Int# -> FloatVec<m>#
indexDoubleVec<m>Array#  :: ByteArray# -> Int# -> DoubleVec<m>#

readInt<w>Vec<m>Array#  :: MutableByteArray# d -> Int# -> State# d -> (# State# d, Int<w>Vec<m>#  #)
readWord<w>Vec<m>Array# :: MutableByteArray# d -> Int# -> State# d -> (# State# d, Word<w>Vec<m># #)
readFloatVec<m>Array#   :: MutableByteArray# d -> Int# -> State# d -> (# State# d, FloatVec<m>#   #)
readDoubleVec<m>Array#  :: MutableByteArray# d -> Int# -> State# d -> (# State# d, DoubleVec<m>#  #)

writeInt<w>Vec<m>Array#  :: MutableByteArray# d -> Int# -> Int<w>Vec<m>#  -> State# d -> State# d
writeWord<w>Vec<m>Array# :: MutableByteArray# d -> Int# -> Word<w>Vec<m># -> State# d -> State# d
writeFloatVec<m>Array#   :: MutableByteArray# d -> Int# -> FloatVec<m>#   -> State# d -> State# d
writeDoubleVec<m>Array#  :: MutableByteArray# d -> Int# -> DoubleVec<m>#  -> State# d -> State# d

readInt<w>Vec<m>OffAddr#  :: Addr# -> Int# -> State# d -> (# State# d, Int<w>Vec<m># #)
readWord<w>Vec<m>OffAddr# :: Addr# -> Int# -> State# d -> (# State# d, Word<w>Vec<m># #)
readFloatVec<m>OffAddr#   :: Addr# -> Int# -> State# d -> (# State# d, FloatVec<m># #)
readDoubleVec<m>OffAddr#  :: Addr# -> Int# -> State# d -> (# State# d, DoubleVec<m># #)

writeInt<w>Vec<m>OffAddr#  :: Addr# -> Int# -> Int<w>Vec<m>#  -> State# d -> State# d
writeWord<w>Vec<m>OffAddr# :: Addr# -> Int# -> Word<w>Vec<m># -> State# d -> State# d
writeFloatVec<m>OffAddr#   :: Addr# -> Int# -> FloatVec<m>#   -> State# d -> State# d
writeDoubleVec<m>OffAddr#  :: Addr# -> Int# -> DoubleVec<m>#  -> State# d -> State# d
```


Extracting and inserting vector elements:


```wiki
extractInt<w>Vec<m>#   :: Int<w>Vec<m>#  -> Int# -> Int#
extractWord<w>Vec<m>#  :: Word<w>Vec<m># -> Int# -> Word#
extractFloatVec<m>#    :: FloatVec<m>#   -> Int# -> Float#
extractDoubleVec<m>#   :: DoubleVec<m>#  -> Int# -> Double#
```

```wiki
insertInt<w>Vec<m>#   :: Int<w>Vec<m>#  -> Int# -> Int#    -> Int<w>Vec<m>#
insertWord<w>Vec<m>#  :: Word<w>Vec<m># -> Int# -> Word#   -> Word<w>Vec<m>#
insertFloatVec#       :: FloatVec<m>#   -> Int# -> Float#  -> FloatVec<m>#
insertDoubleVec#      :: DoubleVec<m>#  -> Int# -> Double# -> DoubleVec<m>#
```


Duplicating a scalar to a vector:


```wiki
replicateToInt<w>Vec<m>#  :: Int<w>Vec<m>#  -> Int#    -> Int<w>Vec<m>#
replicateToWord<w>Vec<m># :: Word<w>Vec<m># -> Word#   -> Word<w>Vec<m>#
replicateToFloatVec#      :: FloatVec<m>#   -> Float#  -> FloatVec<m>#
replicateToDoubleVec#     :: DoubleVec<m>#  -> Double# -> DoubleVec<m>#
```


Vector shuffle:


```wiki
shuffle<t>Vec<m>ToVec<m'> :: <t>Vec<m># -> Int32Vec<m'># -> <t>Vec<m'>#
```


For the fixed size vectors (not native size) we may also want to add pack/unpack functions like:


```wiki
unpackInt<w>Vec4# :: Int<w>Vec4# -> (# Int#, Int#, Int#, Int# #)
packInt<w>Vec4#   :: (# Int#, Int#, Int#, Int# #) -> Int<w>Vec4#
```


Arithmetic operations:


```wiki
plus<t>Vec<m>#, minus<t>Vec<m>#,
times<t>Vec<m>#, quot<t>Vec<m>#, rem<t>Vec<m># :: <t>Vec<m># -> <t>Vec<m># -> <t>Vec<m>#

negate<t>Vec<m># :: <t>Vec<m># -> <t>Vec<m>#
```


Logic operations:


```wiki
andInt<w>Vec<m>#, orInt<w>Vec<m>#, xorInt<w>Vec<m>#    :: Int<w>Vec<m>#  -> Int<w>Vec<m>#  -> Int<w>Vec<m>#
andWord<w>Vec<m>#, orWord<w>Vec<m>#, xorWord<w>Vec<m># :: Word<w>Vec<m># -> Word<w>Vec<m># -> Word<w>Vec<m>#

notInt<w>Vec<m>#  :: Int<w>Vec<m>#  -> Int<w>Vec<m>#
notWord<w>Vec<m># :: Word<w>Vec<m># -> Word<w>Vec<m>#

shiftLInt<w>Vec<m>#,  shiftRAInt<w>Vec<m>#  :: Int<w>Vec<m>#  -> Word# -> Int<w>Vec<m>#
ShiftLWord<w>Vec<m>#, ShiftRLWord<w>Vec<m># :: Word<w>Vec<m># -> Word# -> Word<w>Vec<m>#
```


Comparison:


```wiki
cmp<eq,ne,gt,gt,lt,le>Int<w>Vec<m>#  :: Int<w>Vec<m>#  -> Int<w>Vec<m>#  -> Word<w>Vec<m>#
cmp<eq,ne,gt,gt,lt,le>Word<w>Vec<m># :: Word<w>Vec<m># -> Word<w>Vec<m># -> Word<w>Vec<m>#
```


Note that LLVM does not yet support the comparison operations (See the comment at the end of the documentation for the [
icmp](http://llvm.org/docs/LangRef.html#i_icmp) instruction, for example).



Integer width narrow/widen operations:


```wiki
narrowInt<w>To<w'>Vec<m>#  :: Int<w>Vec<m># -> Int<w'>Vec<m>#     -- for w' < w
narrowWord<w>To<w'>Vec<m># :: Word<w>Vec<m># -> Word<w'>Vec<m>#   -- for w' < w

widenInt<w>To<w'>Vec<m>#   :: Int<w>Vec<m># -> Int<w'>Vec<m>#     -- for w' > w
widenWord<w>To<w'>Vec<m>#  :: Word<w>Vec<m># -> Word<w'>Vec<m>#   -- for w' > w
```


Note: LLVM calls these truncate and extend (signed extend or unsigned extend)



Floating point conversion:


```wiki
narrowDoubleToFloatVec<m>#  :: DoubleVec<m># -> FloatVec<m>#
widenFloatToDoubleVec<m>#   :: FloatVec<m>#  -> DoubleVec<m>#

roundFloatToInt32Vec<m>     :: FloatVec<m>#  -> Int32Vec<m>#
roundFloatToInt64Vec<m>     :: FloatVec<m>#  -> Int64Vec<m>#
roundDoubleToInt32Vec<m>    :: DoubleVec<m># -> Int32Vec<m>#
roundDoubleToInt64Vec<m>    :: DoubleVec<m># -> Int64Vec<m>#

truncateFloatToInt32Vec<m>  :: FloatVec<m>#  -> Int32Vec<m>#
truncateFloatToInt64Vec<m>  :: FloatVec<m>#  -> Int64Vec<m>#
truncateDoubleToInt32Vec<m> :: DoubleVec<m># -> Int32Vec<m>#
truncateDoubleToInt64Vec<m> :: DoubleVec<m># -> Int64Vec<m>#

promoteInt32ToFloatVec<m>   :: Int32Vec<m># -> FloatVec<m>#
promoteInt64ToFloatVec<m>   :: Int64Vec<m># -> FloatVec<m>#
promoteInt32ToDoubleVec<m>  :: Int32Vec<m># -> DoubleVec<m>#
promoteInt64ToDoubleVec<m>  :: Int64Vec<m># -> DoubleVec<m>#
```


TODO Should consider:


- vector constants, at least at Cmm level
- replicating a scalar to a vector
- FMA: fused multiply add, this is supported by NEON and AVX however software fallback may not be possible with the same precision. Tricky.
- SSE/AVX also suppports a bunch of interesting things:

  - add/sub/mul/div of vector by a scalar
  - reciprocal, square root, reciprocal of square root
  - permute, shuffle, "blend", masked moves.
  - abs
  - min, max within a vector
  - average
  - horizontal add/sub
  - shift whole vector left/right by n bytes
  - and not logical op
  - gather (but not scatter) of 32, 64bit int and fp from memory (base + vector of offsets)

### Int/Word size wrinkle



Note that there is a wrinkle with the 32 and 64 bit int and word types. For example, the types for the extract functions should be:


```wiki
extractInt32Vec<m>#  :: Int32Vec#  -> Int# -> INT32
extractInt64Vec<m>#  :: Int64Vec#  -> Int# -> INT64
extractWord32Vec<m># :: Word32Vec# -> Int# -> WORD32
extractWord64Vec<m># :: Word64Vec# -> Int# -> WORD64
```


where `INT32`, `INT64`, `INT64`, `WORD64` are CPP macros that expand in a arch-dependent way to the types Int\#/Int64\# and Word\#/Word64\#.



To describe this in the primop definition we might want something like:


```wiki
primop   IntAddOp <w,m,t>    "extractWord<w>Vec<m>#"    Dyadic
  Word<w>Vec<m># -> Int# -> <t>
  with <w, m, t> in <8, 2,Word#>,<8, 4,Word#>,<8, 8,Word#>,<8, 16,Word#>,<8, 32,Word#>,
                    <16,2,Word#>,<16,4,Word#>,<16,8,Word#>,<16,16,Word#>,
                    <32,2,WORD32>,<32,4,WORD32>,<32,8,WORD32>,
                    <64,2,WORD64>,<64,4,WORD64>
                    <"",2,WORD>,<"",4,WORD> 
```


To iron out this wrinkle we would need the whole family of primitve types: Int8\#, Int16\#, Int32\# etc whereas currently only the native register sized Int\# type is provided, plus a primitive Int64\# type is provided on 32bit systems.


## Data Parallel Haskell layer



In [
DPH](http://www.haskell.org/haskellwiki/GHC/Data_Parallel_Haskell), we will use the new SIMD instructions by suitably modifying the definition of the lifted versions of arithmetic and other operations that we would like to accelerate. These lifted operations are defined in the `dph-common` package and made accessible to the vectoriser via [VECTORISE pragmas](data-parallel/vect-pragma). Many of them currently use `VECTORISE SCALAR` pragmas, such as


```wiki
(+) :: Int -> Int -> Int
(+) = (P.+)
{-# VECTORISE SCALAR (+) #-}
```


We could define them more verbosely using a plain `VECTORISE` pragma, but might instead like to extend `VECTORISE SCALAR` or introduce a variant.



**NB:** The use of SIMD instructions interferes with vectorisation avoidance for scalar subcomputations. Code that avoids vectorisation also avoids the use of SIMD instructions. We would like to use SIMD instructions, but still avoid full-scale vectorisation. This should be possible, but it is not immediately clear how to realise it (elegantly).


## ABIs and calling conventions



For each CPU architecture GHC has a calling convention that it uses for all Haskell function calls. The calling convention specifies where function arguments and results are placed in registers and on the stack. Adhering to the calling convention is necessary for correctness. Code compiled using different calling conventions should not be linked together. Note that currently the LLVM and NCG code generators adhere to the same ABI.



The calling convention needs to be extended to take into account the primitive vector types. We have to decide if vectors should be passed in registers or on the stack and how to handle vectors that do not match the native vector register size.



For efficiency it is highly desirable to make use of vector registers in the calling convention. This can be significantly quicker than copying vectors to and from the stack.



Within the same overall CPU architecture, there are several sub-architectures with different vector capabilities and in particular different vector sizes. The x86-64 architecture supports SSE2 vectors as a baseline which includes pairs of doubles, but the AVX extension doubles the size of the vector registers. Ideally when compiling for AVX we would make use of the larger AVX vectors, including passing the larger vectors in registers.



This poses a major challenge: we want to make use of large vectors when possible but we would also like to maintain some degree of ABI compatibility.


### Alternative design: separate ABIs



It is worth briefly exploring the option of abandoning ABI compatibility. We could declare that we have two ABIs on x86-64, the baseline SSE ABI and the AVX ABI. We would further declare that to generate AVX code you must build all of your libraries using AVX. Essentially this would mean having two complete sets of libraries, or perhaps simply two instances of GHC, each with their own libraries. While this would work and may be satisfactory when speed is all that matters, it would not encourage use of vectors more generally. In practice haskell.org and linux distributions would have to distribute the more compatible SSE build so that in many cases even users with AVX hardware would be using GHC installations that make no use of AVX code. On x86 the situation could be even worse since the baseline x86 sub-architecture used by many linux distributions does not include even SSE2. In addition it is wasteful to have two instances of libraries when most libraries do not use vectors at all.


### Selected design: mixed ABIs using worker/wrapper



It it worth exploring options for making use of AVX without having to force all code to be recompiled. Ideally the base package would not need to be recompiled at all and perhaps only have packages like vector recompiled to take advantage of AVX.



Consider the situation where we have two modules `Lib.hs` and `App.hs` where `App` imports `Lib`. The `Lib` module exports:


```wiki
f :: DoubleVec4# -> Int
g :: (DoubleVec4# -> a) -> a
```


which are used by App. We compile:


```wiki
ghc -msse2 Lib.hs
ghc -mavx  App.hs.
```


There are two cases to consider:


- if the function being called has an unfolding exported from `Lib` then that unfolding can be compiled in the context of App and can make use of AVX instructions
- alternatively we are dealing with object code for the function which follows a certain ABI


Notice that not only do we need to be careful to call `f` and `g` using the right calling convention, but in the case of `g`, the function that we pass as its argument must also follow the calling convention that `g` will call it with.



Our solution is to take a worker/wrapper approach. We will split each function into a wrapper that uses a lowest common denominator calling convention and a worker that uses the best calling convention for the target sub-architecture. The simplest lowest common denominator calling convention is to pass all vectors on the stack, while the fast calling convention will use SSE2 or AVX registers.



For `App` calling `Lib.f` we start with a call to the wrapper, this can be inlined to a call to the worker at which point we discover that the calling convention will use SSE2 registers. For `App` calling `Lib.g` with a locally defined `h`, we would pass the wrapper for `h` to `g` and since we assume we have no unfolding for `g` then this is how it remains: at runtime `g` will call `h` through the wrapper for `h` and so will use the lowest common denominator calling convention.



We might be concerned with the reverse situation where we have A and B, with A importing B:


```wiki
ghc -mavx  B.hs.
ghc -msse2 A.hs
```


That is, a module compiled with SSE2 that imports a module that was compiled with AVX. How can we call functions using AVX registers if we are only targeting SSE2? There are two design options:


- One option is to note that since we will be using AVX instructions at runtime when we call the functions in B, and hence it is legitimate to use AVX instructions in A also, at least for the calling convention.
- The other is to avoid generating AVX instructions at all, even for the calling convention, in which case it is essential to avoid inlining the wrapper function since this exposes the worker that uses the AVX calling convention.


While the first option is in some ways simpler, it also implies that all ABI-compatible code generators can produce at least some vector instructions. In particular it requires data-movement instructions to be supported. If however we wish to completely avoid implementing any vector support in the NCG backend then we must take the second approach.



For the second approach we would need to add an extra architecture flag and check to inlining annotations. There are already several conditions that are checked prior to inlining (e.g. phase checks), this would add an additional check.


### Optional extension: compiling code for multiple sub-architectures



If we have support for arch-conditional inlining, we in future may want to extend the idea to allow inlining to one of a number of arch-specific implementations.



Consider a hypothetical function in a core library that uses vectors but that is too large to be a candidate for inlining. We have to ship core libraries compiled for the base architecture. Hence the function from the core lib will not be compiled to use AVX. Another possibility is to generate several copies of the function worker, compiled for different sub-archtectires. Then when the function is called in another module compiled with -mavx we would like to call the AVX worker. This could be achieved by arch-conditional inlining or rules.



This option should only be considered if we expect to have functions in core libs that are above the inlining threshold. This would probably not be the case for ghc-prim and base. It may however make sense for the vector library should that become part of the standard platform and hence typically shipped to users as a pre-compiled binary.


### Types for calling conventions



One of GHC's clever design tricks is that the type of a function in core determines its calling convention. A function in core that accepts an Int is different to one that accepts an Int\#. The use of two different types, Int and Int\# let us talk about the transformation and lets us write code in core for the wrapper function that converts from Int to Int\#.



If we are to take a worker wrapper approach with calling conventions for vectors then we would do well to use types to distinguish the common and special calling conventions. For example, we could define sub-architecture specific types:


```wiki
FloatSseVec4#
DoubleSseVec2#

FloatAvxVec8#
DoubleAvxVec4#
```


We would also need some suitable primitive conversion operations


```wiki
toSseVec4#   :: FloatVec4# -> FloatSseVec4#
fromSseVec4# :: FloatSseVec4# -> FloatVec4#
etc
```


Then we can describe the types for the worker and wrapper for our function


```wiki
f :: DoubleVec4# -> Int
```


This remains the type of the wrapper, which also is still called f. If we compile the module with -msse2 or -mavx then we would get workers with the respective types:


```wiki
f_worker :: (# DoubleSseVec2#, DoubleSseVec2# #) -> Int
```


or


```wiki
f_worker :: DoubleAvxVec4# -> Int
```


Note that in the SSE2 case we have to synthesize a vector of length 4 using native vectors of length 2.



Now it is clearer what the calling convention of the workers are. What is the calling convention of the wrapper?


```wiki
f :: DoubleVec4# -> Int
```


We have said that this is the lowest common denominator calling convention. The simplest is passing vectors on the stack. This has the advantage of not requiring vector support in the NCG.


### Calling convention and performance



The mixed ABI approach using worker/wrapper trades off some performance for convenience and compatibility. Why do we think the tradeoff is reasonable?



In the case of ordinary unboxed types, Int/Int\# etc, this approach is very effective. It is only when calling unknown functions, e.g. higher order functions that are not inlined that we would be calling the wrapper and using the slower calling convention. This is unlikely for high performance numeric code.


### Optional extension: faster common calling convention



On x86-64 we know we always have SSE2 available, so we might want to use that in our lowest common denominator calling convention. It would of course require support for vector data movement instructions in the NCG.


### Alternative design: only machine-specific ABI types



With the above extension to use vectors registers in the common calling convention, it would make sense to say that in fact the wrapper `f` has type:


```wiki
f :: (# DoubleSseVec2#, DoubleSseVec2# #) -> Int
```


This is a plausible design, but it is not necessary to go this way. We can simply declare types like `DoubleVec4#` to have a particular calling convention without forcing it to be rewritten in terms of machine-specific types in core.



But it would be plausible to say that types like `DoubleVec4#` are ephemeral, having no ABI and must be rewritten by a core -\> core pass to use machine-specific types with an associated ABI.


### Memory alignment for vectors



Many CPUs that support vectors have strict alignment requirements, e.g. that 16 byte vectors must be aligned on 16byte boundaries. On some architectures the requirements are not strict but there may be a performance penalty, or alternative instruction may be required to load unaligned vectors. For example AVX has special instructions for unaligned loads and stores but Intel estimates a [
20% performance loss](http://software.intel.com/en-us/articles/practical-intel-avx-optimization-on-2nd-generation-intel-core-processors/).



LLVM has primitives that can load and store vectors from unaligned memory locations, which (presumably) compile to either aligned vector instructions if the architecture has them, or non-vector instructions if not.  So alignment of vectors in memory is optional, and we can make an independent choice about whether we align stored vectors


- on the stack
- in a heap closure
- in an array


**Alignment in arrays.** Arrays of vectors are clearly the most important case, so we must support allocation of aligned unboxed arrays.  Indeed GHC already does support *pinned* arrays of unboxed data, and any array larger than about 3k is implicitly pinned. Supporting unpinned arrays would be somewhat more difficult, requiring some GC support to keep the objects aligned when copying them, and requiring that the "slop" be filled in some cases, but it could be done.



**Alignment on the stack.** Aligning the stack could be done either by ensuring that all stack allocation is a multiple of the alignment (thus possibly wasting a lot of stack space), or by adding extra frames to fill the slop when allocating a frame that needs alignment.  Neither option is particularly attractive.  We propose to use unaligned access to vectors stored on the stack for the time being, and revisit this decision if it is found to be a performance bottleneck later.



**Alignment in the heap.**  Again, while we could arrange the correct alignment for vectors stored in heap objects, it would be painful to do so, requiring code generator support and GC support.  We propose not to do this, at least for the time being, and to use unaligned loads and stores for vectors in heap objects.


### ABI summary



The size of the native-sized vectors `IntVec#`, `DoubleVec#` etc correspond to the maximum size for any sub-architecture, e.g. AVX on x86-64.



The ordinary `IntVec#`, `Int32Vec4#` etc types correspond to the slow compatible calling convention which passes all vectors on the stack. These vectors must all have their obvious strict alignment. For example `Int32Vec4#` is 16 bytes large and has 16 byte alignment.



Extra machine-specific types `DoubleSseVec2#`, `FloatAvxVec8#`, `FloatNeonVec4#` etc correspond to the fast calling convention which use the corresponding vector registers. These have the alignment requirements imposed by the hardware. The machine-specific types need not be exposed but it is also plausible to do so.



We will use worker/wrapper to convert between the common types and the machine-specific types.



Initially, to avoid implementing vector data-movement instructions in the NCG, we will add arch-conditional inlining of the wrapper functions.



If later on we add vector data-movement instructions to the NCG, then the arch-conditional inlining of the wrapper functions can be discarded and the compatible calling convention could be changed to make use of any vector registers in the base architecture (e.g. SSE2 on x86-64).


## See also


- [
  Blog article about Larrabee and Nvidia, MIMD vs. SIMD](http://perilsofparallel.blogspot.com/2008/09/larrabee-vs-nvidia-mimd-vs-simd.html)
- [SIMD LLVM:](simd/implementation/llvm) A previous (LLVM-specific) iteration of this SIMD proposal.
- [Vector Computing:](simd/implementation/old)  A previous proposal to make use of x86 SSE in GHC.
