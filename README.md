# Autosar debug Step by Step

Guida per affrontare i problemi di autosar che ho riscontrato nel tempo. Avendo usato unicamente l'ambiente Vector e il microcontrollore Infineon AURIX TC375 farò riferimento a questo microcontrollore. Probabilmente alcune cose possono cambiare.

## Memfault exception

Se viene triggherato l'handler di memfault è molto probabile che si stia utilizzando l'mpu.

I casi fondamentalmente sono due:
	- Configurazione di accesso in lettura/scrittura/esecuzione sbagliata sull'applicazione, o per esser più precisi al memory region identifier (si trova in Runtime System/Os/MemoryProtection/MemoryRegions/<regione di interesse>/Memory Region Access Rights
	- Un uso di porte variabili che sono allocate su una memory region diversa dall'applicazione: la soluzione è cambiare la porta in modo che siano chiamate asincrone oppure allocare la variabile sulla partizione con meno diritti (un'applicazione trusted può sempre scrivere su una non trusted)


## Unhandled exception

Per capire la natura dell'eccezione bisogna osservare il valore in ExceptionSource, un intero senza segno a 32bit. I 16 bit alti sono la Class dell'eccezione, cioè la tipologia, e i 16 bit bassi la TIN(Trap Identification Number) dell'eccezione, cioè l'id specifico.
Controllando sul manuale della infineon si può quindi individuare il problema.

Un esempio di problema che ho riscontrato:
Configurando la memory protection unit ho dovuto spostare alcuni task in un'applicazione QM, non trusted. In particolare la gestione dei messaggi provenienti dal can bus.
Nel momento in cui si eseguiva il software scattava l'eccezione ExceptionSource=0x00030001.

Quindi Class = 0x0003 e TIN = 0x0001. Andando a controllare sul datasheet sembrerebbe che la partizione CSA fosse troppo piccola.

Un altro esempio molto comune è quando si vuole scrivere tra partizioni diverse di memoria senza averne il diritto. In questo caso scatta l'eccezione ExceptionSource=0x00010003.

Una tabella riassuntiva delle eccezioni presenti sul Tricore Aurix TC375 qui sotto


| Type                | Class  | TIN    | ExceptionSource | Name | Sync/Async | HW/SW | Definition                           |
|---------------------|--------|--------|-----------------|------|------------|-------|--------------------------------------|
| MMU                 | 0x0000 | 0x0000 | 0x00000000      | VAF  | Sync       | HW    | Virtual Address Fill                 |
| MMU                 | 0x0000 | 0x0001 | 0x00000001      | VAP  | Sync       | HW    | Virtual Address Protection           |
| Internal Protection | 0x0001 | 0x0001 | 0x00010001      | PRIV | Sync       | HW    | Priviliged Instruction               |
| Internal Protection | 0x0001 | 0x0002 | 0x00010002      | MPR  | Sync       | HW    | Memory Protection Read               |
| Internal Protection | 0x0001 | 0x0003 | 0x00010003      | MPW  | Sync       | HW    | Memory Protection Write              |
| Internal Protection | 0x0001 | 0x0004 | 0x00010004      | MPX  | Sync       | HW    | Memory Protection Execution          |
| Internal Protection | 0x0001 | 0x0005 | 0x00010005      | MPP  | Sync       | HW    | Memory Protection Peripheral Access  |
| Internal Protection | 0x0001 | 0x0006 | 0x00010006      | MPN  | Sync       | HW    | Memory Protection Null Address       |
| Internal Protection | 0x0001 | 0x0007 | 0x00010007      | GRWP | Sync       | HW    | Global Registrer Write Protection    |
| Instruction Error   | 0x0002 | 0x0001 | 0x00020001      | IOPC | Sync       | HW    | Illegal Opcode                       |
| Instruction Error   | 0x0002 | 0x0002 | 0x00020002      | UOPC | Sync       | HW    | Unimplemented Opcode                 |
| Instruction Error   | 0x0002 | 0x0003 | 0x00020003      | OPD  | Sync       | HW    | Invalid Operand Specification        |
| Instruction Error   | 0x0002 | 0x0004 | 0x00020004      | ALN  | Sync       | HW    | Data Address Alignment               |
| Instruction Error   | 0x0002 | 0x0005 | 0x00020005      | MEM  | Sync       | HW    | Invalid Local Memory Address         |
| Context Management  | 0x0003 | 0x0001 | 0x00030001      | FCD  | Sync       | HW    | Free Context List Depletion(FCX=LCX) |
| Context Management  | 0x0003 | 0x0002 | 0x00030002      | CDO  | Sync       | HW    | Call Depth Overflow                  |
| Context Management  | 0x0003 | 0x0003 | 0x00030003      | CDU  | Sync       | HW    | Call Depth Underflow                 |
|                     |        |        |                 | FCU  |            | HW    |                                      |
|                     |        |        |                 |      |            |       |                                      |
