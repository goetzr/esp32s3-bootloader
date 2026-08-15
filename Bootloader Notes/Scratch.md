- Newlib subset of of standard C library functions
- printf UART channel (components/esp_rom/esp32s3/include/esp32s3/rom/uart.h):
	```
	/**
	  * @brief Output a char to printf channel, wait until fifo not full.
	  *
	  * @param  None
	  *
	  * @return OK.
	  */
	ETS_STATUS uart_tx_one_char(uint8_t TxChar);
	```
	