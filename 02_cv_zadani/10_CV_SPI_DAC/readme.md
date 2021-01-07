# REV - Desáté cvičení
- SPI a DAC

## Watchdog timer:
Jedná se o specifickou periferii MCU, která slouží pro bezpečnost aplikací. Jde o čítač, který je napojen na vnitřní oscilátor s frekvencí 31,25kHz. Je mu předřazena ještě dělička /128. Výsledná perioda je tedy 4ms (přesněji 4,096). Uživatel zapíná WDT a jeho výstupní děličku pomocí konfiguračních bitů. WDT se pak musí nulovat v softwaru speciální instrukcí procesoru. Pokud dojde k přetečení, dojde k resetu MCU.

<p align="center">
  <img width="850" height="560" src="https://github.com/MBrablc/BUT-FME-REV/blob/master/02_cv_zadani/10_CV_SPI_DAC/DAC.png">
</p>

LFINTOSC = 31,25 kHz

### Konfigurace:

    - #pragma config WDTEN = ON
    - #pragma config WDTPS = 256 (hodnota je dělička jako u timeru)

### smazani WDT:

    - PIC18 má instrukci 'CLRWDT', instrukce lze do C programu zadat příkazem __asm("CLRWDT");


## Přiklad 9.1:
Program demostruje využití WDT. Funkce trap() schválně obsahuje nekonečnou smyčku, kde přeteče WDT a resetuje MCU. BIT RCONbits.TO pak mohu používat k detekci, že nastal reset. V příkladu nám to oznámí zablikáním poslední LED.
```c
// DAC
#pragma config FOSC =   HSMP        // Oscillator Selection bits (HS oscillator (medium power 4-16 MHz))
#pragma config PLLCFG = ON         // 4X PLL Enable (Oscillator multiplied by 4)
#pragma config WDTEN = OFF           // Watchdog Timer Enable bits (WDT is always enabled. SWDTEN bit has no effect)

#include <xc.h>

#define _XTAL_FREQ 32E6         // definice fosc pro knihovnu
#define DAC_SS LATB3            // DAC slave select pin
#define DAC_CH1 0b1001          // kanal A
#define DAC_CH2 0b0001          // kanal B
#define LED LATDbits.LATD2      // ledka

int data0;
int data1=1;

void init(void){
    
    /* vyber pinu jako vystupy */
    TRISCbits.TRISC3 = 0;
    TRISCbits.TRISC5 = 0;
    TRISBbits.TRISB3 = 0;
    TRISDbits.TRISD2 = 0;
 
    LATBbits.LATB3 = 1;         // DAC SS off
    
    SSP1CON1bits.SSPM = 0b0010; // SPI clock
    SSP1CON1bits.SSPEN = 1;     // SPI zapnuto
}
void SPIWrite(char chanel ,char data);

void main(void) {
    init(); // provedeni inicializace
    LED = 0;
    unsigned int i = 0;
    unsigned int j = 0;
    
    // <editor-fold defaultstate="collapsed" desc="SINTABLE">
    const unsigned char tabulka[128] = 
    {
        0x80,0x86,0x8c,0x92,0x98,0x9e,0xa5,0xaa,0xb0,0xb6,0xbc,0xc1,0xc6,0xcb,0xd0,0xd5,
        0xda,0xde,0xe2,0xe6,0xea,0xed,0xf0,0xf3,0xf5,0xf8,0xfa,0xfb,0xfd,0xfe,0xfe,0xff,
        0xff,0xff,0xfe,0xfe,0xfd,0xfb,0xfa,0xf8,0xf5,0xf3,0xf0,0xed,0xea,0xe6,0xe2,0xde,
        0xda,0xd5,0xd0,0xcb,0xc6,0xc1,0xbc,0xb6,0xb0,0xaa,0xa5,0x9e,0x98,0x92,0x8c,0x86,
        0x80,0x79,0x73,0x6d,0x67,0x61,0x5a,0x55,0x4f,0x49,0x43,0x3e,0x39,0x34,0x2f,0x2a,
        0x25,0x21,0x1d,0x19,0x15,0x12,0xf,0xc,0xa,0x7,0x5,0x4,0x2,0x1,0x1,0x0,
        0x0,0x0,0x1,0x1,0x2,0x4,0x5,0x7,0xa,0xc,0xf,0x12,0x15,0x19,0x1d,0x21,
        0x25,0x2a,0x2f,0x34,0x39,0x3e,0x43,0x49,0x4f,0x55,0x5a,0x61,0x67,0x6d,0x73,0x79,
    };
    // </editor-fold>
    
    /* hlavni smycka */
    while(1){

            SPIWrite(DAC_CH1,i++);  
            SPIWrite(DAC_CH2,tabulka[j++]);
            if(j == 128) j=0;
        }
}
/* funkce zapisu SPI funkce zapisuje dva bajty za sebou */
void SPIWrite(char channel, char data){
    
    unsigned char msb, lsb, flush;
    msb = (channel<<4) | (data>>4);     // prvni bajt
    lsb = (data<<4);                    // druhy bajt
    DAC_SS = 0;                         // slave select
    PIR1bits.SSPIF = 0;                 // vynulovani priznaku SPI
    SSPBUF = msb;                       // zapis do bufferu
    while(PIR1bits.SSPIF == 0)NOP();    // pockat nez SPI posle prvni bajt
    
    PIR1bits.SSPIF = 0;                 // vynulovani priznaku SPI
    SSPBUF = lsb;                       // zapis do bufferu
    while(PIR1bits.SSPIF == 0)NOP();    // pockat nez SPI posle druhy bajt
    
    DAC_SS = 1;                         // vypnout slave select
    flush = SSPBUF;                     // vycteni bufferu
    
}
```

## Řízení spotřeby:
PIC18 ma v podstatě dva power módy. Jedná se o IDLE a SLEEP. Rozdíl je ten, že SLEEP opravdu zastaví CPU i periferie, zatímco IDLE jen CPU. V IDLE modu mohu tedy probudit procesor například přerušením od periferie (ukážeme si TIMER). Ve SLEEP modu k tomu může sloužit třeba WDT, který tentokrát neresetuje, ale probudí zařízení. 

## Power mody:
<p align="center">
  <img width="700" height="260" src="https://github.com/MBrablc/BUT-FME-REV/blob/master/02_cv_zadani/10_CV_SPI_DAC/Dac.png">
</p>

## Přiklad 9.2:
IDLEN = 0 je to deep sleep. CPU i periferie neběží. K probuzení použijeme WDT. Který po provedení "SLEEP" instrukce procesor probudí.
```c
// DAC
#pragma config FOSC =   HSMP        // Oscillator Selection bits (HS oscillator (medium power 4-16 MHz))
#pragma config PLLCFG = ON         // 4X PLL Enable (Oscillator multiplied by 4)
#pragma config WDTEN = OFF           // Watchdog Timer Enable bits (WDT is always enabled. SWDTEN bit has no effect)

#include <xc.h>

#define _XTAL_FREQ 32E6         // definice fosc pro knihovnu
#define DAC_SS LATB3            // DAC slave select pin
#define DAC_CH1 0b1001          // kanal A
#define DAC_CH2 0b0001          // kanal B
#define LED LATDbits.LATD2      // ledka

int data0;
int data1=1;

void init(void){
    
    /* vyber pinu jako vystupy */
    TRISCbits.TRISC3 = 0;
    TRISCbits.TRISC5 = 0;
    TRISBbits.TRISB3 = 0;
    TRISDbits.TRISD2 = 0;
 
    LATBbits.LATB3 = 1;         // DAC SS off
    
    SSP1CON1bits.SSPM = 0b0010; // SPI clock
    SSP1CON1bits.SSPEN = 1;     // SPI zapnuto
}
void SPIWrite(char chanel ,char data);

void main(void) {
    init(); // provedeni inicializace
    LED = 0;
    unsigned int i = 0;
    unsigned int j = 0;
    
    // <editor-fold defaultstate="collapsed" desc="SINTABLE">
    const unsigned char tabulka[128] = 
    {
        0x80,0x86,0x8c,0x92,0x98,0x9e,0xa5,0xaa,0xb0,0xb6,0xbc,0xc1,0xc6,0xcb,0xd0,0xd5,
        0xda,0xde,0xe2,0xe6,0xea,0xed,0xf0,0xf3,0xf5,0xf8,0xfa,0xfb,0xfd,0xfe,0xfe,0xff,
        0xff,0xff,0xfe,0xfe,0xfd,0xfb,0xfa,0xf8,0xf5,0xf3,0xf0,0xed,0xea,0xe6,0xe2,0xde,
        0xda,0xd5,0xd0,0xcb,0xc6,0xc1,0xbc,0xb6,0xb0,0xaa,0xa5,0x9e,0x98,0x92,0x8c,0x86,
        0x80,0x79,0x73,0x6d,0x67,0x61,0x5a,0x55,0x4f,0x49,0x43,0x3e,0x39,0x34,0x2f,0x2a,
        0x25,0x21,0x1d,0x19,0x15,0x12,0xf,0xc,0xa,0x7,0x5,0x4,0x2,0x1,0x1,0x0,
        0x0,0x0,0x1,0x1,0x2,0x4,0x5,0x7,0xa,0xc,0xf,0x12,0x15,0x19,0x1d,0x21,
        0x25,0x2a,0x2f,0x34,0x39,0x3e,0x43,0x49,0x4f,0x55,0x5a,0x61,0x67,0x6d,0x73,0x79,
    };
    // </editor-fold>
    
    /* hlavni smycka */
    while(1){

            SPIWrite(DAC_CH1,i++);  
            SPIWrite(DAC_CH2,tabulka[j++]);
            if(j == 128) j=0;
        }
}
/* funkce zapisu SPI funkce zapisuje dva bajty za sebou */
void SPIWrite(char channel, char data){
    
    unsigned char msb, lsb, flush;
    msb = (channel<<4) | (data>>4);     // prvni bajt
    lsb = (data<<4);                    // druhy bajt
    DAC_SS = 0;                         // slave select
    PIR1bits.SSPIF = 0;                 // vynulovani priznaku SPI
    SSPBUF = msb;                       // zapis do bufferu
    while(PIR1bits.SSPIF == 0)NOP();    // pockat nez SPI posle prvni bajt
    
    PIR1bits.SSPIF = 0;                 // vynulovani priznaku SPI
    SSPBUF = lsb;                       // zapis do bufferu
    while(PIR1bits.SSPIF == 0)NOP();    // pockat nez SPI posle druhy bajt
    
    DAC_SS = 1;                         // vypnout slave select
    flush = SSPBUF;                     // vycteni bufferu
    
}
```
## Přiklad 9.3:
IDLEN = 1 je to idle mod. CPU je vypnuto, ale periferie běží dál. Je možné probudit interrupt flagem. V tomto případě používáme TMR1.
```c
// REV TIMER
#pragma config FOSC = HSMP      // Oscillator Selection bits (HS oscillator (medium power 4-16 MHz))
#pragma config PLLCFG = OFF     // 4X PLL Enable (Oscillator OFF)
#pragma config PRICLKEN = ON    // Primary clock enable bit (Primary clock is always enabled)
#pragma config WDTEN = OFF      // Watchdog Timer Enable bits (Watch dog timer is always disabled. SWDTEN has no effect.)

#include <xc.h>

#define _XTAL_FREQ 8E6                  // definice fosc pro knihovnu
#define LED LATDbits.LATD2              // ledka                  

void init(void){
    
    //GPIO
    TRISD = 0b10000011;     // LEDs: 2..6 out
    TRISC = 0b11101111;     // RC0 BTN1, RC4 LED
    ANSELC = 0;
    ANSELD = 0;
    
    // Zhasnu ledky
    LATD2 = 1;
    LATD3 = 1;
    LATC4 = 1;
    LATD4 = 1;
    LATD5 = 1;
    LATD6 = 1;
    
    //TIMER 1   
    T1CONbits.TMR1CS = 0b00;        // zdroj casovace 1
    T1CONbits.T1CKPS = 0b11;        // nastaveni delicky
    IDLEN = 1;                      // musim nechat clock pro periferie
    PEIE = 1;                       // irq od periferii
    TMR1IE = 1;                     // irq od TIMER1
    TMR1ON = 1;                     // Zapnu TIMER1

}

void main(void) {
    init();                         // provedeni inicializace
    
    while(1){
        LED ^= 1;
        TMR1 = 0xFFFF - 50000;               // nastavim citac
        TMR1IF = 0;
        __asm("SLEEP");
        }
}
```

### Zadání:

  1) Nakonfigurujte deep sleep a probouzejte MCU cca jednou za 1s. Po probuzení odešlete hodnotu POT2 přes UART do PC. 

  2) Nakonfugurujte komunikaci s PC. Pokud nepříjde déle než 30s specifická zpráva (kterou si zvolte), tak dojde k SW resetu. Po resetu zobrazte na displeji, že zpráva nedošla.
  3) Provozujte kontroler v IDLE módu. Kontrolér hned po inicilizaci uspěte a komunikujte přes uart. Tedy příchod zprávy MCU probudí. Zpráva se odešle zpět (ECHO) a MCU zase usne.