# TOPICS

- [x] TIMER1
  - [ ] Practice
- ADC
- UART

## Prerequisites

```c
include <xc.h>

// Configuration bits
#pragma config FOSC = XT   // Oscillator Selection bits (XT oscillator)
#pragma config WDTE = OFF  // Watchdog Timer Enable bit (WDT disabled)
#pragma config PWRTE = ON  // Power-up Timer Enable bit (PWRT enabled)
#pragma config BOREN = ON  // Brown-out Reset Enable bit (BOR enabled)
#pragma config LVP = OFF   // Low Voltage Programming Enable bit (disabled)
#pragma config CPD = OFF   // Data EEPROM Memory Code Protection bit (disabled)
#pragma config WRT = OFF   // Flash Program Memory Write Enable bits (Write protection off)
#pragma config CP = OFF    // Flash Program Memory Code Protection bit (disabled)
```

# TIMER1

```c
void interrupt ISR(void)
{
    GIE = 0; // disable all unmasked interrupts (INTCON reg)
    if(TMR1IF==1) // checks Timer1 interrupt flag
    {
        TMR1IF = 0; // clears interrupt flag
        TMR1 = 0x0BDC; // timer will start counting at 0x0BDC (3036)
        RA0 = RA0^1; // toggles the LED at RA0
    }
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
}

void main(void)
{
    ADCON1 = 0x6; // set all pins in PORTA as digital I/O
    TRISA = 0x00; // sets all of PORTA to output
    RA0 = 0; // initialize RA0 to 0 (LED off)
    T1CON = 0x30; // 1:8 prescaler, internal clock, Timer1 off
    TMR1IE = 1; // enable Timer1 overflow interrupt (PIE1 reg)
    TMR1IF = 0; // reset interrupt flag (PIR1 reg)
    PEIE = 1; // enable all peripheral interrupt (INTCON reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
    TMR1 = 0x0BDC; // counter starts counting at 0x0BDC (3036)
    TMR1ON = 1; // Turns on Timer1 (T1CON reg)
    for(;;) // foreground routine
    {
    }
}
```

## TIMER 2

- does not generate interrupt upon overflow
- generates an interrupt upon value equal to the period register (PR2) and resets to `00h`
- PR2 can be read or written to and is also an 8-bit register

```c
void interrupt ISR(void)
{
    GIE = 0; // disable all unmasked interrupts (INTCON reg)
    if(TMR2IF==1) // checks Timer2 interrupt flag (TMR2=PR2)
        {
        TMR2IF = 0; // clears interrupt flag
        RA0 = RA0^1; // toggles the output signal at RA0
        }
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
}

void main(void)
{
    ADCON1 = 0x6; // set all pins in PORTA as digital I/O
    TRISA = 0x00; // sets all of PORTA to output
    RA0 = 0; // initialize RA0 to 0
    T2CON = 0x01; // 1:4 prescaler, Timer2 off
    TMR2IE = 1; // enable Timer2/PR2 match interrupt (PIE1 reg)
    TMR2IF = 0; // reset interrupt flag (PIR1 reg)
    PEIE = 1; // enable all peripheral interrupt (INTCON reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
    PR2 = 0x7D; // match value for TMR2(125)at half cycle
    TMR2ON = 1; // Turns on Timer2 (T2CON reg)
    for(;;) // foreground routine
    {
    }
}
```

## CCP MODULE

- 2 ccp modules: CCP1 and CCP2
- both are identical in operation, with the exception being the operation of the special event trigger

### CCP1 Module

- Capture/Compare/PWM Register 1 (CCPR1) is comprised of two 8-bit registers: CCPR1L and CCPR1H
- CCP1CON register controls the operation of CCP1.
- special event trigger is generated by a compare match and will reset Timer1

### CCP2 Module

- same as CCP1 two 8-bit registers CCPR2L and CCPR2H
CCP2CON register controls the operation of CCP2
- special event trigger is generated by a compare match and will reset Timer1 and start an A/D conversion

### Capture Mode

- captures the 16bit alue of the TMR1 register when an event occurs on pin RC2/CCP1
- Event:
  - every falling edge
  - every rising edge
  - every 4th rising edge
  - every 16th rising edge
- event type is configured by control bits CCP1M3:CCP1M0

### EXAMPLE (CAPTURE)

```c
void interrupt ISR(void)
{
    int period = 0;
    GIE = 0; // disable all unmasked interrupts (INTCON reg)
    if(CCP1IF==1) // checks CCP1 interrupt flag
        {
        CCP1IF = 0; // clears interrupt flag
        TMR1 = 0; // resets TMR1
        period = CCPR1/1000; // transfers captured TMR1 value
        // normalize the value (make the number smaller)
        period = period*8; // multiply by the normalized TMR1 timeout
        }
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
}
void main(void)
{
    TRISC = 0x04; // set RC2 to input
    T1CON = 0x30; // 1:8 prescaler, Timer1 off
    CCP1CON = 0x05; // capture mode: every rising edge
    CCP1IE = 1; // enable TMR1/CCP1 match interrupt (PIE1 reg)
    CCP1IF = 0; // reset interrupt flag (PIR1 reg)
    PEIE = 1; // enable all peripheral interrupt (INTCON reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
    TMR1ON = 1; // Turns on Timer1 (T1CON reg)
    for(;;) // foreground routine
    {
    }
}
```

### Example Compare

```c
void interrupt ISR(void)
{
    GIE = 0; // disable all unmasked interrupts (INTCON reg)
    if(CCP1IF==1) // checks CCP1 interrupt flag
    {
        CCP1IF = 0; // clears interrupt flag
        RA0 = RA0^1; // toggles the output signal at RA0
    }
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
}

void main(void)
{
    ADCON1 = 0x6; // set all pins in PORTA as digital I/O
    TRISA = 0x00; // sets all of PORTA to output
    RA0 = 0; // initialize RA0 to 0
    T1CON = 0x20; // 1:4 prescaler, Timer1 off
    CCP1CON = 0x0A; // compare mode: generate interrupt on match
    CCP1IE = 1; // enable TMR1/CCP1 match interrupt (PIE1 reg)
    CCP1IF = 0; // reset interrupt flag (PIR1 reg)
    CCPR1 = 0x4E2; // set the match value to TMR1
    PEIE = 1; // enable all peripheral interrupt (INTCON reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
    TMR1ON = 1; // Turns on Timer1 (T1CON reg)
    for(;;) // foreground routine
    {
    }
}
```

### Example PWM

```c
void main(void)
{
    /* following the steps in setting up PWM */
    PR2 = 0x7C; // set value for PR2
    CCPR1L = 0x57; // set value for (8 MSBs)
    CCP1CON = 0x2C; // set value for (2 LSBs), PWM mode
    TRISC = 0x00; // sets all of PORTC (RC2) to output
    RC2 = 0; // initialize RC2 to 0
    T2CON = 0x06; // 1:16 prescaler, Timer2 on
    for(;;) // foreground routine
    {
    }
}
```

# Analog To Digital Converter

- AD conversion is the process of converting continously varying signal such as voltage or current into discrete digital quantities that represent the magnitude of the signal compared to a standard or reference voltage

## Conversion Methods

- Flash
  - uses multiple comparators in parallel
  - if analog signal is higher than the known signal, the output of the comparator goes to 1
- Integrating
  - this A/D converter type charges a capacitor for a given amount of time using the analog signal
  - discharges back to zero with a known voltage and the counter provides the value of the unknown signal

### Example A/D Module

```c
int readADC(void)
{
    int temp = 0;
    delay(1000); // delay before reading value
    GO = 1; // start A/D conversion (ADCON0 reg)
    while(GO_DONE==1); // wait until conversion is done (ADCON0 reg)
    /* read result register */
    temp = ADRESH; // read ADRESH
    temp = temp << 8; // move to correct position
    temp = temp | ADRESL; // read ADRESL
    return temp;
}
void delay(int cnt)
{
    while(cnt--);
}

void main(void)
{
    int d_value = 0;
    TRISB = 0x00; // set all PORTB as output
    PORTB = 0X00; // all LEDs OFF
    ADCON1 = 0x80; // result register: right justified, clock: FOSC/2
    // all ports in PORTA are analog
    // VREF+=VDD, VREF-=VSS
    ADCON0 = 0x01; // clock: FOSC/2, analog channel: AN0
    // A/D conversion: STOP, A/D module: ON
    for(;;) // foreground routine
    {
        d_value = readADC();
        /* setting the LEDs */
        if(d_value>=0 && d_value<=169)
        PORTB = 0x00; // all LEDs OFF
        else if(d_value>=170 && d_value<=340)
        PORTB = 0x01; // RB0 LED ON
        ...
        ...
    }
}
```

### ADC with Interrupt

```c
void interrupt ISR(void)
{
    int d_value = 0;
    GIE = 0; // disable all unmasked interrupts (INTCON reg)
    if(ADIF==1) // checks CCP1 interrupt flag
    {
        ADIF = 0; // clears interrupt flag (INTCON reg)
        /* read result register */
        d_value = ADRESH; // read ADRESH
        d_value = d_value << 8; // move to correct position
        d_value = d_value | ADRESL; // read ADRESL
        /* setting the LEDs */
        if(d_value>=0 && d_value<=169)
        PORTB = 0x00; // all LEDs OFF
        else if(d_value>=170 && d_value<=340)
        PORTB = 0x01; // RB0 LED ON
        ...
        ...
    }
    GO = 1; // restart A/D conversion (ADCON0 reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
}


void main(void)
{
    TRISB = 0x00; // set all PORTB as output
    PORTB = 0x00; // all LEDs OFF
    ADCON1 = 0x80; // result register: right Justified, clock: FOSC/8
    // all ports in PORTA are analog
    // VREF+=VDD, VREF-=VSS
    ADCON0 = 0x41; // clock: FOSC/8 analog channel: AN0
    // A/D conversion: STOP, A/D module: ON
    ADIE = 1; // A/D conversion complete interrupt enable (PIE1 reg)
    ADIF = 0; // reset interrupt flag (PIR1 reg)
    PEIE = 1; // enable all peripheral interrupt (INTCON reg)
    GIE = 1; // enable all unmasked interrupts (INTCON reg)
    GO = 1; // start A/D conversion (ADCON0 reg)
    for(;;) // foreground routine
    {
    }
}

```

# USART

- not a communication protocol like SPI and I2C but a physical circuit

## Synchronous

- requires sender and receiver share a clock
- more efficient as only data bits are transmitted
- can be more costly due to extra wiring and circuts are required

## Asynchronous

- data can be transmitted without sender having to send a clock signalto the receiver
- sender and receiver agree on timing parameters in advance and special bits
- LSB are sent first
- transmitter may add a parity bit

### Example USART Asynchronous Tranmission Transmit

```c
void main(void)
{
    SPBRG = 0x19; // 9.6K baud rate @ FOSC=4MHz high speed
    // (see formula in Table 10-1)
    SYNC = 0; // asynchronous mode (TXSTA reg)
    TX9 = 0; // 8-bit transmission (TXSTA reg)
    TXEN = 1; // transmit enable (TXSTA reg)
    BRGH = 1; // asynchronous high speed
    SPEN = 1; // enable serial port (RCSTA reg)
    for(;;) // foreground routine
    {
        while(!TRMT); // wait until TSR buffer is empty
        TXREG = ‘A’; // send character ASCII code of ‘A’
    }
}

```

### Example Asynchronous Mode (Receive)

```c
void main(void)
{
    SPBRG = 0x19; // 9.6K baud rate @ FOSC=4MHz high speed
    // (see formula in Table 10-1)
    SYNC = 0; // asynchronous mode (TXSTA reg)
    BRGH = 1; // asynchronous high speed
    SPEN = 1; // enable serial port (RCSTA reg)
    CREN = 1; // enable continuous receive (RCSTA reg)
    RX9 = 0; // 8-bit reception (RCSTA reg)
    TRISB = 0x00; // set all ports in PORTB to output
    PORTB = 0x00; // initial value of PORTB
    for(;;) // foreground routine
    {
        while(!RCIF); // what until the data receive is complete
        PORTB = RCREG; // read RCREG
    }
}
```

# I2C

### Example Initialization Master Mode

```c
/* I2C initialization sequence*/
void init_I2C_Master(void)
{
TRISC3 = 1; // set RC3 (SCL) to input
TRISC4 = 1; // set RC4 (SDA) to input
SSPCON = 0x28; // SSP enabled, I2C master mode
SSPCON2 = 0x00; // start condition idle, stop condition idle
// receive idle
SSPSTAT = 0x00; // slew rate enabled
SSPADD = 0x09; // clock frequency at 100 KHz (FOSC = 4MHz)
}
```

### i2c wait

```c
void I2C_Wait(void)
{
/* wait until all I2C operations are finished*/
while((SSPCON2 & 0x1F) || (SSPSTAT & 0x04));
}
```

### Start and Stop Conditions

```c
void I2C_Start(void)
{
/* wait until all I2C operations are finished*/
I2C_Wait();
/* enable start condition */
SEN = 1; // SSPCON2
}
void I2C_RepeatedStart(void)
{
/* wait until all I2C operations are finished*/
I2C_Wait();
/* enable repeated start condition */
RSEN = 1; // SSPCON2
}
void I2C_Stop(void)
{
/* wait until all I2C operations are finished*/
I2C_Wait();
/* enable stop condition */
PEN = 1; // SSPCON2
}
```

### Transmit Master Mode

```c
void I2C_Send(unsigned char data)
{
/* wait until all I2C operations are finished*/
I2C_Wait();
/* write data to buffer and transmit */
SSPBUF = data;
}
```

### Receive Master Mode

```c
unsigned char I2C_Receive(unsigned char ack)
{
unsigned char temp;
I2C_Wait(); // wait until all I2C operations are finished
RCEN = 1; // enable receive (SSPCON2 reg)
I2C_Wait(); // wait until all I2C operations are finished
temp = SSPBUF; // read SSP buffer
I2C_Wait(); // wait until all I2C operations are finished
ACKDT = (ack)?0:1; // set ACK or NACK
ACKEN = 1; // enable acknowledge sequence
return temp;
}
```

### Send Receive Example master mode

```c
void main(void)
{
TRISB = 0x00; // set all bits in PORTB to output
PORTB = 0x00; // set all LEDs to off
TRISD = 0xFF; // set all bits in PORTD to input
init_I2C_Master(); // initialize I2C as master
for(;;)
{
    I2C_Start(); // initiate start condition
    I2C_Send(0x10); // send the slave address + write
    // for error handling, check if slave sent ACK bit via ACKSTAT bit
    // in SSPCON2 (for an address match)
    I2C_Send(PORTD); // send 8-bit data frame
    I2C_Stop(); // initiate stop condition
    __delay_ms(200); // delay before next operation
    I2C_Start(); // initiate start condition
    I2C_Send(0x11); // send the slave address + read
    // for error handling, check if slave sent ACK bit via ACKSTAT bit
    // in SSPCON2 (for an address match)
    PORTB = I2C_Receive(0); // read data and not acknowledge (NACK)
    // end of read operation
    // write received data to PORTB
    I2C_Stop(); // initiate stop condition
    __delay_ms(200); // delay before next operation
}
}
```

### Initilization (Slave Mode)

```c
/* I2C initialization sequence*/
void init_I2C_Slave(unsigned char slave_add)
{
TRISC3 = 1; // set RC3 (SCL) to input
TRISC4 = 1; // set RC4 (SDA) to input
SSPCON = 0x36; // SSP enabled, CKP release clock (SCK)
// I2C slave mode 7-bit address
SSPCON2 = 0x01; // start condition idle, stop condition idle
// receive idle
SSPSTAT = 0x80; // slew rate control disabled
SSPADD = slave_add; // 7-bit slave address
SSPIE = 1; // enable SSP interrupt
SSPIF = 0; // clear interrupt flag
PEIE = 1; // enable peripheral interrupt
GIE = 1; // enable unmasked interrupt
}
```

### Receive interrupts Slave mode

```c
void interrupt ISR(void)
{
    unsigned char temp;
    CKP = 0; // hold clock low (SSPCON reg)
    if(WCOL || SSPOV) // check if overflow or data collision (SSPCON reg)
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        WCOL = 0; // clear data collision flag
        SSPOV = 0; // clear overflow flag
        CKP = 1; // release clock (SSPCON reg)
    }
    /* check operation if “write” or "read" */
    if(!SSPSTATbits.D_nA && !SSPSTATbits.R_nW) // write to slave
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        while(!BF); // wait until receive is complete (SSPSTAT reg)
        /* read data from SSPBUF */
        /* data = SSPBUF; */
        CKP = 1; // release clock (SSPCON reg)
    }
    else if(!SSPSTATbits.D_nA && SSPSTATbits.R_nW) // read from slave
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        BF = 0; // clear buffer status bit (SSPSTAT reg)
        /* send data by writing to SSPBUF */
        /* SSPBUF = data; */
        CKP = 1; // release clock (SSPCON reg)
        while(BF); // wait until transmit is complete (SSPSTAT reg)
    }
    SSPIF = 0; // clear interrupt flag
}
```

### Send Receive Example Slave Mode

```c
void interrupt ISR(void)
{
    unsigned char temp;
    CKP = 0; // hold clock low (SSPCON reg)
    if(WCOL || SSPOV) // check if overflow or data collision (SSPCON reg)
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        WCOL = 0; // clear data collision flag
        SSPOV = 0; // clear overflow flag
        CKP = 1; // release clock (SSPCON reg)
    }
    /* check operation if “write” or "read"*/
    if(!SSPSTATbits.D_nA && !SSPSTATbits.R_nW) // write to slave
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        while(!BF); // wait until receive is complete (SSPSTAT reg)
        PORTB = SSPBUF; // write data from master to PORTB
        CKP = 1; // release clock (SSPCON reg)
    }
    else if(!SSPSTATbits.D_nA && SSPSTATbits.R_nW) // read from slave
    {
        temp = SSPBUF; // read SSPBUF to clear buffer
        BF = 0; // clear buffer status bit (SSPSTAT reg)
        SSPBUF = PORTD; // send data from PORTD to master
        CKP = 1; // release clock (SSPCON reg)
        while(BF); // wait until transmit is complete (SSPSTAT reg)
    }
    SSPIF = 0; // clear interrupt flag
}

void main(void)
{
    TRISB = 0x00; // set all bits in PORTB to output
    PORTB = 0x00; // all LEDs in PORTB are off
    TRISD = 0xFF; // set all bits in PORTD to input
    init_I2C_Slave(0x10); // initialize I2C as slave with address 0x10
    for(;;)
    {
    }
}

```
