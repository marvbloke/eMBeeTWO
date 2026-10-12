# eMBee TWO Firmware

This is an implementation of Tiny Basic, with a focus on support for
Arduino.  It was originally written by Gordon Brandly in the form
of "68000 Tiny Basic", and then ported to C by Michael Field as
"Arduino Basic", though still called "Tiny Basic" in the source
files. Finally it was known as TinyBasic Plus with changes made by Scott Lawrence.

This fork of TinyBasic Plus is designed specifically for the eMBee TWO which is a credit card sized computer that boots straight to BASIC. Here are the hardware specs:

- ATmega 328P microcontroller CPU - running at 8MHz
- 16-key multi-purpose keypad allowing numeric, keyword, alphabetic and symbol input
- 1KB RAM available to the user (1,024 bytes shared between the program listing and the @() array)
- 1KB internal EEPROM for storing one program
- 32KB external EEPROM for storing up to 32 programs
- Piezo sounder supporting key clicks and audio commands (with sound on/off switch)
- Ability to send and receive data from another eMBee TWO via Game Boy Link Cable
- Ability to send and receive data from a PC/Mac via UART 

## Full list of supported statements and functions

### System

- END 		- *stops execution from the program, also "STOP"*
- MEM		- *displays bytes of RAM available to @() array*
- NEW		- *clears the current program*
- LIST - *show the current program listing*
- RUN		- *executes the current program*


### Internal EEPROM
- EFORMAT	- clears the EEPROM memory
- ELOAD		- load the program in from EEPROM
- ESAVE		- save the current program to the EEPROM
- ELIST		- print out the contents of EEPROM
- ECHAIN	- load the program from EEPROM and run it


### External EEPROM
- FORMAT - clears all slots on the external EEPROM
- FILES		- 	*shows which slots 1-32 are in use*
- LOAD slot_number	- *loads a file from the external EEPROM*
- SAVE slot_number	- *saves the current program to the external EEPROM, overwriting
- CHAIN slot_number - *equivalent of: load slot_number, run*

### IO, Documentation
- INPUT variable	- *let the user input an expression (number or variable name)*
- PEEK( address )	- *get a value in memory*
- POKE address		- *set a value in memory*
- SEND - *send data to other eMBee TWO or PC/Mac*
- RECV - *receive data from other eMBee TWO or PC/Mac*
- PRINT expression	- *print out the expression, also "?"*
- REM stuff		- *remark/comment, also "'"*
- HELP - *display user manual, optionally with command*

### Expressions, Maths
- A=value, LET A=value	- *assign value to a variable*
- @(x)=value, LET @(x)=value - *assign value to an array in position x (up to the maximum available RAM)*
- +, -, \*, / - *Maths*
- <,<=,=,<>,!=,>=,> - *Comparisons*
- ABS( expression )  - *returns the absolute value of the expression*
- RSEED *or* RSEED( v ) - *sets the random seed to runtime or v*
- RND( m ) - *returns a random number from 0 to m*

### Control
- IF expression statement - *perform statement if expression is true*
- FOR variable = start TO end	- *start for block*
- FOR variable = start TO end STEP value - *start for block with step*
- NEXT - *end of for block*
- GOTO linenumber - *continue execution at this line number*
- GOSUB linenumber - *call a subroutine at this line number*
- RETURN	- *return from a subroutine*

### Pin IO 
- DELAY	timems - *wait (in milliseconds)*
- DWRITE pin,value - *set pin with a value (HIGH,HI,LOW,LO)*
- AWRITE pin,value - *set pin with analog value (pwm) 0..255*
- DREAD( pin ) - *get the value of the pin* 
- AREAD( analogPin ) - *get the value of the analog pin*


### Sound
- TONE freq,timems - *play "freq" for "timems" milleseconds (1000 = 1 second)*
- TONEW freq,timems - *same as above, but also waits for it to finish*
- NOTONE - *stop playback of all playing tones*

## Example programs

Here are a few example programs to get you started...

### #1 - PRIME

Displays all prime numbers up to 32,767

    10 CLS
    20 PRINT "PRIME NUMBERS"
    30 PRINT "2 ";
    40 N = 3
    50 D = 3
    60 IF D > (N / D) GOTO 100
    70 IF (N / D) * D = N GOTO 110
    80 D = D + 2
    90 GOTO 60
    100 PRINT N;" ";
    110 N = N + 2
    120 IF N > 0 GOTO 50
### #2 - HILO

Build up your jackpot by guessing the number between 1 and 100. You have 6 guesses for each number.  
*Inspired by or adapted from:
Ahl, David H. (Ed.). BASIC Computer Games: Microcomputer Edition. 
Workman Publishing, 1978.*

    10 CLS
    20 PRINT "HI-LO JACKPOT"
    30 LET E = 0
    40 PRINT "PRESS A KEY TO START"
    50 IF INKEY = 0 GOTO 50
    60 RSEED
    70 LET N = RND(100)+1
    75 CLS
    80 PRINT "THERE IS MONEY"
    83 PRINT "BETWEEN $1 AND $100"
    85 PRINT "IN THE JACKPOT."
    90 PRINT "YOU HAVE 6 CHANCES TO"
    95 PRINT "GUESS IT."
    100 FOR I = 1 TO 6
    110 PRINT "YOUR GUESS ";I;"?"
    120 INPUT G
    130 IF G = N GOTO 190
    140 IF G < N GOTO 170
    150 PRINT G;" IS TOO HIGH."
    160 GOTO 180
    170 PRINT G;" IS TOO LOW."
    180 NEXT I
    185 PRINT "YOU BLEW IT..."
    186 PRINT "THE JACKPOT WAS $"; N
    187 GOTO 220
    190 PRINT "GOT IT IN "; I; " GUESSES!"
    200 LET E = E + N
    210 PRINT "YOU WON $"; N; " - TOTAL EARNINGS: $"; E
    220 PRINT "PLAY AGAIN"
    225 PRINT "(1=YES/2=NO)?"
    230 INPUT K
    240 IF K = 1 GOTO 70
    250 PRINT "FINAL EARNINGS: $"; E
    260 END
### #3 - FIB

Shows all numbers in the Fibonacci sequence.

    10 CLS
    20 PRINT "FIBONACCI"
    30 LET A = 0
    40 LET B = 1
    50 PRINT A;" ";
    60 LET C = A + B
    70 IF C < 0 GOTO 110
    80 LET A = B
    90 LET B = C
    100 GOTO 50
    110 PRINT B
    120 END

### #4 - BATTLESHIPS

Play a classic game of Battleships (one player only)

    10 CLS:?"BATTLESHIPS"
    20 K=INKEY:IF K=0 GOTO 20
    30 RSEED:H=0:M=0
    40 FOR I=0 TO 62: @(I)=0: NEXT I
    60 FOR S=1 TO 4
    70 Z=3
    80 IF S=1 Z=4
    90 IF S=4 Z=2
    100 D=RND(2):W=1
    110 IF D=1 W=9
    120 IF D=0 X=RND(10-Z):Y=RND(7)
    130 IF D=1 X=RND(9):Y=RND(8-Z)
    140 C=0:B=Y*9+X
    150 FOR I=0 TO Z-1
    160 C=C+@(B+I*W):NEXT I
    170 IF C>0 GOTO 100
    180 FOR I=0 TO Z-1
    190 @(B+I*W)=1:NEXT I
    200 NEXT S
    210 CLS:?" 123456789";
    220 FOR Y=0 TO 6
    230 ?:?CHR(65+Y);".........";
    235 DELAY 20
    240 NEXT Y
    245 ?AT 4,11;"TARGET:";
    250 ?AT 0,11;"H:";H;" L:";12-H;" "
    260 ?AT 5,11;">  ";
    270 K=INKEY:IF K<65 GOTO 270
    280 IF K>71 GOTO 270
    290 Y=K-65:?AT 5,12;CHR(K);
    300 K=INKEY:IF K<49 GOTO 300
    310 IF K>57 GOTO 300
    320 X=K-49:?AT 5,13;CHR(K);:DELAY 200
    330 P=Y*9+X:V=@(P):IF V>1 GOTO 250
    340 IF V=1 GOTO 370
    350 @(P)=2:M=M+1:?AT 7,11;"MISS!";:DELAY 10:?AT Y+1,X+1;"O";
    360 GOTO 250
    370 @(P)=3:H=H+1:?AT 7,11;"HIT! ";:DELAY 10:?AT Y+1,X+1;"X";
    380 IF H<12 GOTO 250
    390 ?AT 7,11;"YOU WIN!"
    400 END

### #5 - LEMONADE

Based on the classic Apple II Lemonade Stand game from 1979

    10 CLS:?"LEMONADE STAND":?"PRESS KEY"
    20 IF INKEY=0 GOTO 20
    30 RSEED:C=200:D=1
    40 CLS:?"DAY ";D;"  $";:V=C:GOSUB 330:?
    50 W=RND(3):?"WEATHER: ";
    60 IF W=0 ?"SUNNY"
    70 IF W=1 ?"CLOUDY"
    80 IF W=2 ?"HOT & DRY"
    90 ?"COST: 2c/GLASS"
    100 ?AT 4,0;"GLASSES?   ";:INPUT G:GOSUB 350
    110 ?AT 4,0;"PRICE(c)?  ";:INPUT P:GOSUB 350
    120 ?AT 4,0;"SIGNS 15c? ";:INPUT N
    130 K=G*2+N*15:IF K<=C GOTO 150
    140 ?AT 6,0;"NO MONEY!":DELAY 900:?AT 7,0;"         ":GOTO 100
    150 C=C-K:CLS:?"SELLING..."
    160 R=60:IF W=1 R=30
    170 IF W=2 R=90
    180 R=R-P*2+N*5:IF R<0 R=0
    190 IF W=1 IF RND(100)<25 ?"STORM!":R=0:DELAY 900
    200 M=0:IF G=0 GOTO 230
    210 FOR J=1 TO G:IF RND(100)<R M=M+1
    220 NEXT J
    230 I=M*P:C=C+I:CLS:?"SOLD: ";M;"/";G
    235 ?"OUT:$";:V=K:GOSUB 330:?
    240 ?"IN: $";:V=I:GOSUB 330:?
    250 ?"CASH:$";:V=C:GOSUB 330
    260 ?AT 6,0;"PRESS KEY";
    270 IF INKEY=0 GOTO 270
    280 D=D+1:IF D<11 GOTO 40
    290 CLS:?"GAME OVER!":?"FINAL:$";:V=C:GOSUB 330
    300 END
    330 M=V/100:E=V-M*100:?M;".";:IF E<10 ?"0";
    340 ?E;:RETURN
    350 ?AT 4,11;"          ";:RETURN

### #6 - SYNTH

Write a small program, store it in EEPROM.  We'll show that variables don't
get erased when chaining happens

    10 CLS:?"POCKET SYNTH"
    20 ?"1-8:PLAY 9:DOWN 0:UP"
    30 V=2:M=2:?AT 3,0;"OCTAVE: ";V
    40 K=INKEY:IF K=0 GOTO 40
    50 IF K=57 GOTO 160
    60 IF K=48 GOTO 180
    70 F=0
    80 IF K=49 F=131
    90 IF K=50 F=147
    100 IF K=51 F=165
    110 IF K=52 F=175
    120 IF K=53 F=196
    130 IF K=54 F=220
    140 IF K=55 F=247
    150 IF K=56 F=262
    155 IF F>0 TONE F*M,200:?AT 5,0;"PLAYING: ";K-48
    157 GOTO 40
    160 IF V>1 V=V-1:M=M/2:?AT 3,8;V
    170 GOTO 40
    180 IF V<4 V=V+1:M=M*2:?AT 3,8;V
    190 GOTO 40
### #7 - TRADER

Armed with \$150 in cash and saddled with a \$100 loan that gathers 10% interest every time you set sail, your goal is to build a trading empire across four distinct ports. Buy low and sell high on food, fuel, and gems while managing your inventory, staying ahead of compounding debt, and surviving the constant threat of pirates who will strip your cargo hold clean.

    10 C=150:D=100:P=1:RSEED:FOR I=1 TO 6:@(I)=0:NEXT I
    20 @(1)=RND(10)+5:@(2)=RND(30)+15:@(3)=RND(60)+40
    30 CLS:?"PORT:";P;" $";C;" DBT:";D
    40 ?"1.FOOD $";@(1);" INV:";@(4):?"2.FUEL $";@(2);" INV:";@(5):?"3.GEMS $";@(3);" INV:";@(6)
    50 ?"1BUY 2SELL 3GO 4LOAN"
    60 K=INKEY:IF K=49 GOTO 200
    70 IF K=50 GOTO 300
    80 IF K=51 GOTO 400
    90 IF K=52 GOTO 500
    100 GOTO 60
    200 GOSUB 600
    210 X=@(I)*Y:IF C<X GOTO 30
    220 C=C-X:@(I+3)=@(I+3)+Y:GOTO 30
    300 GOSUB 600
    310 IF @(I+3)<Y GOTO 30
    320 C=C+(@(I)*Y):@(I+3)=@(I+3)-Y:GOTO 30
    400 P=P+1:IF P>4 P=1
    410 D=D+(D/10)
    420 IF RND(10)>7 ?AT 7,0;"PIRATES!";:TONEW 99,500:@(4)=0:@(5)=0:@(6)=0:DELAY 999
    430 GOTO 20
    500 ?AT 7,0;"1PAY 2BORROW(50) ";
    510 K=INKEY:IF K=49 GOTO 550
    520 IF K=50 C=C+50:D=D+50:GOTO 30
    530 GOTO 510
    550 X=50:IF D<50 X=D
    560 IF C>=X C=C-X:D=D-X
    570 GOTO 30
    600 ?AT 5,0;"ITEM(1-3)?";
    610 K=INKEY:IF K<49 GOTO 610
    620 IF K>51 GOTO 610
    630 I=K-48:?AT 6,10;CHR(K)
    640 ?AT 6,0;"AMOUNT(1-9)?";
    650 K=INKEY:IF K<49 GOTO 650
    660 IF K>57 GOTO 650
    670 Y=K-48:?AT 6,12;CHR(K):DELAY 999:RETURN

## Authors and Contributors

- Tiny Basic 68k - Gordon Brandly [Project Page (via archive.org)](https://web.archive.org/web/20170306113457/http://members.shaw.ca:80/gbrandly/68ktinyb.html)
- Arduino Basic / Tiny Basic C - Michael Field [Project Page](http://hamsterworks.co.nz/mediawiki/index.php/Arduino_Basic)
- Tiny Basic Plus - Scott Lawrence <yorgle@gmail.com> [Github Page](http://github.com/BleuLlama/TinyBasicPlus)

- Jurg Wullschleger - Fix for unary operations and whitespace in expressions
- Matthew Begg - eMBee TWO-specific code

## MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
