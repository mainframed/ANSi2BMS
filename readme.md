# ANSI To CICS BMS Map

This python script allows you to create CICS Basic Maps (BMS) using a ANSI art editor. The supported editor is Moebius but should work with other editors like PabloDraw.

It was developed because there were no free open source BMS creators that supported color, etc. (And specifically for DOGECICS).

## Example

Include with this repo is an example ANSI screen `ansi-test-map.ans`:

<img>

Which when compiled and loaded in CICS looks like:

<img>


## Understanding BMS

When you make a CICS application you typically want to show stuff on the screen. Do do this you use HLASM and some IBM (or KICKS) provided macros, namely `DFHMSD`, `DFHMDI` and `DFHMDF`:

`DFHMSD` defines a new **mapset** and looks like this:

```hlasm
TESTLN   DFHMSD TYPE=&SYSPARM,                                         X
               MODE=INOUT,                                             X
               LANG=COBOL,                                             X
               DSATTS=(COLOR),MAPATTS=(COLOR,HILIGHT),                 X
               TIOAPFX=YES,                                            X
               CTRL=FREEKB 
```
The symbol `TESTLN` will be how we refer to this **mapset** in COBOL/CICS

`DFHMDI` defines a partition (i.e. **map**)

```hlasm
TESTLN1  DFHMDI SIZE=(24,80),                                          X
               LINE=1,                                                 X
               COLUMN=1
```

The symbol `TESTLN1` will be how we refer to this **map** in COBOL/CICS. 

For example:

```cobol
EXEC CICS SEND 
    MAP('TESTLN1')
    MAPSET('TESTLN')
    ERASE
END-EXEC
```

`DFHMDF` defines a field and is used to put characters on the screen and setup input/output variables.

```hlasm
         DFHMDF POS=(10,22),                                           X
               LENGTH=07,                                              X
               COLOR=NEUTRAL,                                          X
               ATTRB=(NORM,PROT),                                      X
               INITIAL='Symbol:'
```

The above puts the text `Symbol:` on the screen at row 10, column 22 with a neutral color (white in x3270).

```hlasm
SYMBOL   DFHMDF POS=(10,30),                                           X
               LENGTH=09,                                              X
               COLOR=TURQUOISE,                                        X
               HILIGHT=UNDERLINE,                                      X
               ATTRB=(NORM,UNPROT,FSET),                               X
               INITIAL='APPL     '
         DFHMDF POS=(10,40),LENGTH=1,ATTRB=ASKIP
```

The above creates an input variable you can use in COBOL (`SYMBOLI`) at row 10, column 30, in turquoise.

```hlasm
FIELD1   DFHMDF POS=(12,16),                                           X
               LENGTH=14,                                              X
               COLOR=GREEN,                                            X
               ATTRB=(NORM,PROT),                                      X
               INITIAL='Company Name :'
```

This creates a variable (`FIELD1O`) that can be used to replace `Company Name :` with any output text you want. 

## Using this script

First, use Mobieus to design your CICS screen. Either intense or non-intense colors, doesn't matter. However, they will be converted to supported tn3270e colors. **Note**: You cannot change colors without putting a space between them due to the way `DFHMDF` puts text on a tn3270 screen (SFE vs SF), also, there's some limitation on character sets, if you only use characters available on your keyboard you should be fine, there's no extended character support.

To add variables use an appropriate background color described below (should work with intense colors as well but untested):
* Dark Red: User input with underline, insert the cursor here
* Dark Green: User input with underline
* Purple: User input with underline, delete placeholder text.
* Dark Blue: Protected field, output variable
* Dark Turquoise: Protected field, output variable, delete placeholder text.

Save your ANSI to any filename, for example: `invoice-display.ans`.

Then, you'll have to decide that to call your mapset and map. For this example we'll use mapset: `invce` and `invce1`. **Note**: the script will convert these to uppercase.

Call the script with three arguments:

`./ansi_to_bms.py invce invce1 invoice-display.ans`

At this point the script will then diplay any text that had a background color set and prompt you to enter a variable (i.e. symbol) name:

```
Initial='INVOICE     '
Input variable/symbol name:
```

Enter any variable name you'd like. The variables you use in COBOL will have I/O appended to them. 

Once done entering variables the script will print the HLASM/BMS to the screen and to ta text file. In our example it would output `invoice-display.ans.bms`.

Upload this to **TK4-** or **z/OS** using whichever method (i prefer FTP, which on TK4- can be started with `/s ftpd,srvport=21021` in the hercules console, if you've disabled daemon mode).

Assuming you uploaded this to `CICS.TESTING(INVOICE)` you can use the following JCL to assemble it:


**z/OS** (more detail here: https://gist.github.com/mainframed/a8e94ec1e2d791eaf96d9aac981e2c10):

```jcl
//COMPBMS  JOB 'Assemble map',NOTIFY=&SYSUID
//IBMLIB   JCLLIB ORDER=DFH520.CICS.SDFHPROC
//* make sure that the MAPLIB is pointed
//* to your CICS loadlib. If you are not sure about this
//* you can go to The JESJCL of the job for your CICS
//* region in Spool and check for DFHRPL dd statement.
//CPLSTP   EXEC DFHMAPS,MAPLIB='USER.CICSLOAD',
//         INDEX='DFH520.CICS',
//         DSCTLIB='DOGE.CICS',
//         MAPNAME='INVCE'
//* MAPLIB is where the compiled CICS BMS MAP is stored!
//* The DD below is the source of the program to assemble!
//COPY.SYSUT1  DD DSN=CICS.TESTING(INVOICE),DISP=SHR
```

**TK4- with KICKS**
```jcl
//KIKMAPJ  JOB  CLASS=A,MSGCLASS=H,MSGLEVEL=(1,1),NOTIFY=HERC01
//JOBPROC  DD   DSN=HERC01.KICKSSYS.V1R5M0.PROCLIB,DISP=SHR
//* CHANGE MAPNAME= TO WHAT YOU USED IN THE SCRIPT
//MAPPYMAP EXEC KIKMAPS,MAPNAME=INVCE	
//COPY.SYSUT1 DD DISP=SHR,DSN=CICS.TESTING(INVOICE)
```

## Script output without any arguments:

```
ANSI to CICS BMS HLASM Map tool.

Requires an ANSI file (tested with the Moebius ANSI editor).

For map variables set the background color you'd like to use. The field starts with the first character and ends when the baground or text ends, you will be prompted for variable a name.

The input background colors determine what its for:
	Dark Red: User input, insert cursor here
	Dark Green: User input with underline
	Dark Blue: Protected field
	Purple: User input with underline, delete placeholder text.
	Dark Turquoise: Protected field, delete placeholder text.

See example ANSI file: ansi-test-map.ans

Note: Colors cannot be in contigous blocks, you need a space for every color change.

Usage:
	ansi_to_bms.py <MAP name e.g. DOGETR max 7 chars> <MAPSET e.g. DOGETR1 max 7 chars> some_ansi_file.ansi
```

