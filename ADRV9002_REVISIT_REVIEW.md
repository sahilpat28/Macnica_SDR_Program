# ADRV9002 Revisit Review Notes

Scope: revisited `hand held radio_03_07_2026.pdf`, `top.qsf`, `top.v`, and the newly added `adrv9002.pdf` data sheet.

## Highest priority findings

1. **ADRV9002 TX data-clock direction mismatch in `top.v`**
   - ADRV9002 data sheet: `TX1_DCLK_IN+/-` and `TX2_DCLK_IN+/-` are **inputs** to ADRV9002; `DGPIO_12/13/14/15` can alternatively be `TX1_DCLK_OUT-/+` and `TX2_DCLK_OUT-/+`, which are ADRV9002 clock outputs.
   - Schematic page 20 shows these exact ADRV9002 pins (`TX1_DCLK_IN+/-`, `TX2_DCLK_IN+/-`, `DGPIO_12/13/14/15 TXx_DCLK_OUT`).
   - `top.v` currently declares `adrv9001_tx1_dclk_in_*` and `adrv9001_tx2_dclk_in_*` as FPGA inputs, and `adrv9001_tx1_dclk_out_*` / `adrv9001_tx2_dclk_out_*` as FPGA outputs.
   - **Action:** Reverse these FPGA directions: FPGA should drive ADRV `TXx_DCLK_IN+/-`; FPGA should receive ADRV `TXx_DCLK_OUT+/-` if those clock outputs are used.

2. **ADRV9002 `GP_INT` is an output, but `top.v` drives it as an FPGA output**
   - ADRV9002 data sheet: `GP_INT` is an output from ADRV9002.
   - Schematic page 20 shows `GP_INT` on U13B K14.
   - `top.v` declares `gp_int_1` as `output wire` and drives it from `gpio_out0_pad[20]`.
   - **Action:** Change FPGA direction to input and route it into fabric/HPS interrupt logic, or leave it unconnected if unused. Do not drive this net from FPGA.

3. **FPGA `RZQ_B_2A` reference pin is assigned as a user output in QSF/RTL**
   - Schematic page 14 shows bank 2A RZQ pins with 240 ohm reference resistors, which is correct for Agilex 3 OCT reference usage.
   - QSF assigns `PIN_AD3 -to rzq`; `top.v` declares `rzq` as an FPGA output and drives it through placeholder GPIO logic.
   - **Action:** Do not use RZQ as user I/O. Remove/replace the user `rzq` port assignment and let the RZQ pin remain the OCT reference pin with the 240 ohm resistor.

4. **External/device clock direction needs correction**
   - ADRV9002 data sheet: `DEV_CLK_IN+/-` is an ADRV9002 input. In differential mode, `MODEA` must be tied to VSSA.
   - Schematic page 25 shows `ADVR9002_DEV_CLK_IN_P/N` and `FPGA_DEV_CLK_IN_P/N` from the RF clock distribution.
   - `top.v` declares `dev_clk_in_p/n` as FPGA outputs and drives them from placeholder GPIO.
   - **Action:** If the FPGA receives the same reference clock, make these FPGA pins inputs. Do not drive the RF clock source from the placeholder GPIO outputs. Also verify `MODEA` is strapped for differential `DEV_CLK_IN+/-`.

5. **Unused MCS pins are routed despite schematic note saying MCS is not used**
   - ADRV9002 data sheet: if `MCS+/-` are unused, connect them to VSSA.
   - Schematic page 20 notes MCS is not used, but also shows `F_DEV_MCS_P_IN_P/N` routed to FPGA.
   - `top.v` drives `mcs_p_in_p/n` as FPGA outputs through placeholder GPIO.
   - **Action:** If MCS is truly unused, connect ADRV9002 `MCS+/-` to VSSA and remove FPGA routing. If future MCS support is required, document it and ensure FPGA drives valid timing/levels.

6. **Control/GPIO placeholders hold ADRV9002 in invalid states**
   - `top.v` sets `gpio_out0_din = 32'b0`, so ADRV control/status nets such as `RESETB`, enables, MODE, TX/RX enable, DGPIO, AGPIO, MCS, and clock-control nets are all statically driven low or not functionally controlled.
   - ADRV9002 data sheet requires RESET to be toggled after supplies stabilize.
   - **Action:** Before hardware release, replace placeholder GPIO constants with real control/status logic and reset sequencing. In particular, ensure `RESETB` is released after rails are valid.

## Additional checks before freeze

- **Power sequence:** ADRV9002 requires `VDD_1P0` first, then `VDDA_1P3` / `VDDA_1P8`; reset must toggle after rails stabilize. Schematic has 1.0 V, 1.3 V, and 1.8 V RF rails, but sequencing/PG/reset relationship should be confirmed.
- **External LO pins:** ADRV9002 data sheet says unused `EXT_LO1+/-` and `EXT_LO2+/-` must connect to VSSA. Schematic note says external LO is unused; verify actual ties to analog ground.
- **SPI/control voltage level:** ADRV9002 SPI, DGPIO, and control signals are referenced to `VDIGIO_1P8`. FPGA bank 2A control pins are on the 1.3 V HSIO domain. The schematic appears to use level translators; confirm VCCA/VCCY, OE, and direction behavior for all SPI/control/GPIO nets.
- **Interface mode:** The schematic routes LVDS/LSSI-style 2-lane I/Q pairs. ADRV9002 data sheet says LSSI 2-lane mode supports the full 40 MHz bandwidth. Confirm software profile uses LVDS LSSI 2-lane if full bandwidth is required.
- **Naming cleanup:** QSF/RTL still use `adrv9001_*` signal names while the schematic/device is ADRV9002. This is not necessarily electrical, but it is a documentation/integration risk and should be cleaned up.

## References from `adrv9002.pdf`

- Page 23: `MCS+/-` unused connection requirement; `DEV_CLK_IN+/-` behavior.
- Page 24: SPI/control pins, `RESETB`, `GP_INT`, `MODE`, `DEV_CLK_OUT`, `DGPIO`, supply domains.
- Pages 25-26: LVDS SSI data-clock/data/strobe pin directions.
- Page 97: power sequencing and reset requirement.
- Page 98: ADRV9002 data-port interface modes and LSSI 2-lane bandwidth capability.
