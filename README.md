/*
 * LTC6811 Battery Monitor Interface with PIC16F1788
 * XC8 Compiler - Register Level Implementation
 * 
 * Pin Configuration:
 * RC3 (SCK/SCL)  - SPI Clock
 * RC4 (SDI/SDA)  - SPI Data In (MISO)
 * RC5 (SDO)      - SPI Data Out (MOSI)
 * RC2            - Chip Select (CS) - Active Low
 * RA0            - Status LED
 * 
 * LTC6811 SPI Mode: CPOL=1, CPHA=1 (SPI Mode 3)
 */

// PIC16F1788 Configuration Bits
#pragma config FOSC = INTOSC    // Internal oscillator
#pragma config WDTE = OFF       // Watchdog Timer disabled
#pragma config PWRTE = OFF      // Power-up Timer disabled
#pragma config MCLRE = ON       // MCLR pin enabled
#pragma config CP = OFF         // Code protection off
#pragma config CPD = OFF        // Data code protection off
#pragma config BOREN = ON       // Brown-out Reset enabled
#pragma config CLKOUTEN = OFF   // CLKOUT disabled
#pragma config IESO = OFF       // Internal/External Switchover disabled
#pragma config FCMEN = ON       // Fail-Safe Clock Monitor enabled
#pragma config WRT = OFF        // Flash Memory Write protection off
#pragma config PLLEN = OFF      // PLL disabled
#pragma config STVREN = ON      // Stack Overflow/Underflow Reset enabled
#pragma config BORV = LO        // Brown-out Reset Voltage low trip point
#pragma config LVP = ON         // Low-voltage programming enabled

#include <xc.h>
#include <stdint.h>

// System definitions
#define _XTAL_FREQ 16000000     // 16 MHz internal oscillator

// Pin definitions
#define CS_PIN      LATCbits.LATC2
#define CS_TRIS     TRISCbits.TRISC2
#define LED_PIN     LATAbits.LATA0
#define LED_TRIS    TRISAbits.TRISA0

// CS control macros
#define CS_LOW()    (CS_PIN = 0)
#define CS_HIGH()   (CS_PIN = 1)
#define LED_ON()    (LED_PIN = 1)
#define LED_OFF()   (LED_PIN = 0)

// LTC6811 Command codes
#define CMD_WRCFG   0x0001      // Write Configuration Register Group
#define CMD_RDCFG   0x0002      // Read Configuration Register Group
#define CMD_RDCVA   0x0004      // Read Cell Voltage Register Group A
#define CMD_RDCVB   0x0006      // Read Cell Voltage Register Group B
#define CMD_RDCVC   0x0008      // Read Cell Voltage Register Group C
#define CMD_RDCVD   0x000A      // Read Cell Voltage Register Group D
#define CMD_ADCV    0x0260      // Start Cell Voltage ADC Conversion

// PEC15 polynomial: x^15 + x^14 + x^10 + x^8 + x^7 + x^4 + x^3 + 1
#define PEC15_POLY  0x4599

// Function prototypes
void system_init(void);
void oscillator_init(void);
void gpio_init(void);
void spi_init(void);
uint8_t spi_transfer(uint8_t data);
void ltc6811_wakeup(void);
uint16_t pec15_calc(uint8_t *data, uint8_t len);
void ltc6811_send_command(uint16_t cmd);
uint8_t ltc6811_read_config(uint8_t *config_data);
void delay_us(uint16_t microseconds);

/*
 * PEC15 CRC Calculation
 * Implements the CRC-15 algorithm used by LTC6811
 */
uint16_t pec15_calc(uint8_t *data, uint8_t len)
{
    uint16_t remainder = 16;  // PEC seed value
    uint8_t i, j;
    
    for (i = 0; i < len; i++)
    {
        remainder ^= ((uint16_t)data[i] << 7);
        
        for (j = 0; j < 8; j++)
        {
            if (remainder & 0x4000)
            {
                remainder = ((remainder << 1) ^ PEC15_POLY);
            }
            else
            {
                remainder = (remainder << 1);
            }
        }
    }
    
    return (remainder << 1);  // Multiply by 2 for final PEC
}

/*
 * Initialize system oscillator to 16 MHz
 */
void oscillator_init(void)
{
    // Configure internal oscillator for 16 MHz
    OSCCONbits.IRCF = 0b1111;   // 16 MHz HF internal oscillator
    OSCCONbits.SCS = 0b10;      // Internal oscillator block
    
    // Wait for oscillator to stabilize
    while (!OSCCONbits.HFIOFS);
}

/*
 * Initialize GPIO pins
 */
void gpio_init(void)
{
    // Disable analog functions on all pins
    ANSELA = 0x00;
    ANSELB = 0x00;
    ANSELC = 0x00;
    
    // Configure LED pin (RA0) as output
    LED_TRIS = 0;
    LED_OFF();
    
    // Configure CS pin (RC2) as output, initially high
    CS_TRIS = 0;
    CS_HIGH();
    
    // Configure SPI pins
    // RC3 (SCK) - output (configured by SPI module)
    // RC4 (SDI) - input
    // RC5 (SDO) - output (configured by SPI module)
    TRISCbits.TRISC3 = 0;   // SCK output
    TRISCbits.TRISC4 = 1;   // SDI input
    TRISCbits.TRISC5 = 0;   // SDO output
}

/*
 * Initialize MSSP module for SPI Mode 3 (CPOL=1, CPHA=1)
 * Required for LTC6811 communication
 */
void spi_init(void)
{
    // Disable MSSP module during configuration
    SSPCONbits.SSPEN = 0;
    
    // Configure SPI mode
    SSPCONbits.SSPM = 0b0001;   // SPI Master mode, clock = Fosc/16 (1 MHz)
    SSPCONbits.CKP = 1;         // Idle state for clock is high level (CPOL=1)
    SSPSTATbits.CKE = 0;        // Transmit occurs on transition from idle to active (CPHA=1)
    SSPSTATbits.SMP = 0;        // Input data sampled at middle of data output time
    
    // Enable MSSP module
    SSPCONbits.SSPEN = 1;
}

/*
 * SPI transfer function - transmit and receive one byte
 */
uint8_t spi_transfer(uint8_t data)
{
    // Clear buffer full flag
    SSPSTATbits.BF = 0;
    
    // Write data to buffer to initiate transmission
    SSPBUF = data;
    
    // Wait for transmission to complete
    while (!SSPSTATbits.BF);
    
    // Return received data
    return SSPBUF;
}

/*
 * Microsecond delay function
 */
void delay_us(uint16_t microseconds)
{
    while (microseconds--)
    {
        __delay_us(1);
    }
}

/*
 * LTC6811 Wake-up pulse
 * The LTC6811 goes to sleep after ~2 seconds of inactivity
 * Wake-up requires pulling CS low for >400us
 */
void ltc6811_wakeup(void)
{
    CS_LOW();
    delay_us(500);      // 500 us pulse (>400us required)
    CS_HIGH();
    delay_us(10);       // Short delay for LTC6811 to wake up
}

/*
 * Send command to LTC6811 with PEC
 */
void ltc6811_send_command(uint16_t cmd)
{
    uint8_t cmd_bytes[2];
    uint16_t pec;
    uint8_t pec_bytes[2];
    
    // Convert command to bytes (MSB first)
    cmd_bytes[0] = (uint8_t)(cmd >> 8);
    cmd_bytes[1] = (uint8_t)(cmd & 0xFF);
    
    // Calculate PEC for command
    pec = pec15_calc(cmd_bytes, 2);
    pec_bytes[0] = (uint8_t)(pec >> 8);
    pec_bytes[1] = (uint8_t)(pec & 0xFF);
    
    // Wake up LTC6811
    ltc6811_wakeup();
    
    // Send command with PEC
    CS_LOW();
    spi_transfer(cmd_bytes[0]);
    spi_transfer(cmd_bytes[1]);
    spi_transfer(pec_bytes[0]);
    spi_transfer(pec_bytes[1]);
    CS_HIGH();
}

/*
 * Read Configuration Register Group from LTC6811
 * Returns 1 if successful (PEC valid), 0 if PEC error
 */
uint8_t ltc6811_read_config(uint8_t *config_data)
{
    uint8_t cmd_bytes[2];
    uint16_t cmd_pec, data_pec;
    uint8_t pec_bytes[2];
    uint8_t rx_data[8];     // 6 bytes config + 2 bytes PEC
    uint8_t i;
    
    // RDCFG command
    cmd_bytes[0] = (uint8_t)(CMD_RDCFG >> 8);
    cmd_bytes[1] = (uint8_t)(CMD_RDCFG & 0xFF);
    
    // Calculate command PEC
    cmd_pec = pec15_calc(cmd_bytes, 2);
    pec_bytes[0] = (uint8_t)(cmd_pec >> 8);
    pec_bytes[1] = (uint8_t)(cmd_pec & 0xFF);
    
    // Wake up LTC6811
    ltc6811_wakeup();
    
    // Send command
    CS_LOW();
    spi_transfer(cmd_bytes[0]);
    spi_transfer(cmd_bytes[1]);
    spi_transfer(pec_bytes[0]);
    spi_transfer(pec_bytes[1]);
    
    // Read configuration data and PEC
    for (i = 0; i < 8; i++)
    {
        rx_data[i] = spi_transfer(0xFF);  // Send dummy bytes to clock in data
    }
    CS_HIGH();
    
    // Copy configuration data to output buffer
    for (i = 0; i < 6; i++)
    {
        config_data[i] = rx_data[i];
    }
    
    // Calculate PEC on received data
    data_pec = pec15_calc(rx_data, 6);
    
    // Compare calculated PEC with received PEC
    if ((uint8_t)(data_pec >> 8) == rx_data[6] && 
        (uint8_t)(data_pec & 0xFF) == rx_data[7])
    {
        return 1;  // PEC valid
    }
    else
    {
        return 0;  // PEC error
    }
}

/*
 * System initialization
 */
void system_init(void)
{
    oscillator_init();
    gpio_init();
    spi_init();
}

/*
 * Main function
 */
void main(void)
{
    uint8_t config_data[6];
    uint8_t comm_status;
    uint16_t poll_count = 0;
    
    // Initialize system
    system_init();
    
    // Startup indication - blink LED 3 times
    for (uint8_t i = 0; i < 3; i++)
    {
        LED_ON();
        __delay_ms(100);
        LED_OFF();
        __delay_ms(100);
    }
    
    // Main loop - poll LTC6811 and indicate status
    while (1)
    {
        // Wake up and read configuration register
        comm_status = ltc6811_read_config(config_data);
        
        // Update LED based on communication status
        if (comm_status)
        {
            // Communication successful - LED ON
            LED_ON();
        }
        else
        {
            // Communication error - blink LED rapidly
            LED_ON();
            __delay_ms(50);
            LED_OFF();
            __delay_ms(50);
        }
        
        // Delay between polls
        __delay_ms(500);
        
        // Optional: Toggle LED every 10 polls to show activity
        poll_count++;
        if (poll_count >= 10)
        {
            poll_count = 0;
            LED_OFF();
            __delay_ms(100);
        }
    }
}
