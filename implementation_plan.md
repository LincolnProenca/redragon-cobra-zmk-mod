# 🔌 Projeto: Alimentação Independente dos LEDs WS2812

## Resumo da Solução

Alimentar os LEDs **diretamente da bateria (VBAT)** em vez do regulador 3.3V, com um **MOSFET de corte** controlado por um GPIO dedicado para desligar completamente os LEDs durante sleep.

```
                    ANTES (problemático)
    ┌────────────────────────────────────────┐
    │  Bateria 3.7V → LDO → 3.3V → nRF52840 │
    │                          ├──→ PMW3610  │
    │                          └──→ 9x LEDs  │ ← sobrecarga do LDO!
    └────────────────────────────────────────┘

                    DEPOIS (proposto)
    ┌────────────────────────────────────────┐
    │  Bateria 3.7V → LDO → 3.3V → nRF52840 │
    │       │                  └──→ PMW3610  │
    │       │                                │
    │       └──→ MOSFET ──→ 9x LEDs VDD     │ ← direto da bateria
    │            (GPIO)                      │
    └────────────────────────────────────────┘
```

---

## ⚡ Preciso de boost 5V para os LEDs?

**Não.** Os WS2812 são especificados para **3.5V–5.3V**. A bateria LiPo fornece **3.7V–4.2V**, que está dentro da faixa. Além disso:

| Parâmetro | Valor |
|---|---|
| VDD dos WS2812 | 3.5V – 5.3V ✅ |
| Tensão da bateria LiPo | 3.7V – 4.2V ✅ |
| Nível lógico HIGH (VIH) | 0.7 × VDD = 2.59V min |
| Nível da data line (3.3V) | 3.3V > 2.59V ✅ |

> [!TIP]
> A 3.7V os LEDs funcionam perfeitamente, só com brilho ~10-15% menor que a 5V. Para um mouse com LEDs decorativos, é imperceptível. Um boost converter adicionaria complexidade, custo, espaço e consumo próprio (~1-2mA quiescente).

Se mesmo assim quiser 5V (máximo brilho), veja a [Seção Opcional: Boost 5V](#opcional-boost-5v-para-máximo-brilho) no final.

---

## 🧩 Componentes Necessários

### Circuito MOSFET (obrigatório)

| # | Componente | Modelo Sugerido | Encapsulamento | Função |
|---|---|---|---|---|
| 1 | P-MOSFET | **AO3401** ou SI2301 | SOT-23 | Chave de potência (corta VDD dos LEDs) |
| 2 | N-MOSFET | **2N7002** ou BSS138 | SOT-23 | Driver do gate (converte 3.3V → VBAT) |
| 3 | Resistor | **10kΩ** | 0805 ou THT | Pull-up do gate do P-MOSFET |
| 4 | Resistor | **100kΩ** | 0805 ou THT | Pull-down do gate do N-MOSFET |

### Proteção (recomendado)

| # | Componente | Modelo Sugerido | Encapsulamento | Função |
|---|---|---|---|---|
| 5 | Capacitor | **100µF 10V** | Eletrolítico radial ou tantalum | Filtro de inrush current dos LEDs |
| 6 | Capacitor | **100nF** | 0805 | Desacoplamento HF perto dos LEDs |

> [!NOTE]
> Todos esses componentes são baratos (<R$1 cada), fáceis de encontrar, e podem ser soldados com ferro de solda comum. Os SOT-23 têm 3 pinos com espaçamento de 0.95mm — perfeitamente soldáveis com um pouco de cuidado.

---

## 📐 Diagrama do Circuito

```
                            VBAT (pino RAW do Nice!Nano)
                              │
                              ├──── 10kΩ ────┐
                              │              │
                         ┌────┴────┐         │
                         │ SOURCE  │         │
                         │         │         │
                         │ AO3401  │         │
                         │ P-MOS   ├─────────┘
                         │         │ GATE
                         │ DRAIN   │
                         └────┬────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                  100µF              LED VDD
                  (VBAT)          (fio vermelho
                    │              da fita WS2812)
                   GND
                              │
                              │ DRAIN
                         ┌────┴────┐
                         │ 2N7002  │
   GPIO P1.11 ───────────┤ N-MOS   │
   (Nice!Nano D14)       │         │
                         │ SOURCE  │
                         └────┬────┘
                              │
                    ┌─────────┤
                    │         │
                  100kΩ      GND
                    │
                   GND
```

### Como funciona:

| Estado do GPIO | N-MOSFET | Gate do P-MOSFET | P-MOSFET | LEDs |
|---|---|---|---|---|
| **HIGH** (ativo) | ON | Puxado p/ GND | **ON** | ⚡ Ligados |
| **LOW** (sleep) | OFF | Puxado p/ VBAT (via 10kΩ) | **OFF** | 💤 Desligados |

> [!IMPORTANT]
> Quando o P-MOSFET está OFF, os LEDs ficam **completamente desconectados** da energia. Os controladores WS2812 param de consumir standby (~1mA/LED × 9 = 9mA economizados no sleep).

---

## 🔌 GPIO Livre para o MOSFET

Analisando o [cobaia.overlay](file:///d:/GitHub/cobaia/boards/shields/cobaia/cobaia.overlay), os GPIOs em uso são:

```
P0.06 ── botão LMB         P0.29 ── encoder A
P0.08 ── botão RMB         P0.31 ── encoder B
P0.09 ── SPI1 CS           P1.00 ── botão TOP_2
P0.10 ── SPI1 MOSI/MISO    P1.04 ── SPI3 LED DIN
P0.11 ── botão BTN_INF     P1.06 ── PMW3610 IRQ
P0.13 ── ext_power          P1.13 ── SPI1 SCK
P0.17 ── botão MMB         P1.15 ── botão TOP_3
P0.20 ── botão SIDE_1
P0.22 ── botão SIDE_2
P0.24 ── botão TOP_1
```

**GPIOs livres no Nice!Nano v2:**

| Pino nRF52840 | Pino Pro Micro | Status | Recomendação |
|---|---|---|---|
| **P1.11** | D14 | ✅ Livre | **← Recomendado** (fácil acesso) |
| **P0.02** | D16 | ✅ Livre | Alternativa (fica do outro lado) |

---

## 🔧 Instruções de Montagem

### Passo 1: Preparar os fios dos LEDs

1. **Dessolde o fio VDD (vermelho) dos LEDs** do ponto onde ele se conecta ao 3.3V no PCB
2. **Mantenha o fio GND e o fio DIN** conectados como estão
3. O fio DIN continua conectado ao P1.04 (SPI3) normalmente

### Passo 2: Montar o circuito MOSFET

Você pode montar tudo em um pequeno pedaço de **placa perfurada** (~1cm × 1cm) ou soldar os componentes "no ar" (dead-bug style):

1. **Solde o AO3401 (P-MOSFET):**
   - Pino 1 (Gate) → resistor 10kΩ → VBAT
   - Pino 2 (Source) → fio de VBAT (pino RAW do Nice!Nano)
   - Pino 3 (Drain) → fio VDD dos LEDs + capacitor 100µF para GND

2. **Solde o 2N7002 (N-MOSFET):**
   - Pino 1 (Gate) → fio para GPIO P1.11 no Nice!Nano + resistor 100kΩ → GND
   - Pino 2 (Source) → GND
   - Pino 3 (Drain) → Gate do P-MOSFET (pino 1 do AO3401)

3. **Solde o capacitor 100nF** entre LED VDD e LED GND, o mais perto possível da fita

### Passo 3: Conectar ao Nice!Nano

Três fios vão do circuito MOSFET ao Nice!Nano:

| Fio | De | Para |
|---|---|---|
| VBAT | Pino **RAW** do Nice!Nano | Source do P-MOSFET |
| GPIO | Pino **P1.11** (D14) do Nice!Nano | Gate do N-MOSFET |
| GND | Pino **GND** do Nice!Nano | Source do N-MOSFET + cap |

---

## 💻 Alterações de Firmware

### Mudanças no overlay — [cobaia.overlay](file:///d:/GitHub/cobaia/boards/shields/cobaia/cobaia.overlay)

Adicionar um **segundo nó ext_power dedicado aos LEDs** no GPIO P1.11:

```dts
/* Dentro do bloco / { ... }, APÓS o cobaia_ext_power existente: */

led_ext_power: led_ext_power {
    compatible = "zmk,ext-power-generic";
    control-gpios = <&gpio1 11 GPIO_ACTIVE_HIGH>;
};
```

> [!NOTE]
> `GPIO_ACTIVE_HIGH` porque o circuito N+P MOSFET inverte a lógica: GPIO HIGH → LEDs ON.

### Mudanças no conf — [cobaia.conf](file:///d:/GitHub/cobaia/config/cobaia.conf)

```conf
# ==========================================
# POWER AND RGB UNDERGLOW LEDS
# ==========================================
# 1. Garante que o pino VCC (Sensor) fique sempre ligado
CONFIG_ZMK_EXT_POWER=y

# 2. PERMITE que os LEDs controlem o ext_power (desliga no sleep)
CONFIG_ZMK_RGB_UNDERGLOW_EXT_POWER=y

# 3. Drivers da Fita de LED
CONFIG_LED_STRIP=y
CONFIG_WS2812_STRIP=y
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=3
CONFIG_ZMK_RGB_UNDERGLOW_SPD_START=3
CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=50
CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE=y
CONFIG_ZMK_RGB_UNDERGLOW_ON_START=y
```

> [!WARNING]
> **Atenção sobre o ZMK e múltiplos ext_power:** O ZMK por padrão suporta **um único** `ext_power` via `chosen`. Se o segundo nó não funcionar automaticamente, a alternativa é:
> 1. Manter o `cobaia_ext_power` (P0.13) para o sensor
> 2. Controlar o P1.11 manualmente via `&ext_power EP_ON`/`EP_OFF` no keymap
> 3. Ou trocar a lógica: sensor no 3.3V direto (sem MOSFET), P0.13 para os LEDs (usando o MOSFET que já existe no PCB)

---

## 🔐 Segurança do Circuito

O circuito proposto garante:

| Proteção | Como |
|---|---|
| **Sem sobrecarga do LDO** | LEDs alimentados direto da bateria, não passam pelo regulador 3.3V |
| **Corte total no sleep** | P-MOSFET desconecta completamente os LEDs (0µA standby) |
| **Sem inrush damage** | Capacitor 100µF amortece picos de corrente na inicialização |
| **Fail-safe no boot** | Pull-down 100kΩ no N-MOSFET → LEDs OFF por padrão até o firmware ativar |
| **Proteção contra bateria baixa** | WS2812 funciona até 3.5V; abaixo disso os LEDs apagam sozinhos sem danos |

---

## Opcional: Boost 5V para Máximo Brilho

Se preferir 5V para os LEDs (brilho máximo, cores mais vivas):

| Componente | Modelo | Preço aprox. |
|---|---|---|
| Módulo boost | **MT3608** (plaquinha pronta) | ~R$5 |
| Tamanho | ~15mm × 10mm | Cabe no mouse |

Wiring com boost:
```
VBAT → P-MOSFET → MT3608 (ajustar trimpot para 5.0V) → LED VDD
```

> [!CAUTION]
> O MT3608 tem **quiescente de ~1-2mA** mesmo sem carga. Por isso é **essencial** que o MOSFET fique **antes** do boost, cortando a entrada dele no sleep. Sem o MOSFET, o boost sozinho drenaria a bateria em ~1-2 semanas.

---

## 📋 Resumo: Lista de Compras

| Componente | Qtd | Preço unitário aprox. |
|---|---|---|
| AO3401 (P-MOSFET SOT-23) | 1 | R$0,50 |
| 2N7002 (N-MOSFET SOT-23) | 1 | R$0,30 |
| Resistor 10kΩ (0805 ou THT) | 1 | R$0,10 |
| Resistor 100kΩ (0805 ou THT) | 1 | R$0,10 |
| Capacitor 100µF 10V eletrolítico | 1 | R$0,50 |
| Capacitor 100nF (0805) | 1 | R$0,10 |
| Nice!Nano v2 (reposição) | 1 | ~R$90-120 |
| **Total (sem Nice!Nano)** | | **~R$1,60** |
