## <img src="https://gitlab.com/nowsecure/research/esilsolve/-/raw/master/raphi.svg" alt="logo" width="200"/> ESILSolve - A python symbolic execution framework using r2 and ESIL

ESILSolve uses the z3 theorem prover and r2's ESIL intermediate representation to symbolically execute code. 

ESILSolve supports the same architectures as ESIL, including x86, amd64, arm, aarch64 and more. This project is a work in progress.

### Example Usage

```python
from esilsolve import ESILSolver

# start the ESILSolver instance
# and init state with r2 symbol for check function
esilsolver = ESILSolver("tests/multibranch", debug=False)
state = esilsolver.call_state("sym.check")

# make rdi (arg1) symbolic
state.set_symbolic_register("rdi")
rdi = state.registers["rdi"]

# hook callback
def success(instr, state):
    print("ARG1: %d" % state.evaluate(rdi).as_long())

# hook any address to manipulate states
# and set targets and avoided addresses
esilsolver.register_hook(0x6a1, success)
esilsolver.run(target=0x6a1, avoid=[0x6a8])
```

ESILSolve also easily works with ipa and apk files since they are supported by r2. 

### IPA CrackMe Example

```python
from esilsolve import ESILSolver

buf_addr = 0x100000
buf_len = 16

esilsolver = ESILSolver("ipa://tests/crackme-level0-symbols.ipa", debug=False)
state = esilsolver.call_state("sym._validate")
state.registers["x0"] = buf_addr

#use r2pipe like normal in context of the app
validate = esilsolver.r2pipe.cmdj("pdj 1")[0]["offset"]
smt = esilsolver.smt # just z3 with some extras

# initialize symbolic bytes of solution
# and constrain them to be /[a-z ]/
b = [smt.BitVec("b%d" % x, 8) for x in range(buf_len)]
for x in b:
    state.solver.add(smt.Or(smt.And(x >= 0x61, x <= 0x7a), x == 0x20))

# concat the bytes and write the BV to memory 
code = smt.Concat(*b)
state.memory[buf_addr] = code

# success hook callback
def success(instr, state):
    cs = smt.BV2Bytes(state.evaluate(code))
    # gives an answer with lots of spaces but it works
    print("CODE: '%s'" % cs.decode())

# set the hooks and run
esilsolver.register_hook(validate+0x210, success)
esilsolver.run(target=validate+0x210, avoid=[validate+0x218, validate+0x3c])
```