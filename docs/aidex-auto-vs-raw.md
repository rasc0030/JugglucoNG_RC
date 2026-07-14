# AiDex — as lanes `auto` e `raw`: origem dos dados e processamentos

> **Escopo.** Este documento descreve **apenas o que o código deste repositório faz**. Não faz
> afirmações sobre o que o firmware do transmissor calcula internamente (é fechado), nem sobre
> qual valor é clinicamente correto. Toda afirmação aqui é rastreável a um arquivo e linha.
>
> Driver coberto: `AiDexSensor.kt` (caminho que usa a lib do fabricante). Existe um segundo driver,
> nativo (`drivers/aidex/native/`, `AiDexBleManager`), instanciado por
> [`AiDexNativeFactory.kt:57`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexNativeFactory.kt#L57).
> Os dois compartilham a camada nativa de persistência.

---

## 1. Resumo em uma frase

O gráfico plota **dois campos diferentes do mesmo pacote BLE**, cada um com sua própria cadeia de
transformações. **Nenhum ponto do código deriva uma lane da outra** — não existe, em lugar algum do
app, uma equação ligando `auto` e `raw`. Por isso não há garantia de que as curvas se movam juntas.

| | Lane `auto` | Lane `raw` |
|---|---|---|
| Campo de origem | `glucose` (10 bits) | `i1` (canal de corrente) |
| Transformação principal | × fator de compensação de reset | **× 10** |
| Unidade persistida | inteiro **mg/dL** | inteiro **0,1 mmol/L** |
| Degrau de quantização final | 1,00 mg/dL | **1,80 mg/dL** |
| Passa por `cf`/`offset` do sensor | **não** | **não** |
| Pode ser repetida do minuto anterior | não | **sim** (até 2 min) |
| Pode ser substituída pela outra lane | não | **sim** (caminho broadcast) |
| Alimenta os alarmes | **sim, sempre** | não |

---

## 2. Origem: um frame, dois campos

O frame de dados carrega os dois valores em bytes distintos
([`AiDexParser.kt:56-65`](../Common/src/main/java/tk/glucodata/drivers/aidex/native/protocol/AiDexParser.kt#L56-L65)):

```
byte[0]:        opcode
bytes[1..4]:    segundos desde o início do sensor (u32 LE) ÷ 60 = timeOffsetMinutes
bytes[6..7]:    glucosePacked (u16 LE)  → glucose = packed & 0x03FF     ← lane AUTO
bytes[8..9]:    i1 raw channel (u16 LE / 100)                            ← lane RAW
bytes[10..11]:  i2 raw channel (u16 LE / 100)                            (não usado)
bytes[15..16]:  CRC-16
```

`i2` é decodificado mas **nunca usado** para a lane raw — ver §4.1.

---

## 3. Cadeia completa — lane `auto` (linha azul no modo Auto+Raw)

### 3.1 Decodificação

```kotlin
private fun decodePackedGlucoseMgDl(bytes: ByteArray, lowIndex: Int, carryIndex: Int): Int {
    val lo = bytes[lowIndex].toInt() and 0xFF
    val carry = bytes[carryIndex].toInt() and 0xFF
    return lo or ((carry and 0x03) shl 8)
}
```
[`AiDexSensor.kt:7604-7609`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7604-L7609) — inteiro de 10 bits, em mg/dL.

**Fallback de byte baixo:** se o valor empacotado cair fora da faixa válida, o código usa
`payload[5]` **sozinho** (8 bits) como glicose
([`AiDexSensor.kt:7740`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7740),
[`:7879`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7879)).

### 3.2 Compensação de bias pós-reset (aplicada **no app**)

```kotlin
val compensationFactor = getResetCompensationFactor()
val compensatedGlucose = if (compensationFactor != 1.0f) {
    val adjusted = glucoseMgDl * compensationFactor
    adjusted.coerceIn(MIN_VALID_GLUCOSE_MGDL, MAX_VALID_GLUCOSE_MGDL)
} else glucoseMgDl
```
[`AiDexSensor.kt:7983-7997`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7983-L7997)

O fator vem de [`AiDexSensor.kt:9074-9099`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L9074-L9099):

| Tempo desde o reset | Fator | Efeito |
|---|---|---|
| 0–24 h | `1 / CAL_FACTOR_PHASE1` (0,85) | **≈ +18 %** |
| 24–48 h | `1 / CAL_FACTOR_PHASE2` (0,95) | **≈ +5 %** |
| > 48 h | `1.0` (auto-desliga) | nenhum |

Só atua se `resetCompensationEnabled` estiver ligado. **Afeta exclusivamente a lane auto.**

### 3.3 Persistência

`storeAidexReading(...)` → `Natives.aidexProcessData(dataptr, bytes, timeMs, autoGlucose, rawGlucose, 1.0f)`
([`AiDexSensor.kt:8971`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L8971))

No nativo ([`aidex/java.cpp:36`](../Common/src/main/cpp/aidex/java.cpp#L36)):

```cpp
internalVal = (glucose > 0) ? (int)std::round(glucose) : 0;   // inteiro mg/dL
```

Gravado em `polls[i].g` via `savepollallIDs<60>` ([`aidex/java.cpp:94`](../Common/src/main/cpp/aidex/java.cpp#L94)).
O índice do slot é o minuto desde o início do sensor:
`id = (timsec - starttime + 30) / 60` ([`aidex/java.cpp:44-49`](../Common/src/main/cpp/aidex/java.cpp#L44-L49)).

### 3.4 Leitura de volta

```cpp
jlong valAuto = 0;
if (autoValid) {
    double cali = calibrateNow(hist, item);
    valAuto = !isnan(cali) ? (jlong)(cali * 10) : (jlong)item.g * 10;
}
```
[`g.cpp:1615-1619`](../Common/src/main/cpp/g.cpp#L1615-L1619) — mg/dL × 10. O Kotlin divide por 10
([`GlucoseRepository.kt:150`](../Common/src/mobile/java/tk/glucodata/data/GlucoseRepository.kt#L150)).

**Resultado: mg/dL, degrau de 1,00 mg/dL.**

---

## 4. Cadeia completa — lane `raw` (linha cinza no modo Auto+Raw)

### 4.1 Decodificação e escala

```kotlin
private fun decodeVendorRawTuplePacked(bytes: ByteArray, offset: Int): VendorRawTuple? {
    val i1Raw = u16Le(bytes, offset)
    ...
    return VendorRawTuple(i1 = i1Raw / 100f, i2 = i2Raw / 100f, vc = vcRaw / 100f, ...)
}
```
[`AiDexSensor.kt:7611-7623`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7611-L7623)

```kotlin
private fun selectVendorRawLane(i1: Float): Float {
    // Raw is always derived from i1 lane only.
    // No i2 fallback in any mode.
    val i1Candidate = i1.takeIf { it.isFinite() && it > 0f } ?: return 0f
    return (i1Candidate * 10f).coerceAtMost(MAX_VALID_GLUCOSE_MGDL)
}
```
[`AiDexSensor.kt:7767-7772`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7767-L7772)

**A lane raw é `i1 × 10`, passando a ser tratada como mg/dL.** O fator 10 é uma constante do código,
não um parâmetro do sensor. O parser nativo registra explicitamente duas convenções concorrentes para
o mesmo `i1` ([`AiDexParser.kt:155-156`](../Common/src/main/java/tk/glucodata/drivers/aidex/native/protocol/AiDexParser.kt#L155-L156)):

```
 *     rawValue = i1 * 10 (matches Android selectVendorRawLane)
 *     sensorGlucose = i1 * 18.0182 (treating i1 as mmol/L -> mg/dL)
```

### 4.2 Gate

`gateVendorRawCandidate` zera o candidato se exceder `MAX_VALID_GLUCOSE_MGDL`; não aplica piso
inferior ([`AiDexSensor.kt:7712-7729`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7712-L7729)).
`HistoryMerge.normalizeRawMgDl` trata valores implausíveis como **ausentes**
([`HistoryMerge.kt:63-70`](../Common/src/main/java/tk/glucodata/drivers/aidex/native/data/HistoryMerge.kt#L63-L70)).

### 4.3 Carry-forward (repetição do valor anterior)

```kotlin
private fun recentStoredRaw(now: Long): Float {
    if (lastRawTime == 0L) return 0f
    if ((now - lastRawTime) > 2 * 60_000L) return 0f
    return lastRawMgDl.takeIf { it.isFinite() && it > 0f } ?: 0f
}
```
[`AiDexSensor.kt:8055-8059`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L8055-L8059)

Usado quando o `i1` não vem no frame
([`AiDexSensor.kt:8566-8572`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L8566-L8572)):

```kotlin
val recentRaw = recentStoredRaw(now)
if (recentRaw > 0f) {
    rawForStore = recentRaw
    Log.d(TAG, "Raw i1 unavailable ($source): reusing recent raw=$recentRaw at offset=$offsetMinutes")
}
```

**Consequência: valores repetidos produzem segmentos retos no gráfico.**

### 4.4 Substituição pela lane auto (caminho broadcast)

```kotlin
private fun storeRawFromBroadcastIfNeeded(glucoseMgDl: Float, offsetMinutes: Long, timeMs: Long) {
    if (!broadcastEnabled) return
    if (!rawBroadcastFallbackEnabled) return
    if (!wantsRaw()) return
    ...
    storeAidexReading(byteArrayOf(0), timeMs, glucoseMgDl, glucoseMgDl)   // auto == raw
}
```
[`AiDexSensor.kt:7173-7196`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7173-L7196)

Nesses pontos a lane raw **não é `i1`** — recebe o próprio valor de glicose. As duas lanes coincidem.

### 4.5 Persistência — conversão de unidade

```cpp
rawVal = 0;
if (rawGlucose > 0) {
    constexpr float mgdlToMmol = 1.0f / 18.0182f;
    rawVal = (int)std::round(rawGlucose * mgdlToMmol * 10.0f);
}
```
[`aidex/java.cpp:37-41`](../Common/src/main/cpp/aidex/java.cpp#L37-L41)

O valor (que é `i1 × 10`) é convertido para **0,1 mmol/L** e **arredondado para inteiro**, guardado em
`rawpolls[i].raw`, que é um `uint16_t`
([`SensorGlucoseData.hpp:138-140`](../Common/src/main/cpp/SensorGlucoseData.hpp#L138-L140)).

### 4.6 Leitura de volta

```cpp
const jlong valRaw = rawVal > 0 ? (jlong)(rawVal * convfactordL) : 0;   // convfactordL = 18.0182
```
[`g.cpp:1621`](../Common/src/main/cpp/g.cpp#L1621) — volta a mg/dL × 10; o Kotlin divide por 10
([`GlucoseRepository.kt:151`](../Common/src/mobile/java/tk/glucodata/data/GlucoseRepository.kt#L151)).

**Round-trip:** mg/dL → 0,1 mmol/L (inteiro) → mg/dL.
**Resultado: `i1 × 10`, quantizado em degraus de 1,80 mg/dL.** A lane auto não sofre essa quantização.

---

## 5. O que o código **nunca** faz: `cf` e `offset`

O app lê do sensor os registros de calibração — referência, fator e offset
([`AiDexParser.kt:203-230`](../Common/src/main/java/tk/glucodata/drivers/aidex/native/protocol/AiDexParser.kt#L203-L230)):

```kotlin
CalibrationRecord(
    index = startIndex + i,
    timeOffsetMinutes = timeOffset,
    referenceGlucoseMgDl = reference,
    calibrationFactor = cfRaw / 100f,
    calibrationOffset = offsetRaw / 100f,
)
```

Esses campos são **parseados, logados e exibidos na UI**
(`"CF: ${"%.2f".format(cal.cf)}"`, [`SensorCard.kt:1881`](../Common/src/mobile/java/tk/glucodata/ui/SensorCard.kt#L1881))
e **não entram em nenhuma operação aritmética**. Nenhuma das duas lanes é transformada por eles
dentro do app.

---

## 6. Os três caminhos que gravam dados

| Caminho | Auto | Raw | Alinhamento temporal |
|---|---|---|---|
| **Ao vivo** (F003 / AUTO_UPDATE) | `glucose` do frame | `i1 × 10` do **mesmo** frame | mesma chamada, mesmo `timeMs` — **não pode desalinhar** |
| **Backfill de histórico** | `entity.glucose` do `GET_HISTORIES` | `vendorRawHistoryByOffset[offset]` do `GET_HISTORIES_RAW` | **casados por um campo `offset`**, vindos de **dois downloads paginados independentes** |
| **Broadcast** | `glucoseMgDl` | **`glucoseMgDl`** (a própria auto) | mesma chamada |

O merge do backfill:

```kotlin
val rawSample = vendorRawHistoryByOffset.remove(offset)
val rawMgDl = selectRawHistoryCandidate(rawSample)
val didStore = storeHistoryRecord(glucose, offset, entity.status, entity.quality, entity.isValid, rawMgDl)
```
[`AiDexSensor.kt:4107-4115`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L4107-L4115)

Os dois parsers calculam o offset da mesma forma (`startOffset + idx`), mas sobre payloads de formatos
diferentes — linhas de 2 bytes (glicose) e de 5 bytes (`i1`/`i2`/`vc`)
([`AiDexSensor.kt:7625-7677`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7625-L7677)).
Existe ainda um prefetch paginado, com teto de 4 páginas, perseguindo a cobertura da lane raw
([`AiDexSensor.kt:4038-4055`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L4038-L4055)).

> **Questão em aberto.** Se o `offset` das duas respostas de histórico não estiver na mesma base de
> tempo, o merge emparelha o `i1` de um minuto com a glicose de outro. O código **assume** que estão.
> Isso não foi confirmado nem refutado — ver §11.

Timestamp do registro histórico: `sensorstartmsec + offsetMinutes * 60_000`
([`AiDexSensor.kt:9286`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L9286)).
`storeHistoryRecord` também descarta: warmup (< 7 min), sentinela de saturação do ADC (`glucoseInt >= 1023`),
fora de faixa, e registros no futuro
([`AiDexSensor.kt:9253-9326`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L9253-L9326)).

---

## 7. `viewMode` — o que ele controla

Os quatro modos (`Auto`, `Raw`, `Auto+Raw`, `Raw+Auto`) estão em
[`strings.xml:557-560`](../Common/src/main/res/values/strings.xml#L557-L560).

### 7.1 Cores e espessuras

```kotlin
val doTintRaw  = !hasCalibration && (viewMode == 1 || viewMode == 3)
val doTintAuto = !hasCalibration && (viewMode == 0 || viewMode == 2)

val rawColor  = when { hasCalibration || viewMode == 2 -> secondaryColor; else -> primaryColor }
val autoColor = when { hasCalibration || viewMode == 3 -> secondaryColor; else -> primaryColor }
```
[`DashboardChart.kt:2387-2403`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L2387-L2403)

| `viewMode` | Linha colorida (grossa, com gradiente) | Linha cinza (fina) |
|---|---|---|
| `2` — Auto+Raw | **auto** | raw |
| `3` — Raw+Auto | **raw** | auto |

**As lanes trocam de cor entre os modos 2 e 3.**

### 7.2 O cabeçalho é ambíguo

[`DisplayValueResolver.kt:150-183`](../Common/src/main/java/tk/glucodata/ui/DisplayValueResolver.kt#L150-L183):

- `viewMode == 2` → formata `"$valStr · $rawStr"` (**auto · raw**)
- `viewMode == 3` → formata `"$rawStr · $valStr"` (**raw · auto**)

O layout é idêntico. `156 · 130` significa coisas opostas nos dois modos.

### 7.3 O que segue o `viewMode` e o que não segue

| Consumidor | Segue `viewMode`? | Referência |
|---|---|---|
| Linha destacada no gráfico | sim | [`DashboardChart.kt:2387-2403`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L2387-L2403) |
| Número principal no cabeçalho | sim | [`DisplayValueResolver.kt:150-183`](../Common/src/main/java/tk/glucodata/ui/DisplayValueResolver.kt#L150-L183) |
| Exportação externa (xDrip/Nightscout) | **sim** — exporta o valor primário | [`ExchangeGlucosePayload.java:137-152`](../Common/src/main/java/tk/glucodata/ExchangeGlucosePayload.java#L137-L152) |
| **Alarmes** | **não** — sempre a lane auto | [`aidex/java.cpp:96`](../Common/src/main/cpp/aidex/java.cpp#L96), [`g.cpp:1171-1189`](../Common/src/main/cpp/g.cpp#L1171-L1189) |

O nativo chama `glucoseback(timsec, internalVal, change, sens)` com o valor **auto**, e o código de
alarme é derivado dele (`getalarmonly(mgL, ...)`).

> **Implicação.** Nos modos `Raw` (1) e `Raw+Auto` (3), o número exibido e o valor exportado para
> integrações passam a ser `i1 × 10` — um canal de corrente escalado por uma constante do código,
> sem `cf` nem `offset` — enquanto os alarmes continuam disparando pela lane auto.

---

## 8. Processamento comum às duas lanes (camada de UI)

1. **Suavização** — média móvel de janela `±N/2` minutos, aplicada **separadamente** a cada lane
   ([`DashboardChart.kt:307-315`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L307-L315),
   [`DataSmoothing.kt:181-189`](../Common/src/main/java/tk/glucodata/DataSmoothing.kt#L181-L189)).
   Opções: 0, 2, 3, 4, 5, 7, 10, 13 min ([`DataSmoothing.kt:15-16`](../Common/src/main/java/tk/glucodata/DataSmoothing.kt#L15-L16)).
   **Padrão: desligada** (`getMinutes` → 0, [`DataSmoothing.kt:30-33`](../Common/src/main/java/tk/glucodata/DataSmoothing.kt#L30-L33)).
   Por ser aplicada igualmente às duas, **não** introduz diferença entre elas.

2. **Calibração do app** — modelo linear por lane; ver §9.

3. **Desenho** ([`DashboardChart.kt:2460-2500`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L2460-L2500)):
   - ponto com valor `< 0.1` → **quebra** a linha (`rawFirst = true`), não interpola;
   - intervalo `> 15 min` entre pontos → quebra a linha;
   - decimação: pontos a menos de 0,8 px em X e 1 px em Y são pulados.

---

## 9. Calibração: aba Sensor vs aba Home

### 9.1 Aba Sensor — envia para o transmissor

Botão disponível para AiDex ([`SensorCard.kt:1790`](../Common/src/mobile/java/tk/glucodata/ui/SensorCard.kt#L1790),
condicionado a `sensor.isAidex` e a BLE do fabricante conectado):

```kotlin
fun calibrateAiDexSensor(serial: String, glucoseMgDl: Int) {
    val gatt = findGatt(serial)
    if (gatt is AiDexDriver) { ... gatt.calibrateSensor(glucoseMgDl) }
}
```
[`SensorViewModel.kt:1147-1155`](../Common/src/mobile/java/tk/glucodata/ui/viewmodel/SensorViewModel.kt#L1147-L1155)

- Age **fora do app** (no transmissor). Não é retroativa.
- **Não afeta a lane raw**: `raw = i1 × 10` é lido antes de qualquer `cf`/`offset`.

### 9.2 Aba Home — modelo linear no app

`CalibrationBottomSheet` → `CalibrationManager` (Room). Cada ponto guarda **os dois** valores e o
modo em que foi criado ([`CalibrationEntity.kt:11-15`](../Common/src/mobile/java/tk/glucodata/data/calibration/CalibrationEntity.kt#L11-L15)):

```kotlin
val sensorValue: Float,      // Auto value at calibration time
val sensorValueRaw: Float,   // Raw value at calibration time
val isRawMode: Boolean = false // Which mode was used to create the calibration
```

O ajuste filtra por modo e escolhe a variável independente conforme o modo
([`CalibrationManager.kt:530-535`](../Common/src/mobile/java/tk/glucodata/data/calibration/CalibrationManager.kt#L530-L535)):

```kotlin
it.isRawMode == isRawMode && ...
CalPoint(x = (if (isRawMode) p.sensorValueRaw else p.sensorValue).toDouble(), ...)
```

- **Modelos independentes** para auto e raw; habilitáveis por sensor e por modo
  ([`CalibrationManager.kt:352-353`](../Common/src/mobile/java/tk/glucodata/data/calibration/CalibrationManager.kt#L352-L353)).
- Retroativa e reversível.
- Um ponto criado em modo Auto **não corrige a lane raw**, e vice-versa.

### 9.3 Resumo

| | Aba Home | Aba Sensor |
|---|---|---|
| Onde age | app (regressão linear) | transmissor (`cf`/`offset`) |
| Afeta `auto` | sim, se criado em modo Auto | sim |
| Afeta `raw` | só se criado em modo Raw | **não** |
| Retroativa | sim | não |
| Reversível pelo app | sim | não |

---

## 10. Consequências observáveis (derivadas do código)

1. **As duas linhas podem divergir em nível.** São campos independentes com escalas independentes; o
   fator 10 da lane raw é uma constante do código, não a calibração do sensor.

2. **A lane raw pode ter trechos retos** que não correspondem a medição nova — carry-forward de até
   2 minutos (§4.3).

3. **A lane raw pode coincidir exatamente com a auto** em trechos servidos pelo caminho broadcast (§4.4).

4. **A lane raw é mais grosseiramente quantizada** (1,80 mg/dL vs 1,00 mg/dL) por causa do round-trip
   mg/dL → 0,1 mmol/L (§4.5).

5. **A lane auto pode saltar ~18 %** ao ligar/expirar a compensação pós-reset, sem que nada tenha
   mudado no sensor (§3.2).

6. **A linha azul não é sempre a auto** — nos modos 1 e 3 é a raw (§7.1).

7. **O gráfico é desenhado sem suavização por padrão**, com um ponto por minuto (§8).

---

## 11. Questões em aberto

O código **não permite concluir** o seguinte; seria necessário instrumentação em campo:

1. **Base de tempo do merge de histórico (§6).** O código assume que o `offset` do `GET_HISTORIES_RAW`
   e o do `GET_HISTORIES` estão na mesma base (minutos desde o início do sensor). Se não estiverem, o
   backfill emparelha lanes desalinhadas no tempo.

   Como verificar — forçar um backfill (reconectar após período desconectado) e capturar:
   ```
   adb logcat -s AiDexSensor | grep -E "Edit 86a|GET_HISTORIES|raw prefetch"
   ```
   O driver loga `cached N raw history rows offsets=A..B` e a resposta do `GET_HISTORIES`. Faixas de
   offset divergentes para o mesmo período confirmariam o desalinhamento.

2. **Frequência real do `i1` nos frames ao vivo.** Se o `i1` vier repetido entre frames, a suavidade da
   linha cinza é carry-forward, não medição. O valor cru é logado em
   [`AiDexSensor.kt:7760-7763`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7760-L7763)
   (`rawTuple=i1/i2/vc`).

3. **Discrepância entre os dois drivers.** O parser nativo mapeia `0x23` → linhas de 2 bytes (glicose) e
   `0x24` → linhas de 5 bytes (ADC)
   ([`AiDexOpcodes.kt:159-163`](../Common/src/main/java/tk/glucodata/drivers/aidex/native/protocol/AiDexOpcodes.kt#L159-L163)).
   O driver da lib do fabricante faz o oposto: `GET_HISTORIES_RAW` → 5 bytes (ADC), `GET_HISTORIES` →
   2 bytes (glicose) ([`AiDexSensor.kt:7625-7677`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7625-L7677)).
   Podem ser transportes distintos com semânticas distintas — ou um dos dois está trocado. Não verificado.

---

## 12. Índice de referências de código

### Kotlin — driver
| Local | Papel |
|---|---|
| [`AiDexSensor.kt:7173-7196`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7173-L7196) | broadcast grava `raw = auto` |
| [`AiDexSensor.kt:7604-7609`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7604-L7609) | decodifica glicose (10 bits) |
| [`AiDexSensor.kt:7611-7623`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7611-L7623) | decodifica `i1`/`i2`/`vc` |
| [`AiDexSensor.kt:7625-7677`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7625-L7677) | parsers de histórico (raw e brief) |
| [`AiDexSensor.kt:7712-7729`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7712-L7729) | gate do candidato raw |
| [`AiDexSensor.kt:7767-7772`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7767-L7772) | **`raw = i1 × 10`** |
| [`AiDexSensor.kt:7983-7997`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L7983-L7997) | compensação pós-reset (auto) |
| [`AiDexSensor.kt:8055-8059`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L8055-L8059) | carry-forward de raw (2 min) |
| [`AiDexSensor.kt:8971`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L8971) | entrada na camada nativa |
| [`AiDexSensor.kt:9074-9099`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L9074-L9099) | fatores da compensação |
| [`AiDexSensor.kt:9253-9326`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L9253-L9326) | gravação de registro histórico |
| [`AiDexSensor.kt:4107-4115`](../Common/src/main/java/tk/glucodata/drivers/aidex/AiDexSensor.kt#L4107-L4115) | merge auto+raw por `offset` |

### C++ — persistência
| Local | Papel |
|---|---|
| [`aidex/java.cpp:36`](../Common/src/main/cpp/aidex/java.cpp#L36) | `internalVal = round(glucose)` |
| [`aidex/java.cpp:37-41`](../Common/src/main/cpp/aidex/java.cpp#L37-L41) | **`rawVal = round(raw / 18.0182 × 10)`** |
| [`aidex/java.cpp:96`](../Common/src/main/cpp/aidex/java.cpp#L96) | `glucoseback(...)` com o valor **auto** |
| [`g.cpp:1602-1626`](../Common/src/main/cpp/g.cpp#L1602-L1626) | exporta `[t, auto×10, raw×10]` |
| [`g.cpp:1171-1189`](../Common/src/main/cpp/g.cpp#L1171-L1189) | código de alarme a partir da auto |
| [`SensorGlucoseData.hpp:138-140`](../Common/src/main/cpp/SensorGlucoseData.hpp#L138-L140) | `RawData { uint16_t raw; }` |

### Kotlin — UI
| Local | Papel |
|---|---|
| [`GlucoseRepository.kt:150-151`](../Common/src/mobile/java/tk/glucodata/data/GlucoseRepository.kt#L150-L151) | `÷ 10` de volta a mg/dL |
| [`DataSmoothing.kt:181-189`](../Common/src/main/java/tk/glucodata/DataSmoothing.kt#L181-L189) | suavização de ambas as lanes |
| [`DashboardChart.kt:2387-2403`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L2387-L2403) | cores por `viewMode` |
| [`DashboardChart.kt:2460-2500`](../Common/src/mobile/java/tk/glucodata/ui/DashboardChart.kt#L2460-L2500) | laço de desenho |
| [`DisplayValueResolver.kt:150-183`](../Common/src/main/java/tk/glucodata/ui/DisplayValueResolver.kt#L150-L183) | valor primário por `viewMode` |
| [`CalibrationManager.kt:530-535`](../Common/src/mobile/java/tk/glucodata/data/calibration/CalibrationManager.kt#L530-L535) | modelo linear por lane |
| [`ExchangeGlucosePayload.java:137-152`](../Common/src/main/java/tk/glucodata/ExchangeGlucosePayload.java#L137-L152) | payload externo |
