### Resarching about Rust architecture.
We're going to start develop and research compilers (Rust, GO, C). Compilers mostly have a few important phases such as Lexer analysis, Syntax analysis, Semantic analysis, IR (Intermediate code/representation generation), Code optimization and generation. We firstly take a look at phases of rustc.  


Register allocation can be used to assign a large number of variables to registers. These variables can be transferred to unlimited registers of IR, but, this process will not be possible with CPU registers. Because, IR format has an unbounded number of temporary registers, otherwise, CPU has a bounded number of physical registers (rax, rdi, rbx, rcx etc.). So, register allocation is an NP-complete problem. Can we allocate all these n temporaries to k registers? There are a few techniques based on graph coloring. Rust doesn't involve a register allocation stage, this process will be done by LLVM.


Data-flow analysis
Compiler design-da istifadə edilən Liveness analysis, dead code elimination, reachability kimi mexanizmlər data-flow analiz metodikasi üzərindən tətbiq edilir. Vulnerability researching ilə məşğul olduğum zamnda akademik metodlar ilə data-flow analizini tətbiq edirdim. Data-flow analiz zamanı biz proqram təminatının control-flow graph forması üzərində analizimizi aparırıq. CFG 2 əsas hissədən ibarətdir flow path (edge) və node. CFG üzərində predecessor və successor node-lar var bu node-lar isə bir node-a bağlı child və parent node-ları bildirir. 

The successor nodes of a node are called its children.

Predecessor node of a node is called its parent.
```
             +-----------------------+
             |                       |
             |     Basic block 1     | Predecessor of B2, B3
             |                       |
             +----------X------------+
                        X
                       Edge
                        X
        XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
        X                               X
        X                               X
+----------------+             +----------------+
|                |             |                |
|  Basic block 2 |             |  Basic block 3 | Successor of B1
|                |             |                |
+----------------+             +----------------+
```

Liveness analysis

Variable x p point-də live olur o halda ki x-in dəyəri digər path-larda redefine olunmadan istifadə edilir. (A variable is live at some point if it holds a value that may be needed in the future, or equivalently if its value may be read before the next time the variable is written to). Bu mexanizm ilə biz dead variable-ları təyin edə bilərik. Məsələn:
```
main:
x = 0xdeadbeef
-- x is dead, it should be read before redefine-- 
func1:
x = 0x41414141
```
Liveness analiz zamanı biz dataflow equation ilə backward analiz edirik yəni variable-ın predeccessor node-larda olan state-lərini hesablayırıq. Bu equation-da istifadə etdiyimiz 2 əsas set var GEN və KILL (DEF or USE).
USE set müyyən basic block/node üzərindəki istifadə edilən variable-ları saxlayır.

<img src="https://render.githubusercontent.com/render/math?math=USE[s]">

DEF isə müyyən basic block/node üzərindəki define edilmiş variable-ları saxlayır. 

<img src="https://render.githubusercontent.com/render/math?math=DEF[s]">

Hər hansı bir CFG üzərində iterativ olaraq liveness analizini riyazi aparmaq istədikdə isə aşağıdaki equationı istifadə edə bilərik.

*Equationdaki bütün əməliyyatlar sadə set theory operatorları ilə aparılır*

<img src="https://raw.githubusercontent.com/goupaz/lowlevel/master/resources/equ.jpg">

Burdaki out[n] seti bizim successor nodunun in setinin dəyərlərini saxlayır (backward). in[n] seti isə node-un define, use və out setindəki məlumatlara uyğun olaraq hesablama aparır.
operation for node 3:

```
use[3] = {i, k}
out[3] = {i,n}
def[3] = {i}
in[3] = (use[3]) U (out[3] - def[3])
in[3] = {i, k, n}
```

Liveness analysis üçün pseudo kodu yerləşdirirəm:

<img width="300" height="200" src="https://raw.githubusercontent.com/goupaz/lowlevel/master/resources/liveness.png">



#### Rust operations
Rust has few stages until the final stage such as HIR, LLVM IR, Machine code generation. Modern Rust has also MIR between the existing HIR and LLVM IR. MIR generates LLVM IR after parsing, type checking, borrow checking and optimization stage. e.g:
rust source code:
```
fn foo() -> i32{
    let a = 0x55;
    let a = a + 0x4;
    return a;
}
```
Rust uses LLVM based intrinsic function for fast arithmetic overflow checking. Therefore, MIR will generate to call llvm.sadd.with.overflow.* function.
```
[DEBUG rustc_codegen_llvm::builder] call (i32 ()*:
      %1 = call { i32, i1 } @llvm.sadd.with.overflow.i32(i32 85, i32 4)
```

<img width="400" height="400" src="https://blog.rust-lang.org/images/2016-04-MIR/flow.svg">



### Rust bound checking
Rust bound check əməliyyatı MIR tərəfindən aparılır yəni syntax parse edildiyi zaman əgər biz index operatoru istifadə etmişiksə MIR əlavə olaraq bound_check funksiyasını əlavə edir. MIR source kodu textual formaya yox Control-flow-graph formasında generate edir. Burada birçox safety-checking prosesləri generated CFG üzərində baş verir. 


safe code

```
#[allow(unused_variables)]
#[allow(const_err)]
fn main() {
    let a = [1, 2, 3];    
    let index = 344;
    let element = a[index];
    println!("The value of element is: {}", element);
}

```
Məsələn biz kod üzərində hər hansı bir index əməliyyatı istifadə etdikdə MIR onun üçün CFG-a bound check əməliyyatı əlavə edir.

./src/librustc_mir_build/build/expr/as_place.rs
```
153             ExprKind::Index { lhs, index } => this.lower_index_expression(
154                 block,
155                 lhs,
156                 index,
157                 mutability,
158                 fake_borrow_temps,
159                 expr.temp_lifetime,
160                 expr_span,
161                 source_info,
162             ),
```

lower_index_expression funksiyası daha sonra bounds_check funksiyasını çağıraraq uyğun operatorları CFG-a əlavə edir.

./src/librustc_mir_build/build/expr/as_place.rs: lower_index_expression function
```
306         block = self.bounds_check(
307             block,
308             base_place.clone().into_place(self.hir.tcx()),
309             idx,
310             expr_span,
311             source_info,
312         );
```
./src/librustc_mir_build/build/expr/as_place.rs: bounds_check function
```
344         self.cfg.push_assign(block, source_info, &len, Rvalue::Len(slice));
345         // lt = idx < len
346         self.cfg.push_assign(
347             block,
348             source_info,
349             &lt,
350             Rvalue::BinaryOp(BinOp::Lt, Operand::Copy(Place::from(index)), Operand::Copy(len)),
351         );
352         let msg = BoundsCheck { len: Operand::Move(len), index: Operand::Copy(Place::from(index)) };
```
bounds_check funksiyasında isə Lt operatoru əlavə edilir (Less than) burada isə tələb olunan offset-in ümumi uzunluğdan kiçik olub olmaması yoxlanılır.

CFG:

<img width="450" height="150" src="https://raw.githubusercontent.com/goupaz/lowlevel/master/resources/boundcheck.png">

MIR output:
```


    bb0: {
        StorageLive(_1);                 // bb0[0]: scope 0 at src/main.rs:4:9: 4:10
        _1 = [const 1i32, const 2i32, const 3i32, const 4i32, const 5i32]; // bb0[1]: scope 0 at src/main.rs:4:13: 4:28
                                         // ty::Const
                                         // + literal: Const { ty: i32, val: Value(Scalar(0x00000005)) }
        StorageLive(_2);                 // bb0[2]: scope 1 at src/main.rs:5:9: 5:14
        _2 = const 10usize;              // bb0[3]: scope 1 at src/main.rs:5:17: 5:19
                                         // ty::Const
                                         // + ty: usize
                                         // + val: Value(Scalar(0x000000000000000a))
                                         // mir::Constant
                                         // + span: src/main.rs:5:17: 5:19
                                         // + literal: Const { ty: usize, val: Value(Scalar(0x000000000000000a)) }
        StorageLive(_3);                 // bb0[4]: scope 2 at src/main.rs:7:9: 7:16
        StorageLive(_4);                 // bb0[5]: scope 2 at src/main.rs:7:21: 7:26
        _4 = _2;                         // bb0[6]: scope 2 at src/main.rs:7:21: 7:26
        _5 = const 5usize;               // bb0[7]: scope 2 at src/main.rs:7:19: 7:27
                                         // ty::Const
                                         // + ty: usize
                                         // + val: Value(Scalar(0x0000000000000005))
                                         // mir::Constant
                                         // + span: src/main.rs:7:19: 7:27
                                         // + literal: Const { ty: usize, val: Value(Scalar(0x0000000000000005)) }
        _6 = Lt(_4, _5);                 // bb0[8]: scope 2 at src/main.rs:7:19: 7:27
        assert(move _6, "index out of bounds: the len is move _5 but the index is _4") -> bb1; // bb0[9]: scope 2 at src/main.rs:7:19: 7:27
    }
    
```

Daha sonra bunun LLVM IR və ASM outputlarına baxaq.

```
define internal void @_ZN10playground4main17he2ed904ee11c0235E() unnamed_addr #0 !dbg !285 {
start:
  %arg0 = alloca i32*, align 8
  %_16 = alloca i32*, align 8
  %_15 = alloca [1 x { i8*, i8* }], align 8
  %_8 = alloca %"core::fmt::Arguments", align 8
  %element = alloca i32, align 4
  %index = alloca i64, align 8
  %a = alloca [5 x i32], align 4
  call void @llvm.dbg.declare(metadata [5 x i32]* %a, metadata !287, metadata !DIExpression()), !dbg !292
  call void @llvm.dbg.declare(metadata i64* %index, metadata !293, metadata !DIExpression()), !dbg !295
  call void @llvm.dbg.declare(metadata i32* %element, metadata !296, metadata !DIExpression()), !dbg !298
  call void @llvm.dbg.declare(metadata i32** %arg0, metadata !299, metadata !DIExpression()), !dbg !303
  %0 = bitcast [5 x i32]* %a to i32*, !dbg !304
  store i32 1, i32* %0, align 4, !dbg !304
  %1 = getelementptr inbounds [5 x i32], [5 x i32]* %a, i32 0, i32 1, !dbg !304
  store i32 2, i32* %1, align 4, !dbg !304
  %2 = getelementptr inbounds [5 x i32], [5 x i32]* %a, i32 0, i32 2, !dbg !304
  store i32 3, i32* %2, align 4, !dbg !304
  %3 = getelementptr inbounds [5 x i32], [5 x i32]* %a, i32 0, i32 3, !dbg !304
  store i32 4, i32* %3, align 4, !dbg !304
  %4 = getelementptr inbounds [5 x i32], [5 x i32]* %a, i32 0, i32 4, !dbg !304
  store i32 5, i32* %4, align 4, !dbg !304
  store i64 10, i64* %index, align 8, !dbg !305
  %_4 = load i64, i64* %index, align 8, !dbg !306
  %_6 = icmp ult i64 %_4, 5, !dbg !307
  %5 = call i1 @llvm.expect.i1(i1 %_6, i1 true), !dbg !307
  br i1 %5, label %bb1, label %panic, !dbg !307
```




```
_ZN10playground5main17he2ed904ee11c0235E: # @_ZN10playground5main17he2ed904ee11c0235E
        mov     eax, dword ptr [rsp - 4]
        mov     dword ptr [rsp - 28], eax
        mov     qword ptr [rsp - 24], 10
        mov     al, 1
        test    al, al
        jne     .LBB0_2
        mov     eax, 1
        ret
```

Burda LLVM optimizasiya üçün offset dəyəri pre-defined olduğun üçün optimal instruction generasiya edib. LLVM həmçinin backend optimizasiyasında bound check elimination-da tətbiq edir.

References:


1. [https://www.youtube.com/watch?v=VgwFvLc9xLM](https://www.youtube.com/watch?v=VgwFvLc9xLM)
2. [https://www.inf.ed.ac.uk/teaching/courses/copt/lecture-3.pdf](https://www.inf.ed.ac.uk/teaching/courses/copt/lecture-3.pdf)
3. [https://www.cs.colostate.edu/~mstrout/CS553Fall06/slides/lecture09-dataflow.pdf](https://www.cs.colostate.edu/~mstrout/CS553Fall06/slides/lecture09-dataflow.pdf)
4. [https://suif.stanford.edu/~courses/cs243/lectures/l2.pdf](https://suif.stanford.edu/~courses/cs243/lectures/l2.pdf)
5. [https://cseweb.ucsd.edu/classes/fa03/cse231/lec3seq.pdf](https://cseweb.ucsd.edu/classes/fa03/cse231/lec3seq.pdf)
6. [https://proglang.informatik.uni-freiburg.de/teaching/compilerbau/2016ws/10-liveness.pdf](https://proglang.informatik.uni-freiburg.de/teaching/compilerbau/2016ws/10-liveness.pdf)
