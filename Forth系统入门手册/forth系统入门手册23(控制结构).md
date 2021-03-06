##第23章 控制结构

所有的编程语言都有顺序、分支、循环这三种控制结构，forth也不例外。

因为计算机只能按顺序执行指令（它就是一根筋），跳转指令则是通过修改IP寄存器内的指令地址数值来实现，所以对计算机来说，保留循环体的首端地址，并在尾端跳转到首端地址的循环结构更容易实现；而分支结构则是从首端跳转到尾端，在首端不知道尾端地址的情况下，比较难办。

例如do...loop循环结构

```ASM
;do word1 word2 word3 loop
;do             ( - )
	$code	'do',do@
		$rsfrom si	 	;把接下来的循环体的首端地址压进返回栈
	$next


;loop  ( - )
	$code	'loop',loop@
		mov si,[bp]	;跳转到首端，并保留首端地址，以待下次循环
	$next
```

循环就是这样了，但是要跳出循环体该怎么办，又不知道尾端地址？于是问题又回到了分支结构上。


传统forth的分支结构是if...else...then，使用方式是：

    条件 if 符合条件的语句 else 不符合条件的语句 then

then在这里作为if语句的结束标记，但是后来也有替换为endif的。但在内部的汇编源代码里用的并不是if和else，取而代之的是?branch和branch

```ASM
;   ?branch	( f -- )	如果栈顶为0，则跳转到指定的标签位置；不为0则跳到下下个词执行
;		Branch if flag is zero.

                $CODE   '?branch',QBRAN
		POP	BX			;pop flag
		OR	BX,BX			;?flag=0
		JZ	BRAN1			;yes, so branch
		add si,celll
		$NEXT
BRAN1:		MOV	SI,0[SI]		;IP:=(IP)
		$NEXT

;   branch	( -- )	 直接跳转到指定的标签位置
;		Branch to an inline address.

                $CODE   'branch',BRAN
		MOV	SI,0[SI]		;IP:=(IP)
		$NEXT


      使用方式：
    ?branch label1
    .......     （符合条件的语句）
    branch label2
label1:
    .......（不符合条件的语句）
label2:
```

在传统forth里，if和else是只能在编译模式下使用的扩展词，在解释模式是不能使用的。if的作用是编译出?branch label，，取代if所在的位置，else则是编译出branch label，它先找出else和then的位置，然后根据它们的位置，计算出label的值，然后直接跳转。这种实现方式是不定长的跳转。


但是为了简化编译器，我打算做个只有编译没有解释的forth编译器，解释模式则用执行单条编译完毕的语句的方式代替。我也不想在汇编源码里用branch这种词来编程，我想在汇编里直接使用if...else...endif这几个词。

第一个设想是不定长查找，让if逐个判断身后的forth词是不是else@和endif@，但是在汇编里是以dw在内存里写数据，else@和endif@也是地址数值，如果在if当中有数字输入，而且还正好就是else@和endif@的数值，或者在if和else@、endif@之间夹着长度奇偶不等的字符串，那就成了隐患了。

第二个设想是定长跳转，让if是有条件跳过两个词，else是直接跳过一个词，endif是个空指令（仅仅是用来补足if里两个词的格式）

    _______________
    |　　　　　　　　↓
    if word1 word2 .....
    最多可以写下两个词

    ______________
    |　　　　　　　↓
    if word endif .....
    如果只有一个词,就用endif这个空指令填补


```ASM
; if	( n --  )
	$code	'if',if@
		or tos,tos
		pop tos		;mov push pop 不会影响到标志位
		jnz if@1		;如果不是0，就顺序执行
		add si,celll*2	;否则跳过两个词
	if@1:
	$next


; if0	( n --  )	与if相反
	$code	'if0',if0@
		or tos,tos
		pop tos		;mov push pop 不会影响到标志位
		jz if@1		;如果是0，就顺序执行
		add si,celll*2
	$next

; endif	( n --  )	空指令，只是用来填if的缺
	$code	'endif',endif@
	$next
```

     _____________
    |　　　　　　　↓
    if word_y else word_n .......
　　|　　  |__________↑

```ASM
;else	 ( - )	 直接跳过一个词，后面也无需跟一个endif
	$code	'else',else@
		add si,celll
	$next
```

虽然这种定长跳转的分支结构限制很大，但是却能逼迫我尽量写出短语句和因子化。实际上，计算机的世界不会没有限制，否则那些算法什么的也没有存在的必要了，大家一律使用穷举法，无脑编程即可。有限制，才能有进步，这也是一个真理。相信你也不想看到一长串挂面一样的if语句吧？

传统forth的循环结构很多，那些个begin until repeat  while limit loop什么的，我都记不住。好在我也没打算记住，只选了for...next和do...loop这两个来做。
    
在开始的时候，myforth的for...next和do...loop也被设计成中间只能容下一个词，实现起来容易，但编程的时候麻烦，总是要多定义一个词来包含循环内容，而且也给词典增加了无法重用的词。

```ASM
;for case_f next
; for           ( n -- )
	$code	 'for',for@
		or tos,tos
		jz if@		;如果栈顶为0，就把for当做if来用，越过next
		jmp Lr@		;否则，">r"


; next  ( - )
	$code	'next',next@
		dec word [bp]	;计数
		jz dropr@		;如果为0则退出循环
		sub si,2*celll	;否则退两个词，循环
	$next

;: case_f	~~~~break~~~；
; break ( rs->si  dropr )
	$code	'break',break@
		mov si,[bp]	;得到上层next的地址
		add si,celll	;越过next
		add bp,2*celll	;丢掉返回地址和计数值
	$next


;do case_o loop
;do             ( - )
	$code	'do',do@
		sub bp,celll	;匹配for...next的格式，为了统一使用break
	$next


;loop  ( - )
	$code	'loop',loop@
		sub si,2*celll
	$next
```

后来我在《1x forth》里得到了启发

“我得到了一个新的循环结构，它在 COLOR Forth 中使用，我觉得它比另外的那些都好。这种方法是：如果我有一个字 WORD ，我可以实现对这个字 WORD 的某种有条件的引用。这就是我的循环方式。 

    WORD ~~~ IF ~~~ WORD ; 
    THEN ~~~ ; 

我回到当前定义的开始 , 这就是我现在使用循环的唯一方法，它是足够而方便的。它还有两个边际影响：一个就是它要求一种递归版本的 Forth, 这个字必须在当前的定义中被引用，而不能够要求被预先定义。这就省去了 SMUDGE/UNSMUDGE 概念， ANS 正准备为这个概念找一个合适的名字。但是最终的结果是它更简单了。 

对于嵌套的循环这种方法当然很不方便，但是嵌套的循环毕竟是一个不确定的概念。你也可以有嵌套定义。你应该条件化地执行一个字，还是应该有某种像 IF THEN 这样的结构？这个问题我们已经讨论 15 年了。这里有一个例子，我想它说得很明白，唯一的循环是必须重复一个字。 ”


于是我换了一种循环的写法， 没有采用上面所说的递归，仍然是do...loop，但不是写成

    : word_o	~~~~~~ ;
    : word	~~~ do word_o loop ~~~ ;

而是写成了一种尾循环

    : word1	~~~ do ~~~~~~ loop ;
    : word2	word1 ~~~ ;

把do~~loop写进一个词里，并把loop固定为定义字的结尾。

这样一来，循环首端地址还是由do来搞定，执行到loop的时候从返回栈读取出来，而break的作用就是跳出循环，既然loop已经被固定在句尾了，只需要丢掉返回栈的首端地址，再exit（也就是“;”），就能跳出循环。

如果用冒号词来写

    (   -RS-  addr )
    : do	r@ >r ;
    执行每个冒号词的时候，都会先保存上层地址，也就是说
    没进入do之前
    RS
    ┏━━━━━━━━━━
    ┃
    ┗━━━━━━━━━━

     进入do的时候
    RS
    ┏━━━━━━━━━━
    ┃上层地址(循环首端)
    ┗━━━━━━━━━━


    执行r@的时候
    RS
    ┏━━━━━━━━━━
    ┃上层地址
    ┗━━━━━━━━━━
    DS
    ┏━━━━━━━━━━
    ┃上层地址
    ┗━━━━━━━━━━


        执行r>的时候
    RS
    ┏━━━━━━━━━━
    ┃上层地址  上层地址
    ┗━━━━━━━━━━
    DS
    ┏━━━━━━━━━━
    ┃
    ┗━━━━━━━━━━

别忘了还有“;”，在汇编源码里，它其实就是exit，比较特殊的是在编译之前靠它来判断冒号语句的结束。

    $colon 'do',do@
    dw	r@@, Lr@, exit@

虽然在一般的冒号词里，你不需要注意到它，但是在这种需要操作返回栈的冒号词里面，你绝对不能忽视它。
最后执行完毕，回到上层的时候

    RS
    ┏━━━━━━━━━━
    ┃上层地址(循环首端)
    ┗━━━━━━━━━━
    DS
    ┏━━━━━━━━━━
    ┃
    ┗━━━━━━━━━━



再说loop

    (   addr   -RS-  addr )
    : loop	dropr r@ >r ;

没进入loop之前

    RS
    ┏━━━━━━━━━━
    ┃循环首端
    ┗━━━━━━━━━━

     进入loop之后
    RS
    ┏━━━━━━━━━━
    ┃循环首端  上层地址
    ┗━━━━━━━━━━


    dropr
    RS
    ┏━━━━━━━━━━
    ┃循环首端
    ┗━━━━━━━━━━


    r@
    RS
    ┏━━━━━━━━━━
    ┃循环首端
    ┗━━━━━━━━━━
    DS
    ┏━━━━━━━━━━
    ┃循环首端
    ┗━━━━━━━━━━


     r>
    RS
    ┏━━━━━━━━━━
    ┃循环首端  循环首端
    ┗━━━━━━━━━━
    DS
    ┏━━━━━━━━━━
    ┃
    ┗━━━━━━━━━━

    ;  消耗一个返回栈栈顶单元，回到循环首端
    RS
    ┏━━━━━━━━━━
    ┃循环首端
    ┗━━━━━━━━━━


    (   addr   -RS-  addr )
    : loop	dropr r@ >r ;

    (   -RS-  addr )
    : do	r@ >r ;

你应该能看出do和loop有相同的部分，那么能不能把loop写成

    : loop	dropr do ;

如果只是操作数据栈的词，这样写没问题，但如果是操作返回栈的“冒号词”，问题就大了，原因就是冒号词一开始就会保存上层地址到返回栈，这就意味着在你要操作的返回栈上无端端的多了一个参数，想要正确执行，就必须解决这个多出来的上层地址。所以对待操作返回栈的“冒号词”，不能简单的进行替换。

顺便一提，在需要用到r>和>r的词里，这两个一般是成对使用（除非你确实有单独使用的需要），当你写下其中一个的时候，就应该马上写下另一个，就像括号配对一样。否则会扰乱返回栈，造成整个系统的崩溃。这就好比你写C程序时，忘掉了“}”一样，只不过C提供了售后服务，检查出来了而已。而forth会默认你的选择，只看你有没有把握这种自由的能力了。


    ( addrUp addrLoop  -RS-    )
    : break	dropr dropr ;

在这里的break同样是个冒号词，它丢掉的两个返回栈单元并不是addrUp和addrLoop，而是addrBreakExit和addrLoop。addrUp则是被“;”给消耗掉了。

break的用法：

    ( -- T | 0 )
    : 条件	~~~ ;
    : word	~~~ do 条件 if break endif ~~~ loop ;
    : word	~~~ do ~~~ 条件 if break endif loop ;

其实后置的break还有另外几种好玩的替换写法

    : word	~~~ do ~~~ 条件 if loop endif dropr ~~~~~~ ;
    : word	~~~ do ~~~ 条件 if word1 loop  ~~~~~~ break  ;

   

还可以写成

    ( n -- ) ( addrLoop addrUp  -RS-  addrLoop |      )
    : loop?	if dropr loop r> dropr >r ;
    : word	~~~ do ~~~ 条件 loop?  ~~~~~~ ;
    ( if 跳过两个词，如果你嫌看起来不够清晰，可以把空指令endif也加进去 ，只要你别忘了if只能跳过一个不多，一个不少的两个词 )

这样一来，你也可以不用被尾循环限制了。但最好还是把循环单独定义成一个词，因为对forth而言，因子化是非常重要的。

    
 
同样的，如果想把if break endif简化一下

    ( n -- ) ( addrLoop addrUp  -RS-      |  addrLoop )
    : break?	dropr if break endif ;
    : word	~~~ do 条件 break? ~~~ loop ;
    : word	~~~ do ~~~ 条件 break? loop ;


由此得出if...endif的另一种用法

从

    : word	~~~ if  ~~ endif ;
    
变成

    : word	~~~ if exit endif ~~~~~~ ;

这让最多只能写下两个词的if有了一个写长句的机会，至于else。。。忘了它吧。


最后是for...next，依然是尾循环

    : word	~~~ num for ~~~ next ;

    ( n -- n n | 0 )
    : dup?	dup if dup endif ;
    ( n --  ) (  -RS- addr n )
    : for	dup? if0 dropr exit  r@ swap >r >r ;

有些版本的forth对 num for...next是循环了num+1次，在我这个版本则是说一不二。当num==0的时候， 不进入循环。

```ASM
; next  ( - )	( addr num -RS-        | addr num'  )
	$code	'next',next@
		dec word [bp]	;计数
		jz next@0	;如果为0则退出循环
		mov si,[bp+celll]	;否则循环
		$next
	next@0:
		add bp,celll*2
	$next
```

中途终止for循环

    : word	~~~ num for ~~ 条件 if dropr break ~~ next ;

