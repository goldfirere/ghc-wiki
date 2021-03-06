



**This page is obsolete**. Please see the [top-level SIMD project page](simd) for further details.


# Using SIMD Instructions via the LLVM Backend



The LLVM compiler tools targeted by GHC's [LLVM backend](commentary/compiler/backends/llvm) support a generic [
vector type](http://llvm.org/docs/LangRef.html#t_vector) of arbitrary, but fixed length whose elements may be any LLVM scalar type. In addition to three [
vector operations](http://llvm.org/docs/LangRef.html#vectorops), LLVM's operations on scalars are overloaded to work on vector types as well. LLVM compiles operations on vector types to target-specific SIMD instructions, such as those of the SSE, AVX, and NEON instruction set extensions. As the capabilities of the various versions of SSE, AVX, and NEON vary widely, LLVM's code generator maps operations on LLVM's generic vector type to the more limited capabilities of the various hardware targets.



The SIMD vector extension to GHC proposed here maps to LLVM's vector type in a straight forward manner, which in turn enables us to target a wide range of hardware capabilities. However, GHC's native code generator will simply map SIMD vector operations to ordinary scalar code (in order to avoid having to deal with the complexities of SSE, AVX, NEON, etc).


## Variations in the most widely used SIMD extensions



Intel and AMD CPUs use the [
SSE family](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) of extensions and, more recently (since Q1 2011), the [
AVX](http://en.wikipedia.org/wiki/Advanced_Vector_Extensions) extensions.  ARM CPUs (Cortex A series) use the [
NEON](http://www.arm.com/products/processors/technologies/neon.php) extensions. Variations between different families of SIMD extensions and between different family members in one family of extensions include the following:


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


### Consequences



While LLVM mostly shields us from these differences, we need to implement traversals of unboxed Haskell arrays as strided loops, where the stride corresponds to the SIMD vector length. LLVM enables us to use a stride that is not the same as that of the SIMD register width of the target architecture, it makes sense to use the target vector width already in the Haskell code. Why? If the Haskell stride is smaller than the SIMD registers, we do not fully exploit all available parallelism. And if the Haskell stride is longer than the SIMD registers, we produce less efficient code for the excess portion at the end of an array whose length is not a multiple of the stride length and force LLVM to expand individual vector operations to multiple target instructions.


### Type-dependent vector sizes



From the above comparison of the capabilities of the various SIMD extensions, it follows that for every scalar data type and for each target architecture, there is a maximum SIMD vector length supported for that data type. Moreover, it appears reasonable to always use the longest vector size that is supported. (Programming guides often recommend to use smaller SIMD vectors if the code cannot guarantee that the data is properly aligned without possibly having to re-align data. This is no issue for us as we can impose suitable alignment constraints on Haskell arrays that we want to process with SIMD instructions.)



As a consequence, we will support exactly one vector length for each scalar data type and this vector length is dependent on the target architecture. Hence, it is sufficient to have one vector type for each scalar data type (that we want to process with SIMD instructions). Concretely, for types, such as `Int#`, `Float#`, and so on, we will provide SIMD vector types `IntVec#`, `FloatVec#`, etc. In addition, we have constants `intVecLen`, `floatVecLen`, and so on that determine the number of `Int#` in a `IntVec#` and the number of `Float#` in a `FloatVec#`, respectively.


### Native code generator



As we do not want to support SIMD instructions in the native code generator, we set the vector lengths (`intVecLen`, `floatVecLen`, and so on) to be 1 for all vector types when we compile with the native code generator.  Then, all vector types and operations on vector types can be trivially implemented with conventional instructions. (We use the same approach to handle scalar data types that are not supported by SIMD instructions on a particular architecture.)


## API



We use the following API extension for `GHC.Prim`.  Due to the odd manner in which sized integral types are implemented in `primops.txt.pp` (see heading "The word size story."), we need to introduce vector types for scalar types that do not exist as unboxed types in `GHC.Prim`. We obviously can't use "\[t\]he word size story" for SIMD vectors as they are packed, unboxed data types.


### Types



SIMD vector types: `IntVec#`, `Int8Vec#`, `Int16Vec#`, `Int32Vec#`, `Int64Vec#`, `WordVec#`, `Word8Vec#`, `Word16Vec#`, `Word32Vec#`, `Word64Vec#`, `FloatVec#`, and `DoubleVec#`.



Vector length constants: `intVecLen`, `intVec8Len`, `intVec16Len`, `intVec32Len`, `intVec64Len`, `wordVecLen`, `wordVec8Len`, `wordVec16Len`, `wordVec32Len`, `wordVec64Len`, `floatVecLen`, and `doubleVecLen`.  Each of these constants is of type `Int#`.



LLVM's [
vector operations:](http://llvm.org/docs/LangRef.html#vectorops) (`INT32`, `INT64`, `WORD32`, and `WORD64` are defined as `primops.txt.pp`)


```wiki
extractIntVec#    :: IntVec#    -> Int# -> Int#
extractInt8Vec#   :: Int8Vec#   -> Int# -> Int#
extractInt16Vec#  :: Int16Vec#  -> Int# -> Int#
extractInt32Vec#  :: Int32Vec#  -> Int# -> INT32
extractInt64Vec#  :: Int64Vec#  -> Int# -> INT64
extractWordVec#   :: WordVec#   -> Int# -> Int#
extractWord8Vec#  :: Word8Vec#  -> Int# -> Word#
extractWord16Vec# :: Word16Vec# -> Int# -> Word#
extractWord32Vec# :: Word32Vec# -> Int# -> WORD32
extractWord64Vec# :: Word64Vec# -> Int# -> WORD64
extractFloatVec#  :: FloatVec#  -> Int# -> Float#
extractDoubleVec# :: DoubleVec# -> Int# -> Double#

insertIntVec#    :: IntVec#    -> Int# -> Int#    -> IntVec#   
insertInt8Vec#   :: Int8Vec#   -> Int# -> Int#    -> Int8Vec#  
insertInt16Vec#  :: Int16Vec#  -> Int# -> Int#    -> Int16Vec# 
insertInt32Vec#  :: Int32Vec#  -> Int# -> INT32   -> Int32Vec# 
insertInt64Vec#  :: Int64Vec#  -> Int# -> INT64   -> Int64Vec# 
insertWordVec#   :: WordVec#   -> Int# -> Int#    -> WordVec#  
insertWord8Vec#  :: Word8Vec#  -> Int# -> Word#   -> Word8Vec# 
insertWord16Vec# :: Word16Vec# -> Int# -> Word#   -> Word16Vec#
insertWord32Vec# :: Word32Vec# -> Int# -> WORD32  -> Word32Vec#
insertWord64Vec# :: Word64Vec# -> Int# -> WORD64  -> Word64Vec#
insertFloatVec#  :: FloatVec#  -> Int# -> Float#  -> FloatVec#
insertDoubleVec# :: DoubleVec# -> Int# -> Double# -> DoubleVec#
```


Notes:


- The order of the second and third argument of the insert family differs from the LLVM operations to be more in line to what we usually use in Haskell.
- We should support LLVM's `shufflevector` operation as a shuffle family, much like the extract and insert families if this operation turns out to be useful in library or user code.


Arithmetic and logic operations:


```wiki
-- the following on all vector types
plusIntVec#, minusIntVec#, timesIntVec#, quotIntVec#, remIntVec# :: IntVec# -> IntVec# -> IntVec#
negateIntVec# :: IntVec# -> IntVec#

-- the following on IntVec#, Int8Vec#, Int16Vec#, Int32Vec#, and Int64Vec#
uncheckedIntVecShiftL#, uncheckedIntVecShiftRA#, uncheckedIntVecShiftRL# :: IntVec# -> Int# -> IntVec#

-- the following on WordVec#, Word8Vec#, Word16Vec#, Word32Vec#, and Word64Vec#
andWordVec#, orWordVec#, xorWordVec# :: Word# -> Word# -> Word#
notWord# :: WordVec# -> WordVec#
uncheckedWordVecShiftL#, uncheckedWordVecShiftRL# :: WordVec# -> Int# -> WordVec#

-- the following on FloatVec# and DoubleVec#
expFloatVec#, logFloatVec#, sqrtFloatVec#,
  sinFloatVec#, cosFloatVec#, tanFloatVec#,
  asinFloatVec#, acosFloatVec#, atanFloatVec#,
  sinhFloatVec#, coshFloatVec#, tanhFloatVec# :: DoubleVec# -> DoubleVec#
```


NB: The [
LLVM reference](http://llvm.org/docs/LangRef.html) states that LLVM doesn't currently support comparisons on vector types. Once that becomes available, we may want to support it as well.


## Using SIMD instructions in Data Parallel Haskell ([
DPH](http://www.haskell.org/haskellwiki/GHC/Data_Parallel_Haskell))



In DPH, we will use the new SIMD instructions by suitably modifying the definition of the lifted versions of arithmetic and other operations that we would like to accelerate. These lifted operations are defined in the `dph-common` package and made accessible to the vectoriser via [VECTORISE pragmas](data-parallel/vect-pragma). Many of them currently use `VECTORISE SCALAR` pragmas, such as


```wiki
(+) :: Int -> Int -> Int
(+) = (P.+)
{-# VECTORISE SCALAR (+) #-}
```


We could define them more verbosely using a plain `VECTORISE` pragma, but might instead like to extend `VECTORISE SCALAR` or introduce a variant.



**NB:** The use of SIMD instructions interferes with vectorisation avoidance for scalar subcomputations. Code that avoids vectorisation also avoids the use of SIMD instructions. We would like to use SIMD instructions, but still avoid full-scale vectorisation. This should be possible, but it is not immediately clear how to realise it (elegantly).


