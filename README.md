# TERMS language specification


## Using terms compiler
### Compile and run terms compiler
Terms compiler source code is written in C99 and can be compiled on Linux systems using GCC.
```
gcc terms.c -Wno-incompatible-pointer-types-discards-qualifiers -o terms
```

Compiler executable file takes filename as argument. Terms language files are expected to have .tt extension.
For example:
```
./terms examples/token.tt
```
This would compile token.tt and produce a range of output files:
token.js   - to deploy a contract using geth, just copy and paste contains of this file into geth console
token.asm  - raw assembly version of a contract. Can be useful for low-level code debugging
token.abi  - contract abi
token.hex  - contract bytecode

A project in terms language can only contain one .tt file. Optionally, it can contain .ttp file that is concatenated to the end of .tt file during compiler preprocessor stage. Normally, .ttp file is used for terms language "procedures".


## Philisophy behind TERMS language
The idea of TERMS language is to provide possibility to write EVM code in a human readable language, subset of [legal English] (https://en.wikipedia.org/wiki/Legal_English). It does not try to follow a particular programming concept paradigm and compiles directly into EVM instructions.


## Contract structure
TERMS contract source code consists of a single .tt file and, optionally, a .ttp file that is concatenated to the end of .tt file during compiler preprocessor stage. Normally, .ttp file is used for terms language "procedures".

Contract constructor section begins with word "Contract", optionally followed by the contract's name in double quotes. The name doesn't affect the bytecode in any way (unlike i.e. in Solidity).

Contract main section consists of word Conditions followed by separator. Separators in TERMS are to divide sentences of the contract.
Separators are the following: dot(.), colon(:), semicolon(;). All separators are equal and have to be used according to context.
All the following sequences are equal:
```
If not REVENUE CONSTANT, stop.
Increment BALANCE by REVENUE CONSTANT.
```

```
If not REVENUE CONSTANT, stop. Increment BALANCE by REVENUE CONSTANT.
```

```
If not REVENUE CONSTANT,
stop;
Increment BALANCE
by REVENUE CONSTANT.
```

Comma(,) is not a separator and can be used in macros as a word.


## TERMS LANGUAGE PRIMITIVES
### Variables
Variable is one or more words starting with capital letter or underscore symbol. The rest of letters are converted to uppercase on preprocessor stage. For instance, BALANCE and Balance are the same variable. USER BALANCE and User Balance are the same, variable, too, but one that is different from BALANCE. It is up to contract writer to decide whether to use all-uppercase variables or not.

### Ethereum environment constants
There are some variables that are predefined by Ethereum environment. The all end with CONSTANT word, are directly compiled into EVM commands according to rules set in file terms.develop.txt:
```
>> CALLER CALLER
>> REVENUE CALLVALUE
>> STACKCOPY DUP1
>> STACK
>> RECORD SLOAD
>> BLOCKNUMBER NUMBER
>> CONTRACT ADDRESS
>> BALANCE BALANCE
>> ORIGIN ORIGIN
>> CALLDATASIZE CALLDATASIZE
>> GASPRICE GASPRICE
>> COINBASE COINBASE
>> TIMESTAMP TIMESTAMP
>> DIFFICULTY DIFFICULTY
>> GASLIMIT GASLIMIT
>> PC PC
>> MSIZE MSIZE
>> GAS GAS
>> GASLIMIT GASLIMIT
```
This means that to get message caller (CALLER opcode), we need to use CALLER CONSTANT variable. To get gas value of a call, we call GAS CONSTANT. To get current block number, we use BLOCKNUMBER CONSTANT (which is translated to opcode NUMBER).


#### Variable initialisation and scope
Variables are implicitly initialised in TERMS language. All variables are initialised with 0. That makes a shorter code but very error prone in hands of inaccurate developer with a wrong IDE. To avoid errors, we recommend using an IDE with autocomplete option and double check spelling of every variable.

TERMS language is dynamically typed. The only way to know the type of a variable is to infer it.
TERMS language does not use negative numbers. Most legal documents do not contain negatives, i.e. one cannot be sent -100 ether.


### Array members
One or more words followed by "#" and a decimal number is an array member. Those all are valid array members:
```
USER #1
USER#1
PARTY MEMBER #1
```
Convention in TERMS language is that count starts from #1.

### Sequences
Sequences, such as strings, are subset of arrays. This special kind of array has the following contents:
STRING PIECE #0   - contains length of STRING PIECE SEQUENCE
STRING PIECE #1   - contains first 32 bytes of the Unicode-encoded string.
STRING PIECE #2   - more 32 bytes
...

Note:  STRING PIECE SEQUENCE is an alias to STRING PIECE #0

### Events
Events are declared in constructor part of the code and follow the syntax rules of Solidity
```
event Transfer(address indexed _from, address indexed _to, uint256 _value);
```

Events are called by a set of log macros. Full set of all macros can be found in terms.develop.txt
```
Log TRANSFER EVENT with topics FROM, TO, data VALUE.
```

## Method signatures
Example of a method signature is:
```
constant balanceOf(address owner);
```
Here, variable OWNER is passed to the method. The name of the variable OWNER is inferred from balanceOf(address owner) method signature by capitalising the parameter's name. Every method has one and only one return statement

The "constant" modifier is to be applied to methods that don't change the state. It is brought from Solidity and Will likely be removed in later versions as it can be inserted programmatically on preprocessor stage.


## Clauses
Clauses are numbers optionally divided by dots, used exactly like in legal contracts. The depth of dots is unlimited, i.e. clause 1.1.1.1.1.1.1.1.1.1.1.2.100.2.4  is totally valid. Clause can be addressed by "see" macro. For example, the following would produce an infinite loop:
```
1.2.
See 1.2.
```

## If-else statements
There are kinds of if-else statements. For example, here is the method trim() that returns its input and guards that output is not greater than 100. There are few ways to write it.

If-else statement can be written in one line, as far as every branch is a single sentence.
```
constant trim(uint256 input)
If INPUT > 100, let OUTPUT = 100, else let OUTPUT = INPUT.
Return uint256 at OUTPUT.
```

More complex logic can be fit in the extended form of if-else statement.
```
constant trim(uint256 input)
If INPUT > 100:
Let OUTPUT = 100.
Else:
Let OUTPUT = INPUT
End.
Return uint256 at OUTPUT.
```

TERMS language allows using if without else. It is processed as if else branch was empty.
```
constant trim(uint256 input)
If INPUT <= 100, see 1.1.
Let INPUT = 100.
1.1. Return uint256 at INPUT.
```

There is a special kind if-else:  if not - else. Due to decisions made in EVM, it spends less gas and produce shorter opcode than the if-else version.
```
constant trim(uint256 input)
If not INPUT > 100, see 1.1.
Let INPUT = 100.
1.1. Return uint256 at INPUT.
```

## Procedures
Procedures provide the necessary abstraction for repeated sequences of calculations. They can also help producing smaller bytecode. Procedures take all available variables as their input and may change any of the variables, change state and to whatever is possible in the contract main section. Procedures are kept outside method bodies. It is recommended to keep procedures in a separate file. By convention, it has the same name as the main contract file, with extension .ttp instead of .tt
Here is a sample procedure:
```
Procedure "get allowance".
Let TMP = FROM - TO.
Let REMAINING read record TMP.
Procedure end.
```
A procedure starts with the word procedure followed by it's name in double quotes and ends with sentence "Procedure end." One is not allowed to call a procedure from another procedure. Recursive calls are disallowed either. Those all are gas saving decisions.


## More info
More info on current TERMS language implementation can be found in file
[terms.develop.txt] (https://github.com/termslang/termslang/blob/master/terms.develop.txt)


## Contacts
Feel free to contact me on matters of the TERMS language project.

Mikhail Baynov  
e-mail:      m.baynov@gmail.com
twitter:     @bzz
donate ETH:  0x701bc9829738a3a3feefe9e74294baa96b487d63