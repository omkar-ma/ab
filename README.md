/*
 * LTC6811-2 Cell Voltage Monitoring
 * MCU     : PIC16F183xx
 * Fosc    : 16 MHz
 * SPI Clk : 250 kHz (FOSC/64)
 * SPI Mode: Mode 3 (CKP=1, CKE=0)
 */

#include <xc.h>
#include <stdint.h>
#include <stdio.h>

// ─────────────────────────────────────────
//  PIN DEFINITIONS  (change to your pins)
// ─────────────────────────────────────────
#define CS_LOW()   LATCbits.LATC2 = 0
#define CS_HIGH()  LATCbits.LATC2 = 1

// ─────────────────────────────────────────
//  PEC15 LOOKUP TABLE
// ─────────────────────────────────────────
const uint16_t pec15Table[256] = {
    0x0000, 0xC599, 0xCEAB, 0x0B32, 0xD8CF, 0x1D56, 0x1664, 0xD3FD,
    0xF407, 0x319E, 0x3AAC, 0xFF35, 0x2CC8, 0xE951, 0xE263, 0x27FA,
    0xAD97, 0x680E, 0x633C, 0xA6A5, 0x7558, 0xB0C1, 0xBBF3, 0x7E6A,
    0x5990, 0x9C09, 0x973B, 0x52A2, 0x815F, 0x44C6, 0x4FF4, 0x8A6D,
    0x5B2E, 0x9EB7, 0x9585, 0x501C, 0x83E1, 0x4678, 0x4D4A, 0x88D3,
    0xAF29, 0x6AB0, 0x6182, 0xA41B, 0x77E6, 0xB27F, 0xB94D, 0x7CD4,
    0xF6B9, 0x3320, 0x3812, 0xFD8B, 0x2E76, 0xEBEF, 0xE0DD, 0x2544,
    0x02BE, 0xC727, 0xCC15, 0x098C, 0xDA71, 0x1FE8, 0x14DA, 0xD143,
    0xF3C5, 0x365C, 0x3D6E, 0xF8F7, 0x2B0A, 0xEE93, 0xE5A1, 0x2038,
    0x07C2, 0xC25B, 0xC969, 0x0CF0, 0xDF0D, 0x1A94, 0x11A6, 0xD43F,
    0x5E52, 0x9BCB, 0x90F9, 0x5560, 0x869D, 0x4304, 0x4836, 0x8DAF,
    0xAA55, 0x6FCC, 0x64FE, 0xA167, 0x729A, 0xB703, 0xBC31, 0x79A8,
    0xA8EB, 0x6D72, 0x6640, 0xA3D9, 0x7024, 0xB5BD, 0xBE8F, 0x7B16,
    0x5CEC, 0x9975, 0x9247, 0x57DE, 0x8423, 0x41BA, 0x4A88, 0x8F11,
    0x057C, 0xC0E5, 0xCBD7, 0x0E4E, 0xDDB3, 0x182A, 0x1318, 0xD681,
    0xF17B, 0x34E2, 0x3FD0, 0xFA49, 0x29B4, 0xEC2D, 0xE71F, 0x2286,
    0xA213, 0x678A, 0x6CB8, 0xA921, 0x7ADC, 0xBF45, 0xB477, 0x71EE,
    0x5614, 0x938D, 0x98BF, 0x5D26, 0x8EDB, 0x4B42, 0x4070, 0x85E9,
    0x0F84, 0xCA1D, 0xC12F, 0x04B6, 0xD74B, 0x12D2, 0x19E0, 0xDC79,
    0xFB83, 0x3E1A, 0x3528, 0xF0B1, 0x234C, 0xE6D5, 0xEDE7, 0x287E,
    0xF93D, 0x3CA4, 0x3796, 0xF20F, 0x21F2, 0xE46B, 0xEF59, 0x2AC0,
    0x0D3A, 0xC8A3, 0xC391, 0x0608, 0xD5F5, 0x106C, 0x1B5E, 0xDEC7,
    0x54AA, 0x9133, 0x9A01, 0x5F98, 0x8C65, 0x49FC, 0x42CE, 0x8757,
    0xA0AD, 0x6534, 0x6E06, 0xAB9F, 0x7862, 0xBDFB, 0xB6C9, 0x7350,
    0x51D6, 0x944F, 0x9F7D, 0x5AE4, 0x8919, 0x4C80, 0x47B2, 0x822B,
    0xA5D1, 0x6048, 0x6B7A, 0xAEE3, 0x7D1E, 0xB887, 0xB3B5, 0x762C,
    0xFC41, 0x39D8, 0x32EA, 0xF773, 0x248E, 0xE117, 0xEA25, 0x2FBC,
    0x0846, 0xCDDF, 0xC6ED, 0x0374, 0xD089, 0x1510, 0x1E22, 0xDBBB,
    0x0AF8, 0xCF61, 0xC453, 0x01CA, 0xD237, 0x17AE, 0x1C9C, 0xD905,
    0xFEFF, 0x3B66, 0x3054, 0xF5CD, 0x2630, 0xE3A9, 0xE89B, 0x2D02,
    0xA76F, 0x62F6, 0x69C4, 0xAC5D, 0x7FA0, 0xBA39, 0xB10B, 0x7492,
    0x5368, 0x96F1, 0x9DC3, 0x585A, 0x8BA7, 0x4E3E, 0x450C, 0x8095
};

// ─────────────────────────────────────────
//  PEC15 CALCULATION
// ─────────────────────────────────────────
uint16_t PEC15_Calc(uint8_t len, uint8_t *data) {
    uint16_t remainder = 16;  // seed
    uint16_t addr;
    for (uint8_t i = 0; i < len; i++) {
        addr = ((remainder >> 7) ^ data[i]) & 0xFF;
        remainder = (remainder << 8) ^ pec15Table[addr];
    }
    return (remainder * 2);  // shift left 1
}

// ─────────────────────────────────────────
//  SPI INIT
// ─────────────────────────────────────────
void SPI_Init(void) {
    // Set pin directions
    TRISCbits.TRISC2 = 0;   // CS  → Output
    TRISCbits.TRISC3 = 0;   // SCK → Output
    TRISCbits.TRISC4 = 1;   // SDI (MISO) → Input
    TRISCbits.TRISC5 = 0;   // SDO (MOSI) → Output

    CS_HIGH();               // CS idle high

    SSP1CON1bits.SSPEN = 0;         // Disable SPI before config
    SSP1CON1bits.SSPM  = 0b0010;    // SPI Master, FOSC/64 = 250kHz @ 16MHz
    SSP1CON1bits.CKP   = 1;         // Clock idle HIGH  (Mode 3)
    SSP1STATbits.CKE   = 0;         // Data valid on 2nd edge (Mode 3)
    SSP1STATbits.SMP   = 0;         // Sample at middle
    SSP1CON1bits.SSPEN = 1;         // Enable SPI
}

// ─────────────────────────────────────────
//  SPI WRITE & READ BYTE
// ─────────────────────────────────────────
uint8_t SPI_TransferByte(uint8_t tx) {
    SSP1BUF = tx;                        // Send byte
    while (!SSP1STATbits.BF);           // Wait until transfer complete
    return SSP1BUF;                      // Return received byte
}

void SPI_WriteByte(uint8_t tx) {
    SPI_TransferByte(tx);               // Send, ignore received
}

uint8_t SPI_ReadByte(void) {
    return SPI_TransferByte(0xFF);      // Send dummy, read response
}

// ─────────────────────────────────────────
//  LTC6811-2 WAKE UP
// ─────────────────────────────────────────
void LTC6811_WakeUp(void) {
    CS_LOW();
    SPI_WriteByte(0xFF);    // dummy byte to wake IC
    CS_HIGH();
    __delay_us(400);        // tWAKE = 400us minimum
}

// ─────────────────────────────────────────
//  SEND COMMAND HELPER
// ─────────────────────────────────────────
void LTC6811_SendCmd(uint8_t cmd0, uint8_t cmd1) {
    uint8_t cmd[2] = {cmd0, cmd1};
    uint16_t pec = PEC15_Calc(2, cmd);

    CS_LOW();
    SPI_WriteByte(cmd[0]);
    SPI_WriteByte(cmd[1]);
    SPI_WriteByte((uint8_t)(pec >> 8));
    SPI_WriteByte((uint8_t)(pec));
    CS_HIGH();
}

// ─────────────────────────────────────────
//  WRCFGA - Write Configuration Register A
// ─────────────────────────────────────────
void LTC6811_WRCFGA(void) {
    uint8_t cmd[2]  = {0x00, 0x01};  // WRCFGA command

    // Configuration bytes
    uint8_t data[6];
    data[0] = 0xFE;  // CFGR0: REFON=1 (keep reference on), GPIO pulldowns off
    data[1] = 0x00;  // CFGR1: VUV[7:0] = 0
    data[2] = 0x00;  // CFGR2: VOV[3:0]=0, VUV[11:8]=0
    data[3] = 0x00;  // CFGR3: VOV[11:4] = 0
    data[4] = 0x00;  // CFGR4: DCC8-DCC1 = 0 (no cell balancing)
    data[5] = 0x00;  // CFGR5: DCTO=0, DCC12-DCC9=0

    uint16_t cmdPEC  = PEC15_Calc(2, cmd);
    uint16_t dataPEC = PEC15_Calc(6, data);

    CS_LOW();
    // Send command + PEC
    SPI_WriteByte(cmd[0]);
    SPI_WriteByte(cmd[1]);
    SPI_WriteByte((uint8_t)(cmdPEC >> 8));
    SPI_WriteByte((uint8_t)(cmdPEC));
    // Send data + PEC
    for (uint8_t i = 0; i < 6; i++) SPI_WriteByte(data[i]);
    SPI_WriteByte((uint8_t)(dataPEC >> 8));
    SPI_WriteByte((uint8_t)(dataPEC));
    CS_HIGH();
}

// ─────────────────────────────────────────
//  ADCV - Start ADC Voltage Conversion
//  MD=01 (Normal), DCP=0, CH=000 (All cells)
// ─────────────────────────────────────────
void LTC6811_ADCV(void) {
    // ADCV command: 0x0260 (Normal mode, all cells)
    LTC6811_SendCmd(0x02, 0x60);
    __delay_ms(3);   // Wait for conversion (~2.3ms for normal mode)
}

// ─────────────────────────────────────────
//  READ CELL VOLTAGE REGISTERS
//  RDCVA = cells 1,2,3  → cmd 0x0004
//  RDCVB = cells 4,5,6  → cmd 0x0006
// ─────────────────────────────────────────
void LTC6811_ReadVoltages(float *cellVoltage) {
    uint8_t rxBuf[8];   // 6 data bytes + 2 PEC bytes
    uint8_t cmd[2];
    uint16_t pec;
    uint16_t rawVal;

    // --- Read RDCVA (Cell 1, 2, 3) ---
    cmd[0] = 0x00; cmd[1] = 0x04;
    pec = PEC15_Calc(2, cmd);

    CS_LOW();
    SPI_WriteByte(cmd[0]);
    SPI_WriteByte(cmd[1]);
    SPI_WriteByte((uint8_t)(pec >> 8));
    SPI_WriteByte((uint8_t)(pec));
    for (uint8_t i = 0; i < 8; i++) rxBuf[i] = SPI_ReadByte();
    CS_HIGH();

    // Parse Cell 1, 2, 3 (each 2 bytes, LSB first)
    rawVal = (uint16_t)(rxBuf[1] << 8) | rxBuf[0];
    cellVoltage[0] = rawVal * 0.0001f;   // 100uV per LSB

    rawVal = (uint16_t)(rxBuf[3] << 8) | rxBuf[2];
    cellVoltage[1] = rawVal * 0.0001f;

    rawVal = (uint16_t)(rxBuf[5] << 8) | rxBuf[4];
    cellVoltage[2] = rawVal * 0.0001f;

    // --- Read RDCVB (Cell 4, 5, 6) ---
    cmd[0] = 0x00; cmd[1] = 0x06;
    pec = PEC15_Calc(2, cmd);

    CS_LOW();
    SPI_WriteByte(cmd[0]);
    SPI_WriteByte(cmd[1]);
    SPI_WriteByte((uint8_t)(pec >> 8));
    SPI_WriteByte((uint8_t)(pec));
    for (uint8_t i = 0; i < 8; i++) rxBuf[i] = SPI_ReadByte();
    CS_HIGH();

    // Parse Cell 4, 5, 6
    rawVal = (uint16_t)(rxBuf[1] << 8) | rxBuf[0];
    cellVoltage[3] = rawVal * 0.0001f;

    rawVal = (uint16_t)(rxBuf[3] << 8) | rxBuf[2];
    cellVoltage[4] = rawVal * 0.0001f;

    rawVal = (uint16_t)(rxBuf[5] << 8) | rxBuf[4];
    cellVoltage[5] = rawVal * 0.0001f;
}

// ─────────────────────────────────────────
//  MAIN
// ─────────────────────────────────────────
void main(void) {

    // Oscillator setup (16MHz internal - adjust for your config bits)
    OSCCONbits.IRCF = 0b1111;   // 16MHz HFINTOSC
    OSCCONbits.SCS  = 0b10;     // Internal oscillator

    float cellVoltage[6];       // Stores voltage for cells 1-6

    SPI_Init();                 // Init SPI

    while (1) {

        LTC6811_WakeUp();       // 1. Wake up LTC6811-2
        __delay_us(10);

        LTC6811_WRCFGA();       // 2. Write configuration
        __delay_us(10);

        LTC6811_ADCV();         // 3. Start ADC conversion (includes 3ms wait)

        LTC6811_WakeUp();       // 4. Wake up again before reading
        __delay_us(10);

        LTC6811_ReadVoltages(cellVoltage);  // 5. Read all cell voltages

        // cellVoltage[0] = Cell 1 voltage in Volts
        // cellVoltage[1] = Cell 2 voltage in Volts
        // cellVoltage[2] = Cell 3 voltage in Volts
        // cellVoltage[3] = Cell 4 voltage in Volts
        // cellVoltage[4] = Cell 5 voltage in Volts
        // cellVoltage[5] = Cell 6 voltage in Volts

        // Add your UART print or LCD display code here
        // Example: printf("C1: %.4fV\n", cellVoltage[0]);

        __delay_ms(500);        // Read every 500ms
    }
}
