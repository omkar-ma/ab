uint8_t test_cmd[2] = {0x00, 0x04};
uint16_t test_pec = pec15_calc(2, test_cmd);

printf("PEC = %04X\r\n", test_pec);



PEC = 07C2
