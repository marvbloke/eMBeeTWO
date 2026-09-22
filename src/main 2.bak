////////////////////////////////////////////////////////////////////////////////
// TinyBasic Plus - eMBee TWO Firmware Fork
////////////////////////////////////////////////////////////////////////////////
//
// Authors: 
//    Gordon Brandly (Tiny Basic for 68000)
//    Mike Field (Arduino Basic)
//    Scott Lawrence (TinyBasic Plus)
//    Matthew Begg (eMBee TWO Custom Firmware & OLED/Keypad Drivers)
//
// Optimized for ATmega328P (8MHz Internal Clock), SSD1306 OLED, 
// Internal AVR EEPROM + External AT24C256 I2C EEPROM (Slots 1-32)
//
// License: MIT License (see LICENSE file for full text)
//
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software.
////////////////////////////////////////////////////////////////////////////////

#define SIMULATOR_BUILD 1

#include <Arduino.h>
#include <Wire.h>
#include <EEPROM.h>
#include <SSD1306Ascii.h>
#include <SSD1306AsciiWire.h>
#include <avr/pgmspace.h>

#define I2C_ADDRESS     0x3C
#define EEPROM_I2C_ADDR 0x50
#define SLOT_SIZE       1024
#define kVersion        "V0.1"
#define kConsoleBaud    9600

SSD1306AsciiWire oled;

// Stream Selection
enum {
  kStreamSerial = 0,
  kStreamEEProm,        // Internal EEPROM stream
  kStreamExtEEProm     // External I2C EEPROM stream
};
static unsigned char inStream = kStreamSerial;
static unsigned char outStream = kStreamSerial;
static int eepos = 0;

// ASCII Definitions
#define CR      '\r'
#define NL      '\n'
#define LF      0x0A
#define TAB     '\t'
#define BELL    '\b'
#define SPACE   ' '
#define SQUOTE  '\''
#define DQUOTE  '\"'
#define CTRLC   0x03
#define CTRLH   0x08

typedef unsigned short LINENUM;

// Memory Buffers
#define kRamSize (RAMEND - 1160)
static unsigned char program[kRamSize];
static unsigned char *txtpos, *list_line, *tmptxtpos;
static unsigned char expression_error;
static unsigned char *tempsp;

static boolean inhibitOutput = false;
static boolean runAfterLoad = false;
static boolean triggerRun = false;

// Keyword Table
const static unsigned char keywords[] PROGMEM = {
  'L','I','S','T'+0x80,
  'L','O','A','D'+0x80,
  'N','E','W'+0x80,
  'R','U','N'+0x80,
  'S','A','V','E'+0x80,
  'F','O','R','M','A','T'+0x80,
  'C','L','S'+0x80,
  'N','E','X','T'+0x80,
  'L','E','T'+0x80,
  'I','F'+0x80,
  'G','O','T','O'+0x80,
  'G','O','S','U','B'+0x80,
  'R','E','T','U','R','N'+0x80,
  'R','E','M'+0x80,
  'F','O','R'+0x80,
  'I','N','P','U','T'+0x80,
  'P','R','I','N','T'+0x80,
  'P','O','K','E'+0x80,
  'S','T','O','P'+0x80,
  'B','Y','E'+0x80,
  'F','I','L','E','S'+0x80,
  'M','E','M'+0x80,
  '?'+ 0x80,
  '\''+ 0x80,
  'A','W','R','I','T','E'+0x80,
  'D','W','R','I','T','E'+0x80,
  'D','E','L','A','Y'+0x80,
  'E','N','D'+0x80,
  'R','S','E','E','D'+0x80,
  'C','H','A','I','N'+0x80,
  'E','C','H','A','I','N'+0x80,
  'E','L','I','S','T'+0x80,
  'E','L','O','A','D'+0x80,
  'E','F','O','R','M','A','T'+0x80,
  'E','S','A','V','E'+0x80,
  0
};

enum {
  KW_LIST = 0, KW_LOAD, KW_NEW, KW_RUN, KW_SAVE, KW_FORMAT, KW_CLS,
  KW_NEXT, KW_LET, KW_IF, KW_GOTO, KW_GOSUB, KW_RETURN, KW_REM,
  KW_FOR, KW_INPUT, KW_PRINT, KW_POKE, KW_STOP, KW_BYE, KW_FILES,
  KW_MEM, KW_QMARK, KW_QUOTE, KW_AWRITE, KW_DWRITE, KW_DELAY,
  KW_END, KW_RSEED, KW_CHAIN,
  KW_ECHAIN, KW_ELIST, KW_ELOAD, KW_EFORMAT, KW_ESAVE,
  KW_DEFAULT
};

struct stack_for_frame {
  char frame_type;
  char for_var;
  short int terminal;
  short int step;
  unsigned char *current_line;
  unsigned char *txtpos;
};

struct stack_gosub_frame {
  char frame_type;
  unsigned char *current_line;
  unsigned char *txtpos;
};

const static unsigned char func_tab[] PROGMEM = {
  'P','E','E','K'+0x80, 'A','B','S'+0x80, 'A','R','E','A','D'+0x80,
  'D','R','E','A','D'+0x80, 'R','N','D'+0x80, 0
};
#define FUNC_PEEK    0
#define FUNC_ABS     1
#define FUNC_AREAD   2
#define FUNC_DREAD   3
#define FUNC_RND     4
#define FUNC_UNKNOWN 5

const static unsigned char to_tab[] PROGMEM = { 'T','O'+0x80, 0 };
const static unsigned char step_tab[] PROGMEM = { 'S','T','E','P'+0x80, 0 };
const static unsigned char relop_tab[] PROGMEM = {
  '>','='+0x80, '<','>'+0x80, '>'+0x80, '='+0x80, '<','='+0x80, '<'+0x80, '!','='+0x80, 0
};

#define RELOP_GE        0
#define RELOP_NE        1
#define RELOP_GT        2
#define RELOP_EQ        3
#define RELOP_LE        4
#define RELOP_LT        5
#define RELOP_NE_BANG   6
#define RELOP_UNKNOWN   7

const static unsigned char highlow_tab[] PROGMEM = { 
  'H','I','G','H'+0x80, 'H','I'+0x80, 'L','O','W'+0x80, 'L','O'+0x80, 0
};
#define HIGHLOW_HIGH    1
#define HIGHLOW_UNKNOWN 4

#define STACK_SIZE (sizeof(struct stack_for_frame)*5)
#define VAR_SIZE sizeof(short int)

// Interpreter State Variables
static unsigned char *stack_limit;
static unsigned char *program_start;
static unsigned char *program_end;
static unsigned char *variables_begin;
static unsigned char *current_line;
static unsigned char *sp;
#define STACK_GOSUB_FLAG 'G'
#define STACK_FOR_FLAG   'F'
static unsigned char table_index;
static LINENUM linenum;

// PROGMEM Output Messages
static const unsigned char okmsg[]            PROGMEM = "OK";
static const unsigned char whatmsg[]          PROGMEM = "WHAT? ";
static const unsigned char howmsg[]           PROGMEM = "HOW?";
static const unsigned char sorrymsg[]         PROGMEM = "SORRY!";
static const unsigned char initmsg[]          PROGMEM = "EMBEE TWO " kVersion;
static const unsigned char memorymsg[]        PROGMEM = " BYTES RAM";
static const unsigned char eeprommsg[]        PROGMEM = " BYTES INT EEPROM";
static const unsigned char breakmsg[]         PROGMEM = "BREAK!";
static const unsigned char backspacemsg[]     PROGMEM = "\b \b";
static const unsigned char spacemsg[]         PROGMEM = " ";

static const unsigned char savedmsg[]     PROGMEM = "SAVED";
static const unsigned char loadedmsg[]    PROGMEM = "LOADED";
static const unsigned char formattedmsg[] PROGMEM = "EEPROM FORMATTED";
static const unsigned char noslotsmsg[]   PROGMEM = "NO SAVED SLOTS";
static const unsigned char slotmsg[]      PROGMEM = "SLOT ";
static const unsigned char usedmsg[]      PROGMEM = ": [USED]";

// Forward Declarations
static unsigned char *findline(void);
void printnum(int num);
void printmsgNoNL(const unsigned char *msg);
void printmsg(const unsigned char *msg);
static void line_terminator(void);
static short int expression(void);
static unsigned char breakcheck(void);
static int inchar(void);
static void outchar(unsigned char c);

static void ignore_blanks(void) {
  while(*txtpos == SPACE || *txtpos == TAB) txtpos++;
}

// --- External I2C EEPROM (AT24C256) Drivers ---
void extEEPROM_writeByte(uint16_t addr, uint8_t data) {
  Wire.beginTransmission(EEPROM_I2C_ADDR);
  Wire.write((uint8_t)(addr >> 8));   
  Wire.write((uint8_t)(addr & 0xFF)); 
  Wire.write(data);
  Wire.endTransmission();
  delay(5); 
}

uint8_t extEEPROM_readByte(uint16_t addr) {
  uint8_t data = 0xFF;
  Wire.beginTransmission(EEPROM_I2C_ADDR);
  Wire.write((uint8_t)(addr >> 8));   
  Wire.write((uint8_t)(addr & 0xFF)); 
  Wire.endTransmission();
  
  Wire.requestFrom((uint8_t)EEPROM_I2C_ADDR, (uint8_t)1);
  if (Wire.available()) data = Wire.read();
  return data;
}

void cmd_Format(void) {
  for (uint8_t slot = 1; slot <= 32; slot++) {
    uint16_t slotAddr = (slot - 1) * SLOT_SIZE;
    extEEPROM_writeByte(slotAddr, 0xFF);
  }
  printmsg(formattedmsg);
}

void cmd_SaveSlot(uint8_t slot) {
  if (slot < 1 || slot > 32) return;
  uint16_t startAddr = (slot - 1) * SLOT_SIZE;
  unsigned char *ptr = findline();
  uint16_t offset = 0;

  while (ptr != program_end) {
    uint8_t len = ptr[sizeof(LINENUM)];
    for (uint8_t i = 0; i < len; i++) {
      if (offset < (SLOT_SIZE - 1)) {
        extEEPROM_writeByte(startAddr + offset, ptr[i]);
        offset++;
      }
    }
    ptr += len;
  }
  extEEPROM_writeByte(startAddr + offset, 0xFF);

  printmsgNoNL(slotmsg);
  printnum(slot);
  printmsgNoNL(spacemsg);
  printmsg(savedmsg);
}

void cmd_LoadSlot(uint8_t slot) {
  if (slot < 1 || slot > 32) return;
  
  program_end = program_start; 
  uint16_t startAddr = (slot - 1) * SLOT_SIZE;
  uint16_t offset = 0;
  
  while (offset < SLOT_SIZE) {
    uint8_t b = extEEPROM_readByte(startAddr + offset);
    if (b == 0xFF) break; 
    
    *program_end = b;
    program_end++;
    offset++;
  }

  printmsgNoNL(slotmsg);
  printnum(slot);
  printmsgNoNL(spacemsg);
  printmsg(loadedmsg);
}

void cmd_FilesSlot(void) {
  uint8_t count = 0;
  for (uint8_t slot = 1; slot <= 32; slot++) {
    uint16_t startAddr = (slot - 1) * SLOT_SIZE;
    uint8_t header = extEEPROM_readByte(startAddr);
    
    if (header != 0xFF) {
      printmsgNoNL(slotmsg);
      printnum(slot);
      printmsg(usedmsg);
      count++;
    }
  }
  if (count == 0) {
    printmsg(noslotsmsg);
  }
}

// --- Helper Functions ---
static void scantable(const unsigned char *table) {
  int i = 0;
  table_index = 0;
  while(1) {
    if(pgm_read_byte( table ) == 0) return;

    if(txtpos[i] == pgm_read_byte( table )) {
      i++;
      table++;
    } else {
      if(txtpos[i]+0x80 == pgm_read_byte( table )) {
        txtpos += i+1;  
        ignore_blanks();
        return;
      }
      while((pgm_read_byte( table ) & 0x80) == 0) table++;
      table++;
      table_index++;
      ignore_blanks();
      i = 0;
    }
  }
}

static void pushb(unsigned char b) {
  sp--;
  *sp = b;
}

static unsigned char popb() {
  unsigned char b = *sp;
  sp++;
  return b;
}

void printnum(int num) {
  int digits = 0;
  if(num < 0) {
    num = -num;
    outchar('-');
  }
  do {
    pushb(num%10+'0');
    num = num/10;
    digits++;
  } while (num > 0);

  while(digits > 0) {
    outchar(popb());
    digits--;
  }
}

static unsigned short testnum(void) {
  unsigned short num = 0;
  ignore_blanks();
  while(*txtpos>= '0' && *txtpos <= '9' ) {
    if(num >= 0xFFFF/10) {
      num = 0xFFFF;
      break;
    }
    num = num *10 + *txtpos - '0';
    txtpos++;
  }
  return num;
}

static unsigned char print_quoted_string(void) {
  int i=0;
  unsigned char delim = *txtpos;
  if(delim != '"' && delim != '\'') return 0;
  txtpos++;

  while(txtpos[i] != delim) {
    if(txtpos[i] == NL) return 0;
    i++;
  }

  while(*txtpos != delim) {
    outchar(*txtpos);
    txtpos++;
  }
  txtpos++; 
  return 1;
}

void printmsgNoNL(const unsigned char *msg) {
  while( pgm_read_byte( msg ) != 0 ) {
    outchar( pgm_read_byte( msg++ ) );
  };
}

void printmsg(const unsigned char *msg) {
  printmsgNoNL(msg);
  line_terminator();
}

static void getln(char prompt) {
  outchar(prompt);
  txtpos = program_end + sizeof(LINENUM);

  while(1) {
    char c = inchar();
    switch(c) {
    case NL:
    case CR:
      line_terminator();
      txtpos[0] = NL;
      return;

    case CTRLH: 
    case 0x7F:  
      if (txtpos > program_end + sizeof(LINENUM)) {
        txtpos--;                   
        outchar(CTRLH);             
      }
      break;

    default:
      if (txtpos == variables_begin - 2) {
        outchar(BELL);
      } else {
        txtpos[0] = c;
        txtpos++;
        outchar(c);
      }
    }
  }
}

static unsigned char *findline(void) {
  unsigned char *line = program_start;
  while(1) {
    if(line == program_end) return line;
    if(((LINENUM *)line)[0] >= linenum) return line;
    line += line[sizeof(LINENUM)];
  }
}

static void toUppercaseBuffer(void) {
  unsigned char *c = program_end+sizeof(LINENUM);
  unsigned char quote = 0;

  while(*c != NL) {
    if(*c == quote) quote = 0;
    else if(*c == '"' || *c == '\'') quote = *c;
    else if(quote == 0 && *c >= 'a' && *c <= 'z') *c = *c + 'A' - 'a';
    c++;
  }
}

void printline() {
  LINENUM line_num = *((LINENUM *)(list_line));
  list_line += sizeof(LINENUM) + sizeof(char);

  printnum(line_num);
  outchar(' ');
  while(*list_line != NL) {
    outchar(*list_line);
    list_line++;
  }
  list_line++;
  line_terminator();
}

// --- Expression Evaluator ---
static short int expr4(void) {
  ignore_blanks();
  if( *txtpos == '-' ) {
    txtpos++;
    return -expr4();
  }

  if(*txtpos == '0') {
    txtpos++;
    return 0;
  }

  if(*txtpos >= '1' && *txtpos <= '9') {
    short int a = 0;
    do {
      a = a*10 + *txtpos - '0';
      txtpos++;
    } while(*txtpos >= '0' && *txtpos <= '9');
    return a;
  }

  if(txtpos[0] >= 'A' && txtpos[0] <= 'Z') {
    short int a;
    if(txtpos[1] < 'A' || txtpos[1] > 'Z') {
      a = ((short int *)variables_begin)[*txtpos - 'A'];
      txtpos++;
      return a;
    }

    scantable(func_tab);
    if(table_index == FUNC_UNKNOWN) goto expr4_error;

    unsigned char f = table_index;
    if(*txtpos != '(') goto expr4_error;

    txtpos++;
    a = expression();
    if(*txtpos != ')') goto expr4_error;
    txtpos++;

    switch(f) {
    case FUNC_PEEK:  return program[a];
    case FUNC_ABS:   return (a < 0) ? -a : a;
    case FUNC_AREAD: pinMode(a, INPUT); return analogRead(a);                        
    case FUNC_DREAD: pinMode(a, INPUT); return digitalRead(a);
    case FUNC_RND:   return random(a);
    }
  }

  if(*txtpos == '(') {
    short int a;
    txtpos++;
    a = expression();
    if(*txtpos != ')') goto expr4_error;
    txtpos++;
    return a;
  }

expr4_error:
  expression_error = 1;
  return 0;
}

static short int expr3(void) {
  short int a = expr4(), b;
  ignore_blanks();

  while(1) {
    if(*txtpos == '*') {
      txtpos++;
      b = expr4();
      a *= b;
    } else if(*txtpos == '/') {
      txtpos++;
      b = expr4();
      if(b != 0) a /= b;
      else expression_error = 1;
    } else return a;
  }
}

static short int expr2(void) {
  short int a, b;
  if(*txtpos == '-' || *txtpos == '+') a = 0;
  else a = expr3();

  while(1) {
    if(*txtpos == '-') {
      txtpos++;
      b = expr3();
      a -= b;
    } else if(*txtpos == '+') {
      txtpos++;
      b = expr3();
      a += b;
    } else return a;
  }
}

static short int expression(void) {
  short int a = expr2(), b;
  if(expression_error) return a;

  scantable(relop_tab);
  if(table_index == RELOP_UNKNOWN) return a;

  switch(table_index) {
  case RELOP_GE:      b = expr2(); if(a >= b) return 1; break;
  case RELOP_NE:
  case RELOP_NE_BANG: b = expr2(); if(a != b) return 1; break;
  case RELOP_GT:      b = expr2(); if(a > b) return 1; break;
  case RELOP_EQ:      b = expr2(); if(a == b) return 1; break;
  case RELOP_LE:      b = expr2(); if(a <= b) return 1; break;
  case RELOP_LT:      b = expr2(); if(a < b) return 1; break;
  }
  return 0;
}

// --- Main Execution Loop ---
void loop() {
  unsigned char *start, *newEnd;
  unsigned char linelen;
  boolean isDigital;
  int val;

  program_start = program;
  program_end = program_start;
  sp = program + sizeof(program);
  
  stack_limit = program + sizeof(program) - STACK_SIZE;
  variables_begin = stack_limit - 27 * VAR_SIZE;

  printnum(variables_begin - program_end);
  printmsg(memorymsg);

warmstart:
  current_line = 0;
  sp = program + sizeof(program);
  printmsg(okmsg);

prompt:
  if( triggerRun ){
    triggerRun = false;
    current_line = program_start;
    goto execline;
  }

  getln('>');
  toUppercaseBuffer();
  txtpos = program_end + sizeof(unsigned short);

  while(*txtpos != NL) txtpos++;

  {
    unsigned char *dest = variables_begin - 1;
    while(1) {
      *dest = *txtpos;
      if(txtpos == program_end + sizeof(unsigned short)) break;
      dest--;
      txtpos--;
    }
    txtpos = dest;
  }

  linenum = testnum();
  ignore_blanks();
  if(linenum == 0) goto direct;
  if(linenum == 0xFFFF) goto qhow;

  linelen = 0;
  while(txtpos[linelen] != NL) linelen++;
  linelen++; 
  linelen += sizeof(unsigned short) + sizeof(char); 

  txtpos -= 3;
  *((unsigned short *)txtpos) = linenum;
  txtpos[sizeof(LINENUM)] = linelen;

  start = findline();

  if(start != program_end && *((LINENUM *)start) == linenum) {
    unsigned char *dest = start, *from = start + start[sizeof(LINENUM)];
    unsigned tomove = program_end - from;
    while(tomove > 0) {
      *dest++ = *from++;
      tomove--;
    } 
    program_end = dest;
  }

  if(txtpos[sizeof(LINENUM)+sizeof(char)] == NL) goto prompt;

  while(linelen > 0) { 
    unsigned int tomove, space_to_make = txtpos - program_end;
    unsigned char *from, *dest;

    if(space_to_make > linelen) space_to_make = linelen;
    newEnd = program_end + space_to_make;
    tomove = program_end - start;

    from = program_end;
    dest = newEnd;
    while(tomove > 0) {
      from--; dest--;
      *dest = *from;
      tomove--;
    }

    for(tomove = 0; tomove < space_to_make; tomove++) {
      *start++ = *txtpos++;
      linelen--;
    }
    program_end = newEnd;
  }
  goto prompt;

qhow: 
  printmsg(howmsg);
  goto prompt;

qwhat:  
  printmsgNoNL(whatmsg);
  if(current_line != NULL) {
    unsigned char tmp = *txtpos;
    if(*txtpos != NL) *txtpos = '^';
    list_line = current_line;
    printline();
    *txtpos = tmp;
  }
  line_terminator();
  goto prompt;

qsorry: 
  printmsg(sorrymsg);
  goto warmstart;

run_next_statement:
  while(*txtpos == ':') txtpos++;
  ignore_blanks();
  if(*txtpos == NL) goto execnextline;
  goto interperateAtTxtpos;

direct: 
  txtpos = program_end + sizeof(LINENUM);
  if(*txtpos == NL) goto prompt;

interperateAtTxtpos:
  if(breakcheck()) {
    printmsg(breakmsg);
    goto warmstart;
  }

  scantable(keywords);

  switch(table_index) {
  case KW_DELAY:
    expression_error = 0;
    val = expression();
    delay(val);
    goto execnextline;

  case KW_FILES:
    cmd_FilesSlot();
    goto prompt;

  case KW_LIST:
    linenum = testnum();
    if(txtpos[0] != NL) goto qwhat;
    list_line = findline();
    while(list_line != program_end) printline();
    goto warmstart;

  case KW_CHAIN:
    expression_error = 0;
    val = expression();
    if (expression_error || val < 1 || val > 32) goto qhow;
    cmd_LoadSlot((uint8_t)val);
    current_line = program_start;
    goto execline;

  case KW_LOAD:
    expression_error = 0;
    val = expression();
    if (expression_error || val < 1 || val > 32) goto qhow;
    cmd_LoadSlot((uint8_t)val);
    goto warmstart;

  case KW_MEM:
    printnum(variables_begin - program_end);
    printmsg(memorymsg);
    printnum(E2END + 1);
    printmsg(eeprommsg);
    goto run_next_statement;

  case KW_NEW:
    if(txtpos[0] != NL) goto qwhat;
    program_end = program_start;
    goto prompt;

  case KW_RUN:
    current_line = program_start;
    goto execline;

  case KW_SAVE:
    expression_error = 0;
    val = expression();
    if (expression_error || val < 1 || val > 32) goto qhow;
    cmd_SaveSlot((uint8_t)val);
    goto warmstart;

  case KW_FORMAT:
    cmd_Format();
    goto warmstart;

  case KW_CLS:
    oled.clear();
    oled.setCursor(0, 0);
    goto run_next_statement;

  // --- Internal AVR EEPROM Commands ---
  case KW_EFORMAT:
    for (int i = 0; i <= E2END; i++) {
      EEPROM.write(i, 0);
    }
    printmsg(formattedmsg);
    goto warmstart;

  case KW_ESAVE:
    outStream = kStreamEEProm;
    eepos = 0;
    list_line = findline();
    while (list_line != program_end) {
      printline();
    }
    EEPROM.write(eepos, '\0');
    outStream = kStreamSerial;
    goto warmstart;

  case KW_ELOAD:
    program_end = program_start;
    eepos = 0;
    inStream = kStreamEEProm;
    inhibitOutput = true;
    goto warmstart;

  case KW_ECHAIN:
    runAfterLoad = true;
    program_end = program_start;
    eepos = 0;
    inStream = kStreamEEProm;
    inhibitOutput = true;
    goto warmstart;

  case KW_ELIST:
    for (int i = 0; i <= E2END; i++) {
      val = EEPROM.read(i);
      if (val == '\0') break;
      if (((val < ' ') || (val > '~')) && (val != NL) && (val != CR)) {
        outchar('?');
      } else {
        outchar(val);
      }
    }
    line_terminator();
    goto run_next_statement;

  case KW_NEXT:
    ignore_blanks();
    if(*txtpos < 'A' || *txtpos > 'Z') goto qhow;
    txtpos++;
    ignore_blanks();
    if(*txtpos != ':' && *txtpos != NL) goto qwhat;

    tempsp = sp;
    while(tempsp < program + sizeof(program) - 1) {
      if(tempsp[0] == STACK_FOR_FLAG) {
        struct stack_for_frame *f = (struct stack_for_frame *)tempsp;
        if(txtpos[-1] == f->for_var) {
          short int *varaddr = ((short int *)variables_begin) + txtpos[-1] - 'A'; 
          *varaddr += f->step;
          if((f->step > 0 && *varaddr <= f->terminal) || (f->step < 0 && *varaddr >= f->terminal)) {
            txtpos = f->txtpos;
            current_line = f->current_line;
            goto run_next_statement;
          }
          sp = tempsp + sizeof(struct stack_for_frame);
          goto run_next_statement;
        }
      }
      tempsp += sizeof(struct stack_for_frame);
    }
    goto qhow;

  case KW_LET:
  case KW_DEFAULT:
    {
      short int value, *var;
      if(*txtpos < 'A' || *txtpos > 'Z') goto qhow;
      var = (short int *)variables_begin + *txtpos - 'A';
      txtpos++;
      ignore_blanks();
      if (*txtpos != '=') goto qwhat;
      txtpos++;
      ignore_blanks();
      expression_error = 0;
      value = expression();
      if(expression_error || (*txtpos != NL && *txtpos != ':')) goto qwhat;
      *var = value;
    }
    goto run_next_statement;

  case KW_IF:
    expression_error = 0;
    val = expression();
    if(expression_error || *txtpos == NL) goto qhow;
    if(val != 0) goto interperateAtTxtpos;
    goto execnextline;

  case KW_GOTO:
    expression_error = 0;
    linenum = expression();
    if(expression_error || *txtpos != NL) goto qhow;
    current_line = findline();
    goto execline;

  case KW_GOSUB:
    expression_error = 0;
    linenum = expression();
    if(!expression_error && *txtpos == NL) {
      struct stack_gosub_frame *f;
      if(sp + sizeof(struct stack_gosub_frame) < stack_limit) goto qsorry;
      sp -= sizeof(struct stack_gosub_frame);
      f = (struct stack_gosub_frame *)sp;
      f->frame_type = STACK_GOSUB_FLAG;
      f->txtpos = txtpos;
      f->current_line = current_line;
      current_line = findline();
      goto execline;
    }
    goto qhow;

  case KW_RETURN:
    tempsp = sp;
    while(tempsp < program + sizeof(program) - 1) {
      if(tempsp[0] == STACK_GOSUB_FLAG) {
        struct stack_gosub_frame *f = (struct stack_gosub_frame *)tempsp;
        current_line = f->current_line;
        txtpos       = f->txtpos;
        sp += sizeof(struct stack_gosub_frame);
        goto run_next_statement;
      }
      tempsp += sizeof(struct stack_gosub_frame);
    }
    goto qhow;

  case KW_REM:
  case KW_QUOTE:
    goto execnextline;

  case KW_FOR:
    {
      unsigned char var;
      short int initial, step, terminal;
      ignore_blanks();
      if(*txtpos < 'A' || *txtpos > 'Z') goto qwhat;
      var = *txtpos++;
      ignore_blanks();
      if(*txtpos != '=') goto qwhat;
      txtpos++;
      ignore_blanks();

      expression_error = 0;
      initial = expression();
      if(expression_error) goto qwhat;

      scantable(to_tab);
      if(table_index != 0) goto qwhat;

      terminal = expression();
      if(expression_error) goto qwhat;

      scantable(step_tab);
      if(table_index == 0) {
        step = expression();
        if(expression_error) goto qwhat;
      } else step = 1;

      ignore_blanks();
      if(*txtpos != NL && *txtpos != ':') goto qwhat;

      if(!expression_error && *txtpos == NL) {
        struct stack_for_frame *f;
        if(sp + sizeof(struct stack_for_frame) < stack_limit) goto qsorry;
        sp -= sizeof(struct stack_for_frame);
        f = (struct stack_for_frame *)sp;
        ((short int *)variables_begin)[var-'A'] = initial;
        f->frame_type = STACK_FOR_FLAG;
        f->for_var = var;
        f->terminal = terminal;
        f->step     = step;
        f->txtpos   = txtpos;
        f->current_line = current_line;
        goto run_next_statement;
      }
    }
    goto qhow;

  case KW_INPUT:
    {
      unsigned char var;
      int value;
      ignore_blanks();
      if(*txtpos < 'A' || *txtpos > 'Z') goto qwhat;
      var = *txtpos++;
      ignore_blanks();
      if(*txtpos != NL && *txtpos != ':') goto qwhat;

inputagain:
      tmptxtpos = txtpos;
      getln('?');
      toUppercaseBuffer();
      txtpos = program_end + sizeof(unsigned short);
      ignore_blanks();
      expression_error = 0;
      value = expression();
      if(expression_error) goto inputagain;
      ((short int *)variables_begin)[var-'A'] = value;
      txtpos = tmptxtpos;
      goto run_next_statement;
    }

  case KW_PRINT:
  case KW_QMARK:
    if(*txtpos == ':' ) {
      line_terminator();
      txtpos++;
      goto run_next_statement;
    }
    if(*txtpos == NL) goto execnextline;

    while(1) {
      ignore_blanks();
      if(print_quoted_string()) { ; }
      else if(*txtpos == '"' || *txtpos == '\'') goto qwhat;
      else {
        short int e;
        expression_error = 0;
        e = expression();
        if(expression_error) goto qwhat;
        printnum(e);
      }

      if(*txtpos == ',') txtpos++;
      else if(txtpos[0] == ';' && (txtpos[1] == NL || txtpos[1] == ':')) {
        txtpos++;
        break;
      } else if(*txtpos == NL || *txtpos == ':') {
        line_terminator();
        break;
      } else goto qwhat;
    }
    goto run_next_statement;

  case KW_POKE:
    {
      short int value;
      unsigned char *addr;
      expression_error = 0;
      value = expression();
      if(expression_error) goto qwhat;
      addr = (unsigned char *)value;

      ignore_blanks();
      if (*txtpos != ',') goto qwhat;
      txtpos++;
      ignore_blanks();

      expression_error = 0;
      value = expression();
      if(expression_error || (*txtpos != NL && *txtpos != ':')) goto qwhat;
      *addr = (unsigned char)value;
    }
    goto run_next_statement;

  case KW_END:
  case KW_STOP:
    if(txtpos[0] != NL) goto qwhat;
    current_line = program_end;
    goto execline;

  case KW_BYE:
    return;

  case KW_AWRITE:
  case KW_DWRITE:
    {
      boolean isDigital = (table_index == KW_DWRITE);
      short int pinNo, value;
      expression_error = 0;
      pinNo = expression();
      if(expression_error) goto qwhat;

      ignore_blanks();
      if (*txtpos != ',') goto qwhat;
      txtpos++;
      ignore_blanks();

      scantable(highlow_tab);
      if(table_index != HIGHLOW_UNKNOWN) {
        value = (table_index <= HIGHLOW_HIGH) ? 1 : 0;
      } else {
        expression_error = 0;
        value = expression();
        if(expression_error) goto qwhat;
      }
      pinMode(pinNo, OUTPUT);
      if(isDigital) digitalWrite(pinNo, value);
      else analogWrite(pinNo, value);
    }
    goto run_next_statement;

  case KW_RSEED:
    {
      short int value;
      expression_error = 0;
      value = expression();
      if(expression_error) goto qwhat;
      randomSeed(value);
      goto run_next_statement;
    }

  default:
    break;
  }

execnextline:
  if(current_line == NULL) goto prompt;
  current_line += current_line[sizeof(LINENUM)];

execline:
  if(current_line == program_end) goto warmstart;
  txtpos = current_line + sizeof(LINENUM) + sizeof(char);
  goto interperateAtTxtpos;
}

static void line_terminator(void) {
  outchar(NL);
  outchar(CR);
}

void setup() {
  Serial.begin(kConsoleBaud);

  Wire.begin();
  Wire.setClock(400000L);
  oled.begin(&Adafruit128x64, I2C_ADDRESS);
  oled.setFont(lcd5x7);
  oled.setScrollMode(SCROLL_MODE_AUTO);
  oled.clear();

  printmsg(initmsg);

  // Auto-run program stored in internal EEPROM if available
  int val = EEPROM.read(0);
  if(val >= '0' && val <= '9') {
    program_end = program_start;
    inStream = kStreamEEProm;
    eepos = 0;
    inhibitOutput = true;
    runAfterLoad = true;
  }
} 

static unsigned char breakcheck(void) {
  if(Serial.available()) return Serial.read() == CTRLC;
  return 0;
}

static int inchar() {
  int v;
  switch(inStream) {
  case(kStreamEEProm):
    v = EEPROM.read(eepos++);
    if(v == '\0') {
      goto inchar_loadfinish;
    }
    if (v >= 'a' && v <= 'z') v -= 32;
    return v;

  case(kStreamSerial):
  default:
    while(1) {
      if(Serial.available()) {
        v = Serial.read();
        if (v >= 'a' && v <= 'z') v -= 32;
        return v;
      }
    }
  }

inchar_loadfinish:
  inStream = kStreamSerial;
  inhibitOutput = false;

  if(runAfterLoad) {
    runAfterLoad = false;
    triggerRun = true;
  }
  return NL;
}

static void outchar(unsigned char c) {
  // Output redirection to internal EEPROM during ESAVE
  if (outStream == kStreamEEProm) {
    EEPROM.write(eepos++, c);
    return;
  }

  if (c == '\n') Serial.write('\r');
  Serial.write(c);

  if (inhibitOutput) return;

  if (c == '\r') return;
  else if (c == '\n') {
    oled.println();
#if SIMULATOR_BUILD
    if (oled.row() >= 7) {
      oled.clear();
      oled.setCursor(0, 0);
      return;
    }
#endif
  } 
  else if (c == '\b' || c == 0x7F) {
    int col = oled.col();
    if (col >= 5) { 
      oled.setCursor(col - 5, oled.row());
      oled.print(' ');
      oled.setCursor(col - 5, oled.row());
    }
  } 
  else oled.write(c);
}