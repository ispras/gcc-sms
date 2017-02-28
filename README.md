# GCC with extended modulo scheduling support

All patches descriptions available in the [gcc-patches mailing list archives](https://gcc.gnu.org/ml/gcc-patches/2017-02/msg01647.html). There are also branches with almost the same changes for gcc branches starting from 4.9 (sms-4_9, sms-5, sms-6 branches).

### Loops supported
This significantly enhances the existing implementation of GCC modulo scheduler. It adds support for scheduling loops without doloop pattern, which allows using modulo scheduler on x86 and x86_64 platforms. The loops should meet the following requirements, first three are the same as for loops with doloop pattern:
- loop body contains only one basic block;
- basic block contains no calls or barriers inside;
- there are no !single_set instructions (with the exception of control part
  instructions);

The next three describe the control part of new supported loops:

- the last jump instruction should look like: `pc=(regF!=0)?label:pc`, `regF` is
  flag register;
- the last instruction which sets regF should be:`regF=COMPARE(regC,X)`, where `X`
  is a constant, or a register, which is not changed inside a loop;
- only one instruction modifies regC inside a loop (other can use regC, but not
  write), and it should simply adjust it by a constant: `regC=regC+step`, where
  `step` is a constant.

It provides backward compatibility, which means that doloop loops are processed as usual.
When doloop is succesfully scheduled by SMS, its number of iterations of loop kernel should be decreased by the number of stages in a schedule minus one, while other iterations are pushed out to prologue and epilogue.
In new supported loops such approach can't be used, because some instructions can use count register `regC`.
Instead of this, the final register value `X` in compare instruction `regF=COMPARE(regC,X)` is changed to another value `Y` respective to the stage this instruction is scheduled: `Y = X - stage * step`.
If `X` is an immediate value and `Y` is also possible to be immediate, we just change an instruction.
In other cases, new register `regY` is used to store new final value.
The instructions of `regY` initialization are added to the prologue and compare instruction is changed to `regF=COMPARE(regC,regY)`.


