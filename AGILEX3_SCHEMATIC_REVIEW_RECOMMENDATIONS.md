# Agilex 3 Schematic Review Recommendations

Scope: reviewed `ag3_schematic_review.xlsm` Checklist rows (including all row context columns such as guideline/additional info/power group/decap), `A3CW135B.xlsx` sheet `Pin List M16A`, `846812_846813.pdf`, `hand held radio_03_07_2026.pdf`, `top.qsf`, and `top.v`. Also spot-checked current public Altera Agilex 3 pin-connection guidance dated 2026-05-13.

I did **not** edit `ag3_schematic_review.xlsm`; per request, the proposed Column C/D entries are in:

- `agilex3_checklist_column_cd_recommendations.csv`

## Highest-priority updates before customer release

1. **Device OPN mismatch (Checklist row 16):** QSF/DevSel use `A3CW135BM16AI6S`, but the schematic FPGA symbol text repeatedly shows `A3CY135BM16AE6S` on U9 pages. A3CY is listed as the **without-HPS** family in the worksheet OPN table, while this design uses HPS/EMIF. Update the schematic symbol/BOM/title-block device text to `A3CW135BM16AI6S` or realign every file if A3CY was actually intended.
2. **GTS vs HSIO confusion (rows 22, 158-171):** The RF ADRV9002 interface is on HSIO/LVDS bank 2A, not Agilex GTS. Set `Use Transceiver?` to **No** for the GTS section, but still fix unused GTS input/reference pins: unused GTS REFCLK/RX pins must be tied to GND; CDRCLKOUT/TX/APROBE may float as noted in the CSV.
3. **Missing Quartus electrical assignments:** `top.qsf` contains pin locations but no `IO_STANDARD`, termination, slew/current, or differential-standard assignments. Add these for bank 2A LVDS/True Differential 1.3 V, bank 3A LPDDR4/HPS EMIF, and bank 5A/5B HVIO peripherals before relying on the pinout.
4. **SDM optional signal assignment rows are incomplete:** Rows 72-79 are blank while the schematic routes CONF_DONE/INIT_DONE/HPS_COLD_RESET/PWRMGT nets. Fill the assignment rows so rows 94/104/105/108/110 evaluate against the real signals.
5. **HPS section flags over-enabled:** The checklist marks HPS JTAG/NAND/USB/I2C/I3C/SPI/UART/Trace as `Yes` even though their detailed rows are not used or the implementation is fabric I/O, not HPS dedicated I/O. Set those section flags to `No` unless the HPS pinmux confirms otherwise.
6. **Potential HPS pinmux overlap to confirm:** If SDMMC uses HPS_IOA_7/A8 for DATA3/CMD, then EMAC2 MDIO/MDC should use the alternate HPS_IOB_9/B10 group; otherwise SDMMC and MDIO can overlap.

## Validation notes

- QSF package balls: all 153 `set_location_assignment` pins exist in `Pin List M16A`; no invalid balls found.
- DDR/HPS LPDDR4 pins: QSF places the LPDDR4/HPS EMIF pins in 3A_T/3A_B M16A HPS_DDR-capable pins.
- RREF_SDM: schematic shows 2 kohm +/-1% to GND, matching the guideline.
- RZQ: schematic extraction shows 240 ohm +/-1% RZQ resistors; keep, but describe as 240 ohm to GND, not direct ground.
- Power rails: the stated 0.78 V/1.0 V/1.1 V/1.2 V/1.3 V/1.8 V/3.3 V rails generally align with the selected -6S device and intended interfaces, subject to the OPN mismatch and QSF electrical-assignment updates above.

## Row recommendation counts

- ERROR: 1
- INFO: 62
- NOT USED: 28
- OK: 246
- UPDATE: 9
- WARNING: 32
