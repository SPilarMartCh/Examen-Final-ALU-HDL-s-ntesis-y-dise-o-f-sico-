# ALU+ parametrizable: Prefix Adder y Divisor, del RTL al GDSII

**Examen final de Diseño Digital (UNIS), con el Ing. Víctor Vargas.**
Autora: Sofia "Pili" Martínez.

Este proyecto es una ALU+ parametrizable en SystemVerilog. Por defecto es de 8 bits y se puede instanciar en 16 o 32 bits. Agrega un sumador **Prefix (Sklansky)** y un **divisor de arreglo**. Se verificó con VCS y Verdi, se sintetizó con Design Compiler y se implementó físicamente con IC Compiler II, usando el PDK GF 150 nm (180MCU) y las celdas estándar M31.

📄 **Informe completo:** [`docs/EXAMEN_FINAL_VICTOR.pdf`](docs/EXAMEN_FINAL_VICTOR.pdf)

---

## Videos (YouTube, no listados)

| Parte | Contenido | Video |
|---|---|---|
| **Parte 1: HDL** | Estructura básica, mux ternario, Prefix Adder, divisor, ALUControl de 3 bits, AND/OR triestado y banderas NZCV | ▶️ [Ver en YouTube](https://youtu.be/PR7IEzdCDtw) |
| **Parte 2: Síntesis y diseño físico** | VCS/Verdi, Design Compiler, IC Compiler II, resultados y GDSII, entregables, presentación y discusión | ▶️ [Ver en YouTube](https://youtu.be/kq-6IF25Txw) |

---

## Parte 1: HDL de la ALU+

| ALUControl | Operación |
|---|---|
| `000` | A + B (Prefix Adder) |
| `001` | A − B (Prefix Adder con ~B y cin = 1) |
| `010` | A AND B (bus triestado) |
| `011` | A OR B (bus triestado) |
| `100` | Reservado para el multiplicador de otro grupo; la salida es 0 |
| `101` | Cociente A / B |
| `110` | Residuo A % B |
| `111` | Todos los bits en 0 |

Banderas `{N, Z, C, V}`:

- **C** solo se activa en suma y resta.
- **V** indica overflow en suma y resta, y también se enciende cuando se divide entre 0.

**Verificación:** el testbench se compara solo contra un modelo de referencia.

| Ancho | Resultado | Checks |
|---|---|---|
| 8 bits | PASS (exhaustivo) | 524,584 |
| 16 bits | PASS (aleatorio) | 16,296 |
| 32 bits | PASS (aleatorio) | 16,296 |

## Parte 2: Síntesis y diseño físico (16 bits, síncrono)

| Métrica | Design Compiler | ICC2 final |
|---|---|---|
| Reloj | 40 ns (25 MHz) | 40 ns (25 MHz) |
| WNS setup | 0.00 ns | 0.00 ns |
| Hold | 5 violaciones pequeñas | +0.89 ns (sin violaciones) |
| Área de lógica | 29,988.4 µm² | 28,909.4 µm² (−3.6 %) |
| Área total con fillers | — | 45,508.4 µm² |
| DRC / LVS | — | 0 / limpio |

- **Simulación del sistema síncrono:** VCS PASS con 2,045 checks; las formas de onda se revisaron en Verdi/nWave.
- **Camino crítico (DC):** de `b_q_reg[1]` a `flags_reg[2]`, pasando por las 16 restas del divisor.
- **WNS por etapa en ICC2:** place_opt +0.06 → clock_opt +0.01 → route_auto −0.56 → route_opt −0.01 → route_opt incremental 0.00.

---

## Estructura del repositorio

```
rtl/          mux.sv, prefix_adder.sv, array_divider.sv,
              tristate_logic_unit.sv, alu_plus.sv, alu_plus_top.sv
tb/           tb_alu_plus.sv (8/16/32 bits), tb_alu_plus_top.sv, tb_demo.sv
sim/          run_all.sh (regresión 8/16/32 con VCS)
constraints/  alu_plus_top.sdc
scripts/      common_setup.tcl (rutas de PDK/M31 del servidor)
syn/          dc_synth.tcl, reports/, results/ (netlist .mapped.v, .sdf, .ddc)
pnr/          make_ndm.tcl, icc2_flow.tcl, fix_timing.tcl,
              reports/ (QoR/timing por etapa), results/ (.pnr.v, .spef, .sdf, .gds),
              alu_plus_top_lib/ (base de datos ICC2 del diseño)
docs/         informe en PDF
Makefile      make sim | syn | ndm | pnr
```

## Cómo reproducir (en el servidor Synopsys UNIS)

```bash
make sim                          # VCS: testbench síncrono (FSDB para Verdi)
cd sim && ./run_all.sh vcs 2000   # regresión de la ALU en 8, 16 y 32 bits
make syn                          # Design Compiler
make ndm                          # M31 → NDM (una sola vez)
make pnr                          # ICC2: floorplan → place → CTS → route → GDSII
cd pnr && icc2_shell -f fix_timing.tcl   # route_opt incremental (WNS 0.00)
```

Para ver el layout:

```
icc2_shell -gui
open_lib alu_plus_top_lib
open_block alu_plus_top/final_fix
```

> **Nota sobre las librerías:** el PDK de GF 150 nm y las celdas M31 tienen licencia, así que **no están incluidos** en el repo. Esto incluye los `.db`, `.lef`, `.tf`, TLU+ y NDM de referencia. La base de datos de ICC2 (`pnr/alu_plus_top_lib`) solo se puede abrir en el servidor donde están instaladas esas librerías.
