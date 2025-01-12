;;........................
;; DI -> SI optimizations
;;........................

;; Simplify (int)(a + 1), etc.
(define_peephole2
  [(set (match_operand:DI 0 "register_operand")
	(match_operator:DI 4 "modular_operator"
	  [(match_operand:DI 1 "register_operand")
	   (match_operand:DI 2 "arith_operand")]))
   (set (match_operand:SI 3 "register_operand")
	(truncate:SI (match_dup 0)))]
  "TARGET_64BIT && (REGNO (operands[0]) == REGNO (operands[3]) || peep2_reg_dead_p (2, operands[0]))
   && (GET_CODE (operands[4]) != ASHIFT || (CONST_INT_P (operands[2]) && INTVAL (operands[2]) < 32))"
  [(set (match_dup 3)
	  (truncate:SI
	     (match_op_dup:DI 4 
	       [(match_operand:DI 1 "register_operand")
		(match_operand:DI 2 "arith_operand")])))])

;; Simplify (int)a + 1, etc.
(define_peephole2
  [(set (match_operand:SI 0 "register_operand")
	(truncate:SI (match_operand:DI 1 "register_operand")))
   (set (match_operand:SI 3 "register_operand")
	(match_operator:SI 4 "modular_operator"
	  [(match_dup 0)
	   (match_operand:SI 2 "arith_operand")]))]
  "TARGET_64BIT && (REGNO (operands[0]) == REGNO (operands[3]) || peep2_reg_dead_p (2, operands[0]))"
  [(set (match_dup 3)
	(match_op_dup:SI 4 [(match_dup 1) (match_dup 2)]))])

;; Simplify -(int)a, etc.
(define_peephole2
  [(set (match_operand:SI 0 "register_operand")
	(truncate:SI (match_operand:DI 2 "register_operand")))
   (set (match_operand:SI 3 "register_operand")
	(match_operator:SI 4 "modular_operator"
	  [(match_operand:SI 1 "reg_or_0_operand")
	   (match_dup 0)]))]
  "TARGET_64BIT && (REGNO (operands[0]) == REGNO (operands[3]) || peep2_reg_dead_p (2, operands[0]))"
  [(set (match_dup 3)
	(match_op_dup:SI 4 [(match_dup 1) (match_dup 2)]))])

;; Simplify (unsigned long)(unsigned int)a << const
(define_peephole2
  [(set (match_operand:DI 0 "register_operand")
	(ashift:DI (match_operand:DI 1 "register_operand")
		   (match_operand 2 "const_int_operand")))
   (set (match_operand:DI 3 "register_operand")
	(lshiftrt:DI (match_dup 0) (match_dup 2)))
   (set (match_operand:DI 4 "register_operand")
	(ashift:DI (match_dup 3) (match_operand 5 "const_int_operand")))]
  "TARGET_64BIT
   && INTVAL (operands[5]) < INTVAL (operands[2])
   && (REGNO (operands[3]) == REGNO (operands[4])
       || peep2_reg_dead_p (3, operands[3]))"
  [(set (match_dup 0)
	(ashift:DI (match_dup 1) (match_dup 2)))
   (set (match_dup 4)
	(lshiftrt:DI (match_dup 0) (match_operand 5)))]
{
  operands[5] = GEN_INT (INTVAL (operands[2]) - INTVAL (operands[5]));
})

;; Simplify PIC loads to static variables.
;; These will go away once we figure out how to emit auipc discretely.
(define_insn "*local_pic_load<mode>"
  [(set (match_operand:ANYI 0 "register_operand" "=r")
	(mem:ANYI (match_operand 1 "absolute_symbolic_operand" "")))]
  "USE_LOAD_ADDRESS_MACRO (operands[1])"
  "<load>\t%0,%1"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_load<mode>"
  [(set (match_operand:ANYF 0 "register_operand" "=f")
	(mem:ANYF (match_operand 1 "absolute_symbolic_operand" "")))
   (clobber (reg:DI T0_REGNUM))]
  "TARGET_HARD_FLOAT && TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[1])"
  "<load>\t%0,%1,t0"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_load<mode>"
  [(set (match_operand:ANYF 0 "register_operand" "=f")
	(mem:ANYF (match_operand 1 "absolute_symbolic_operand" "")))
   (clobber (reg:SI T0_REGNUM))]
  "TARGET_HARD_FLOAT && !TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[1])"
  "<load>\t%0,%1,t0"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_loadu<mode>"
  [(set (match_operand:SUPERQI 0 "register_operand" "=r")
	(zero_extend:SUPERQI (mem:SUBDI (match_operand 1 "absolute_symbolic_operand" ""))))]
  "USE_LOAD_ADDRESS_MACRO (operands[1])"
  "<load>u\t%0,%1"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_storedi<mode>"
  [(set (mem:ANYI (match_operand 0 "absolute_symbolic_operand" ""))
	(match_operand:ANYI 1 "reg_or_0_operand" "rJ"))
   (clobber (reg:DI T0_REGNUM))]
  "TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[0])"
  "<store>\t%z1,%0,t0"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_storesi<mode>"
  [(set (mem:ANYI (match_operand 0 "absolute_symbolic_operand" ""))
	(match_operand:ANYI 1 "reg_or_0_operand" "rJ"))
   (clobber (reg:SI T0_REGNUM))]
  "!TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[0])"
  "<store>\t%z1,%0,t0"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_storedi<mode>"
  [(set (mem:ANYF (match_operand 0 "absolute_symbolic_operand" ""))
	(match_operand:ANYF 1 "register_operand" "f"))
   (clobber (reg:DI T0_REGNUM))]
  "TARGET_HARD_FLOAT && TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[0])"
  "<store>\t%1,%0,t0"
  [(set (attr "length") (const_int 8))])
(define_insn "*local_pic_storesi<mode>"
  [(set (mem:ANYF (match_operand 0 "absolute_symbolic_operand" ""))
	(match_operand:ANYF 1 "register_operand" "f"))
   (clobber (reg:SI T0_REGNUM))]
  "TARGET_HARD_FLOAT && !TARGET_64BIT && USE_LOAD_ADDRESS_MACRO (operands[0])"
  "<store>\t%1,%0,t0"
  [(set (attr "length") (const_int 8))])
