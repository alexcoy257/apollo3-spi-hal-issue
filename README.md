# apollo3-spi-hal-issue
Documenting a workaround in the new SparkFun Apollo3 core that requires a rebuild of mbed

## Main Issue
In ```am_hal_iom.c```, the function ```am_hal_spi_blocking_fullduplex``` sets the ```MSPICFG```
fullduplex bit. This appears to break half duplex transfers. The IO Modules (IOMs) each have a
RX and TX buffer and some mechanism to tell whether the CPU is reading from  or writing
to those buffers. If the fullduplex bit is set, but the CPU is set only to write to the TX
buffer (in a half duplex transfer), the module gets stuck after the RX buffer fills without
any CPU reads. I found that a simple fix is simply to ensure that the half duplex transfer
unsets the full duplex bit. In ```am_hal_spi_blocking_transfer```, place the line
```
IOMn(ui32Module)->MSPICFG &= ~(_VAL2FLD(IOM0_MSPICFG_FULLDUP, 1));
```
before the transfer starts. I placed it after the line
```
IOMn(ui32Module)->OFFSETHI = (uint16_t)(ui32Offset >> 8);
```

## Rebuilding libmbedos is required
Unfortunately, you must rebuild libmbedos and place it into the variants/<actual variant>/mbed
folder. It is also useful to copy over mbed-config.h so that the compiler knows if anything had
changed. I simply used the copy of mbed OS that was bundled with the board support package.
(submodule mbed-os-ambiq-apollo3).
  
### Stumbling blocks for rebuilding libembedos
* Depending on your application, you may want to increase the main thread stack size from 4096
  to something larger.
* You generally should use the same compiler version that Arduino uses. (in fact, you can, and
  probably should, use
  that same compiler!) I found this out when successfully building mbed-os with GCC 7.3.1 when
  Arduino used GCC 8.2.1. One symptom is that the board immediately hardfaults. (you may see a
  builtin LED blink SOS, but you might not have Serial debugging) At the time, I followed these
  instructions. [https://os.mbed.com/docs/mbed-os/v6.15/build-tools/install-and-set-up.html](https://os.mbed.com/docs/mbed-os/v6.15/build-tools/install-and-set-up.html)

## TransferIn and TransferOut functions
You can modify the Arduino SPI library to provide TransferIn and TransferOut functions that
don't clobber buffers inadvertently. Take care
that you don't transfer more than 4095 bytes in one round because that is all that the IOM can 
handle.
