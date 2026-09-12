# Processador MIPS Pipeline

Este repositório contém o projeto e a implementação em VHDL de um **processador de 4 bits baseado na arquitetura MIPS Pipeline**. O projeto abrange os 5 estágios clássicos de execução, validação por simulação (ISim / Xilinx ISE) e demonstração prática em hardware na placa **FPGA Digilent Nexys 3**.

Projeto desenvolvido para a disciplina de **Arquitetura de Computadores**

---

## 👥 Autores

* **Ana Luiza Boeng**
* **Gabriele Bueno Santos**
* **Mona Aya Kanso**
* **Marcello Eduardo Pereira**

---

## 📌 Visão Geral da Arquitetura

O processador segue um modelo simplificado de pipeline de 5 estágios sem unidade de detecção de hazard nem unidade de adiantamento. Devido a isso, a resolução de hazards de dados deve ser feita via software, inserindo instruções de bolha (`NOP`) entre instruções dependentes.

### Diagrama em Bloco da Arquitetura

```

+------+     +-------+     +------+     +-------+     +------+
|  IF  | ==> |  ID   | ==> |  EX  | ==> |  MEM  | ==> |  WB  |
+------+     +-------+     +------+     +-------+     +------+
(Fetch)       (Decode)      (Execute)    (Memory)     (WriteBack)

```

### Estágios do Pipeline

1. **IF (Instruction Fetch):** Incrementa sequencialmente o *Program Counter* (PC) e busca a instrução de 12 bits na memória ROM.
2. **ID (Instruction Decode):** Decodifica a instrução, gera os sinais de controle, realiza extensão de zeros (*Zero Extension*) do imediato e lê os registradores.
3. **EX (Execute):** Processa operações aritméticas e lógicas através da ULA (4 bits) e seleciona operandos via multiplexadores.
4. **MEM (Memory Access):** Realiza operações de leitura e escrita na RAM de dados (16 posições de 4 bits).
5. **WB (Write-Back):** Seleciona via multiplexador (`MemtoReg`) se o dado a ser gravado no Banco de Registradores vem da ULA ou da Memória.

### Registradores Inter-estágios
Para garantir a sincronia e o fluxo correto dos dados ciclo a ciclo (captura na borda de subida do clock):
* `IF/ID`: Armazena a instrução lida da ROM.
* `ID/EX`: Armazena operandos lidos, imediato e sinais de controle.
* `EX/MEM`: Armazena resultados da ULA e controles de memória.
* `MEM/WB`: Armazena dados lidos da memória/ULA e endereço de destino.

---

## 💻 Conjunto de Instruções (ISA)

As instruções possuem tamanho fixo de 12 bits.

### Formatos das Instruções

* **Tipo R:** `[ OPCODE (3b) | RS (3b) | RT (3b) | RD (3b) ]`
* **Tipo I:** `[ OPCODE (3b) | RS (3b) | RT/RD (3b) | IMM (3b) ]`

### Instruções Suportadas

| Instrução | Tipo | Opcode (Binário) | Descrição |
| :--- | :---: | :---: | :--- |
| **NOP** | - | `000` | Nenhuma operação (utilizado para evitar hazards) |
| **ADD** | R | `001` | Soma: `RD = RS + RT` |
| **SUB** | R | `010` | Subtração: `RD = RS - RT` |
| **AND** | R | `011` | Operação lógica AND: `RD = RS AND RT` |
| **LW** | I | `100` | Load Word: `RT = Memoria[RS + IMM]` |
| **SW** | I | `101` | Store Word: `Memoria[RS + IMM] = RT` |
| **ADDI** | I | `110` | Soma Imadiata: `RT = RS + IMM` |

### Mapas de Registradores

| Registrador | Endereço | Descrição / Função |
| :---: | :---: | :--- |
| **R0** | `000` | Registrador constante Zero (`0x0`) |
| **R1** | `001` | Registrador de uso geral |
| **R2** | `010` | Registrador de uso geral |
| **R3** | `011` | Registrador de uso geral |
| **R4** | `100` | Registrador de uso geral |
| **R5** | `101` | Registrador de uso geral |
| **R6** | `110` | Registrador de uso geral |
| **SP** | `111` | Stack Pointer (Aponta para a memória de dados) |

---

## ⚙️ Sinais de Controle

A Unidade de Controle gera os seguintes sinais base de acordo com a instrução decodificada:

| Instrução | RegDst | RegWrite | ALUSrc | ALUOp | MemWrite | MemRead | MemtoReg |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Lógicas / Aritméticas (R)** | 1 | 1 | 0 | *Varia (`010`/`110`/`000`) | 0 | 0 | 0 |
| **ADDI** | 0 | 1 | 1 | `010` | 0 | 0 | 0 |
| **LW** | 0 | 1 | 1 | `010` | 0 | 1 | 1 |
| **SW** | 1 | 0 | 1 | `010` | 1 | 0 | 0 |

---

## 🧪 Testes e Validação (Simulação)

A validação foi conduzida via **ISim (Xilinx ISE)** através da bancada de testes `tb_pipeline_debug`.

1. **Teste 01 - Instruções Aritméticas e Lógicas:** Validação das operações `ADDI`, `ADD`, `SUB` e `AND` com escrita correta no banco de registradores.
2. **Teste 02 - Instruções de Memória:** Validação da persistência de dados utilizando `SW` e a recuperação com `LW`.
3. **Teste 03 - Instruções Mistas:** Demonstração da execução intercalada entre aritmética e leitura/escrita na RAM.
4. **Teste 04 - Hazard de Dados:** Demonstração prática do problema de dependência de dados ao remover propositalmente a instrução `NOP` entre a produção e o consumo de um valor.

---

## 🚀 Implementação em Hardware (FPGA)

O projeto foi sintetizado e gravado na placa **Digilent Nexys 3** (FPGA Xilinx Spark-6):

* **Displays de 7 Segmentos:** Utilizados para monitorar em tempo real:
  * Os dois dígitos da **esquerda**: Exibem o valor atual do *Program Counter* (`PC`).
  * Os dois dígitos da **direita**: Exibem o dado gravado durante o estágio WB (`wb_write_data`).
* **Botão Central:** Mapeado como sinal de `Reset` síncrono para a arquitetura.

---

## 🛠️ Ferramentas Utilizadas

* **Linguagem HDL:** VHDL
* **Ambiente de Desenvolvimento:** Xilinx ISE Design Suite
* **Simulador:** ISim
* **Placa Alvo:** Digilent Nexys 3 (FPGA Spartan-6)

```