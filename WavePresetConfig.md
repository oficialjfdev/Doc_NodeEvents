# WavePresetConfig

## Visao geral

Este arquivo documenta o comportamento real do preset de waves usado pelo `NodeEvents`.

O preset e carregado pelo nome do arquivo usado no comando:

```text
/node.start <zona> <preset>
```

Exemplo:

```text
/node.start z1 teste
```

Nesse caso o plugin procura o arquivo:

```text
carbon/data/NodeEvents/PresetWave/teste.json
```

ou, dependendo do ambiente:

```text
oxide/data/NodeEvents/PresetWave/teste.json
```

Observacoes importantes:

- O nome operacional do preset e o nome do arquivo.
- O campo `PresetName` e metadata interna e nao e usado como chave principal de busca.
- Quando existe `WavePreset` ativo, o preset passa a ser a fonte de verdade para `SpawnCount`, `WaveTiming` e `SpawnFlow` da sessao.
- Nesse caminho com preset, `zone.PlayerScaling` da configuracao da zona nao e aplicado por cima do resultado do preset.
- A dificuldade e recalculada no fim de cada wave e capturada na fila da proxima wave.
- NPCs que ja estao vivos nao sao reescalados quando a dificuldade muda; apenas NPCs novos da nova wave usam a dificuldade recalculada.

## Enums numericos

### WaveMode

- `0`: `Finite`
- `1`: `Endless`

### WavePresetCurveType

- `0`: `Linear`
- `1`: `Exponential`
- `2`: `Logarithmic`
- `3`: `Step`
- `4`: `Hybrid`
- `5`: `Smooth`
- `6`: `Aggressive`
- `7`: `Saturating`

Observacao:

- `Ramp` existe no enum como alias de `Linear`, entao conceitualmente representa o mesmo valor de `0`.

## Sistema de curvas configuraveis

O projeto agora usa um sistema compartilhado de curvas em todos os pontos do preset que dependem de progressao por wave, progressao por dificuldade ou crescimento por degraus de `10` de dificuldade.

Os pontos que usam esse sistema sao:

- `Difficulty.WaveCurve`
- `WaveTiming.AboveNominalCurve`
- `WaveTiming.BelowNominalCurve`
- `SpawnCount.DifficultyScaling.GrowthCurve`
- `Roles[].DifficultyGrowthCurve`
- `NpcPools[].DifficultyGrowthCurve`
- `NpcScaling.Health.AboveNominalCurve`
- `NpcScaling.Health.BelowNominalCurve`
- `NpcScaling.MeleeDamageToPlayers.AboveNominalCurve`
- `NpcScaling.MeleeDamageToPlayers.BelowNominalCurve`
- `NpcScaling.MeleeDamageToCore.AboveNominalCurve`
- `NpcScaling.MeleeDamageToCore.BelowNominalCurve`

### Formula conceitual base

O avaliador de curva recebe um `input` positivo e produz `curveUnits`.

```text
x = max(0, input)
shifted = max(0, x - max(0, InflectionPoint))
```

Depois aplica o tipo de curva:

- `Linear`: `shifted`
- `Exponential`: `exp(shifted * SmoothingFactor) - 1`
- `Logarithmic`: `log(1 + shifted)`
- `Step`: `floor(shifted / StepEveryUnits)`
- `Hybrid`: `shifted * HybridLinearPart + log(1 + shifted) * (1 - HybridLinearPart)`
- `Smooth`: `(1 - exp(-SmoothingFactor * shifted)) / SmoothingFactor`
- `Aggressive`: `pow(shifted, Exponent)`
- `Saturating`: `shifted / (1 + shifted * SaturationFactor)`

Depois disso:

```text
output = curveUnits * BaseSlope
output = aplica Band, se existir
output = clamp por MinimumOutput e MaximumOutput quando esses valores forem > 0
output = max(0, output)
resultado final = output * AmountOrWeight
```

### Campos comuns de `WavePresetCurveSettings`

Esses campos aparecem nas curvas auxiliares do preset, como `AboveNominalCurve`, `BelowNominalCurve`, `GrowthCurve` e `DifficultyGrowthCurve`.

- `CurveType`: tipo da curva.
- `StepEveryUnits`: usado por `Step`; minimo real `1`.
- `HybridLinearPart`: peso da parte linear na curva `Hybrid`; o runtime limita para `0..1`.
- `BaseSlope`: multiplicador aplicado ao resultado bruto da curva antes do `AmountOrWeight`.
- `Exponent`: usado por `Aggressive`; se vier `<= 0`, o runtime trata como `1`.
- `InflectionPoint`: desloca o inicio da curva para frente.
- `SmoothingFactor`: usado por `Exponential` e `Smooth`; se vier `<= 0`, o runtime usa `0.1`.
- `SaturationFactor`: usado por `Saturating`; se vier `<= 0`, o runtime usa `0.25`.
- `MinimumOutput`: piso opcional do resultado da curva antes do `AmountOrWeight`; so entra se for `> 0`.
- `MaximumOutput`: teto opcional do resultado da curva antes do `AmountOrWeight`; so entra se for `> 0`.
- `Bands`: lista opcional de faixas para corrigir a curva por segmento.

### Estrutura de `Bands`

Cada item de `Bands` possui:

- `FromInput`
- `Multiplier`
- `Additive`

Regra de aplicacao:

- o runtime procura a banda com maior `FromInput` que ainda seja `<= input`
- se encontrar, substitui o resultado da curva por:

```text
(output * Multiplier) + Additive
```

### Observacoes de compatibilidade

- Se uma curva auxiliar nao for informada, o runtime cai no fallback daquele ponto especifico.
- Esses fallbacks continuam usando `CurveType = Linear`, `StepEveryUnits = 1` e `HybridLinearPart = 0.5` quando o preset nao fornece algo mais detalhado.
- Isso preserva o comportamento linear antigo quando voce nao adiciona os novos campos.

## Campos raiz

### `SchemaVersion`

- Tipo: `int`
- Uso atual: informativo.
- Nao existe branch de runtime por versao.
- Recomendacao: manter `1` enquanto o formato atual continuar neste schema.

### `PresetName`

- Tipo: `string`
- Uso atual: metadata do preset.
- Se vier vazio, o plugin preenche com o nome de fallback durante a normalizacao.
- Nao substitui o nome do arquivo nem o nome usado em `/node.start`.

### `Description`

- Tipo: `string`
- Uso atual: metadata descritiva.

### `WaveMode`

- Tipo: `enum` numerico.
- Valores:
  - `0 = Finite`
  - `1 = Endless`
- `Finite` encerra o evento ao atingir `FiniteWaveCount`.
- `Endless` ignora `FiniteWaveCount` como limite final.

### `FiniteWaveCount`

- Tipo: `int`
- Uso: quantidade maxima de waves quando `WaveMode = 0`.
- Valor minimo real: `1`.
- Se vier menor que `1`, o plugin corrige para `1`.

### `InterWaveDelaySeconds`

- Tipo: `float`
- Uso: tempo entre o fim de uma wave e o inicio da proxima.
- No fim de cada wave o plugin ja recalcula a dificuldade da proxima wave antes de agendar esse delay.

## `NpcSelection`

### `PreferHigherUnlockWeight`

- Tipo: `float`
- Uso atual: consumido na selecao de `NpcPools`.
- Serve para favorecer pools com `UnlockDifficultyPercent` maior.
- O efeito atual entra como um bonus adicional no peso do pool:

```text
advancedBoost = max(0, UnlockDifficultyPercent / 100) * PreferHigherUnlockWeight * 0.5
```

- O default `0.2` preserva praticamente o mesmo viés antigo que existia hardcoded.
- Se usar `0`, o bonus por unlock deixa de existir.

## `Difficulty`

Formula atual da dificuldade:

```text
difficulty =
  BasePercent
  + WaveCurve
  + AliveCarryOver
  + Players
  + Pressure
  + ObjectiveFailed
  - ObjectiveCompleted
```

Depois disso:

- aplica minimo em `MinimumPercent`
- aplica maximo apenas se `MaximumPercent > 0`

### `BasePercent`

- Tipo: `float`
- Uso: ponto de partida da dificuldade.
- `100` e o nominal.

### `MinimumPercent`

- Tipo: `float`
- Uso: piso da dificuldade final.
- O codigo garante minimo real de `1`.
- Na inicializacao da sessao com preset, o valor inicial tambem respeita esse piso de `1`.

### `MaximumPercent`

- Tipo: `float`
- Uso: teto da dificuldade final.
- Se for `0` ou menor, o preset fica sem teto maximo.

## `Difficulty.WaveCurve`

Usa a wave atual para somar dificuldade ao longo da sessao.

Internamente:

- `wave 1` usa `steps = 0`
- `wave 2` usa `steps = 1`
- `wave 3` usa `steps = 2`

### Campos de `Difficulty.WaveCurve`

`Difficulty.WaveCurve` usa uma estrutura propria chamada `WavePresetCurveContribution`, com estes campos:

- `Enabled`
- `CurveType`
- `AmountPerWave`
- `StepEveryWaves`
- `HybridLinearPart`
- `BaseSlope`
- `Exponent`
- `InflectionPoint`
- `SmoothingFactor`
- `SaturationFactor`
- `MinimumOutput`
- `MaximumOutput`
- `Bands`

### `Enabled`

- Tipo: `bool`
- Liga ou desliga o crescimento da dificuldade por wave.

### `AmountPerWave`

- Tipo: `float`
- Intensidade final da curva.
- O runtime calcula a curva em `steps` e multiplica por `AmountPerWave`.

### `StepEveryWaves`

- Tipo: `int`
- Usado por `CurveType = Step`.
- O codigo corrige para pelo menos `1`.

### Demais campos

- `CurveType`, `HybridLinearPart`, `BaseSlope`, `Exponent`, `InflectionPoint`, `SmoothingFactor`, `SaturationFactor`, `MinimumOutput`, `MaximumOutput` e `Bands` seguem as mesmas regras descritas em `Sistema de curvas configuraveis`.

## `Difficulty.AliveCarryOver`

- Soma dificuldade com base em quantos NPCs sobraram vivos no fim da wave anterior.
- Campos:
  - `Enabled`
  - `AmountPerUnit`

## `Difficulty.Players`

- Soma dificuldade por participantes validos alem do primeiro.
- Campos:
  - `Enabled`
  - `AmountPerUnit`

## `Difficulty.Pressure`

- Soma dificuldade com base no `PressureLevel`.
- Campos:
  - `Enabled`
  - `AmountPerUnit`

## `Difficulty.ObjectiveFailed`

- Soma dificuldade pelo total acumulado de objetivos falhados.
- Campos:
  - `Enabled`
  - `AmountPerUnit`

## `Difficulty.ObjectiveCompleted`

- Reduz a dificuldade pelo total acumulado de objetivos concluidos.
- Campos:
  - `Enabled`
  - `AmountPerUnit`

## `WaveTiming`

Controla a duracao da wave.

Duracao atual:

```text
duration = valor base da wave
```

Depois ajusta por dificuldade:

- se dificuldade > `100`, aplica `AboveNominalCurve` sobre `delta / 10` e multiplica por `SecondsPer10DifficultyAboveNominal`
- se dificuldade < `100`, aplica `BelowNominalCurve` sobre `abs(delta) / 10` e multiplica por `SecondsPer10DifficultyBelowNominal`

Depois aplica:

- minimo em `MinimumDurationSeconds`
- maximo apenas se `MaximumDurationSeconds > 0`

### `BaseDurationSeconds`

- Tipo: `float`
- Duracao base da wave quando nao existe entrada mais especifica em `Entries`.

### `Entries`

- Tipo: `List<WavePresetWaveValueEntry>`
- Estrutura de cada item:
  - `FromWave`
  - `Value`
- O plugin usa a entrada com maior `FromWave` que ainda seja `<= waveAtual`.
- Se `Entries` vier vazio, a normalizacao preenche com uma grade padrao.

### `SecondsPer10DifficultyAboveNominal`

- Tipo: `float`
- Intensidade por unidade de curva acima de `100`.
- Valor negativo encurta a wave quando a dificuldade sobe.
- Valor positivo alonga a wave quando a dificuldade sobe.

### `SecondsPer10DifficultyBelowNominal`

- Tipo: `float`
- Intensidade por unidade de curva abaixo de `100`.
- Valor positivo alonga waves mais faceis.
- Valor negativo encurta waves mais faceis.

### `AboveNominalCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional usada quando `difficultyPercent > 100`.
- Se nao existir, o comportamento cai no fallback linear.

### `BelowNominalCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional usada quando `difficultyPercent < 100`.
- Se nao existir, o comportamento cai no fallback linear.

### `MinimumDurationSeconds`

- Tipo: `float`
- Piso da duracao final.

### `MaximumDurationSeconds`

- Tipo: `float`
- Teto da duracao final.
- Se for `0` ou menor, nao existe limite maximo.

## `SpawnCount`

Controla quantos NPCs a wave gera.

Formula atual:

```text
totalCount = baseCount + playersExtra + difficultyExtra
```

### `BaseCount`

- Tipo: `int`
- Quantidade base de spawns da wave quando nao existe entrada mais especifica em `Entries`.

### `Entries`

- Tipo: `List<WavePresetWaveIntEntry>`
- Estrutura de cada item:
  - `FromWave`
  - `Value`
- O plugin usa a entrada com maior `FromWave` que ainda seja `<= waveAtual`.
- Se `Entries` vier vazio, a normalizacao preenche com uma grade padrao.

## `SpawnCount.PlayerScaling`

### `ExtraSpawnsPerExtraValidPlayer`

- Tipo: `int`
- Quantos spawns extras entram para cada participante valido alem do primeiro.
- Esse campo faz parte do proprio preset e continua valendo normalmente quando o preset esta ativo.

## `SpawnCount.DifficultyScaling`

### Formula atual

```text
over = difficultyPercent - ReferenceDifficultyPercent
raw = over / ExtraSpawnEveryDifficultyPercent
curveUnits = EvaluateCurve(GrowthCurve, raw)
difficultyExtra = floor(curveUnits) ou round(curveUnits)
```

### `ReferenceDifficultyPercent`

- Tipo: `float`
- Ponto de referencia a partir do qual a dificuldade passa a adicionar NPCs extras.

### `ExtraSpawnEveryDifficultyPercent`

- Tipo: `float`
- Quantos pontos de dificuldade sao necessarios para ganhar uma unidade de input para a curva.
- So funciona se for maior que `0`.

### `UseFloor`

- Tipo: `bool`
- Define como arredondar o bonus de dificuldade:
  - `true`: usa `floor`
  - `false`: usa `round`

### `GrowthCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional para transformar o bonus bruto de dificuldade em spawns extras.
- Se nao existir, o comportamento cai no linear antigo.

## `SpawnFlow`

### `SpawnIntervalSeconds`

- Tipo: `float`
- Intervalo base entre um spawn e outro dentro da mesma wave.

### `MinimumSpawnIntervalSeconds`

- Tipo: `float`
- Piso do intervalo real.
- O plugin faz `max(MinimumSpawnIntervalSeconds, SpawnIntervalSeconds)`.
- Em runtime o intervalo final ainda nao cai abaixo de `0.1`.

### `MeleeCoreHitIntervalSeconds`

- Tipo: `float`
- Intervalo entre os golpes de NPCs melee contra o core.

### `MeleeCoreMoveSpeed`

- Tipo: `float`
- Uso atual: declarado e normalizado no preset, mas ainda nao e aplicado pela runtime atual.
- Pode ser mantido como reservado para evolucao futura.

## `Roles`

Define os papeis logicos escolhidos antes do pool de NPC.

Cada spawn escolhe um role elegivel por peso.

Elegibilidade:

- `difficultyPercent >= UnlockDifficultyPercent`

Peso atual:

```text
baseWeight = BaseSpawnWeight ou ExtraSpawnWeight
growthInput = max(0, difficultyPercent - UnlockDifficultyPercent) / 10
growth = EvaluateCurve(DifficultyGrowthCurve, growthInput) * DifficultyGrowthWeight
weight = baseWeight + growth
```

O peso final nunca cai abaixo de `0.01`.

### `Id`

- Tipo: `string`
- Identificador interno do role.
- Deve bater com `NpcPools[].RoleId`.

### `UnlockDifficultyPercent`

- Tipo: `float`
- Dificuldade minima para esse role entrar na rolagem.

### `BaseSpawnWeight`

- Tipo: `float`
- Peso usado nos spawns do bloco base da wave.

### `ExtraSpawnWeight`

- Tipo: `float`
- Peso usado nos spawns extras, acima do `BaseCount`.

### `DifficultyGrowthWeight`

- Tipo: `float`
- Intensidade final da curva de crescimento do peso.

### `DifficultyGrowthCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional para transformar os degraus de `+10` de dificuldade acima do unlock em crescimento de peso.
- Se nao existir, o comportamento cai no linear antigo.

### `CanTargetPlayers`

- Tipo: `bool`
- Se `false`, o NPC desse role nao causa dano em jogador.

### `CanTargetCore`

- Tipo: `bool`
- Se `true`, o role pode operar como atacante do core.

Observacao:

- Se nenhum role for elegivel, o codigo cai no primeiro role valido da lista.

## `NpcPools`

Depois do role ser escolhido, o plugin escolhe um pool compativel com esse role.

Elegibilidade:

- `pool.RoleId == role.Id`
- `difficultyPercent >= UnlockDifficultyPercent`
- `Preset` nao pode estar vazio

Peso atual:

```text
advancedBoost = max(0, UnlockDifficultyPercent / 100) * PreferHigherUnlockWeight * 0.5
growthInput = max(0, difficultyPercent - UnlockDifficultyPercent) / 10
growth = EvaluateCurve(DifficultyGrowthCurve, growthInput) * DifficultyGrowthWeight
weight = SelectionWeight + advancedBoost + growth
```

O peso final nunca cai abaixo de `0.01`.

### `Id`

- Tipo: `string`
- Nome interno do pool.

### `RoleId`

- Tipo: `string`
- Deve apontar para um `Roles[].Id`.

### `Preset`

- Tipo: `string`
- Nome do preset do plugin `NpcSpawn` usado no spawn real.
- Precisa existir no runtime do servidor.

### `UnlockDifficultyPercent`

- Tipo: `float`
- Dificuldade minima para esse pool participar da rolagem.

### `SelectionWeight`

- Tipo: `float`
- Peso base do pool na selecao.

### `DifficultyGrowthWeight`

- Tipo: `float`
- Intensidade final da curva de crescimento do peso.

### `DifficultyGrowthCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional para transformar os degraus de `+10` de dificuldade acima do unlock em crescimento de peso.
- Se nao existir, o comportamento cai no linear antigo.

### `CoreDamageBase`

- Tipo: `float`
- Dano base aplicado no core por esse NPC.
- O dano final no core ainda pode ser multiplicado por `NpcScaling.MeleeDamageToCore`.

Observacao:

- Se nenhum pool elegivel for encontrado, o codigo usa o primeiro pool valido que combine com o role.

## `NpcScaling`

Controla escalas por dificuldade para vida e dano.

### Campos comuns das regras

Cada regra (`Health`, `MeleeDamageToPlayers`, `MeleeDamageToCore`) possui:

- `Enabled`
- `CurveType`
- `PercentPer10DifficultyAboveNominal`
- `PercentPer10DifficultyBelowNominal`
- `AboveNominalCurve`
- `BelowNominalCurve`
- `MinimumMultiplier`
- `MaximumMultiplier`

Formula base do multiplicador:

```text
delta = difficultyPercent - 100
steps = abs(delta) / 10
curve = AboveNominalCurve ou BelowNominalCurve, dependendo do sinal de delta
percent = EvaluateCurve(curve, steps) * PercentPer10Difficulty... 
factor = 1 + percent / 100
```

Depois o fator e limitado por:

- `MinimumMultiplier`
- `MaximumMultiplier` se for maior que `0`

Se `AboveNominalCurve` ou `BelowNominalCurve` nao existirem, o runtime usa o fallback de `CurveType` da propria regra.

### `Enabled`

- Tipo: `bool`
- Liga ou desliga a regra.
- Se `false`, o multiplicador vira `1`.

### `CurveType`

- Tipo: `enum` numerico.
- Agora e consumido de verdade pelo calculo do multiplicador.
- Serve como fallback quando a curva especifica acima ou abaixo do nominal nao e fornecida.

### `PercentPer10DifficultyAboveNominal`

- Tipo: `float`
- Quanto aumentar por unidade de curva acima de `100`.

### `PercentPer10DifficultyBelowNominal`

- Tipo: `float`
- Quanto reduzir por unidade de curva abaixo de `100`.

### `AboveNominalCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional usada quando `difficultyPercent > 100`.

### `BelowNominalCurve`

- Tipo: `WavePresetCurveSettings`
- Curva opcional usada quando `difficultyPercent < 100`.

### `MinimumMultiplier`

- Tipo: `float`
- Piso do multiplicador final.

### `MaximumMultiplier`

- Tipo: `float`
- Teto do multiplicador final.
- Se for `0` ou menor, nao existe teto maximo.

## `NpcScaling.Health`

Uso atual no codigo:

- a regra e resolvida no spawn de cada NPC novo da wave
- o multiplicador calculado e salvo no runtime do NPC
- se o NPC for resetado para o spawn, o mesmo multiplicador salvo e reaplicado
- NPCs que ja estavam vivos nao recebem nova vida automaticamente quando a dificuldade muda depois
- a dificuldade usada pelo NPC e a dificuldade capturada quando a wave atual montou a fila de spawn

Comportamento atual da aplicacao:

- quando o multiplicador de `Health` e diferente de `1`, o plugin aplica apenas scaling de vida
- esse scaling de vida ajusta `MaxHealth` e `health` para o novo valor
- nao escala dano nem attack speed nesse caminho novo

Compatibilidade legada:

- se `Health` resultar em multiplicador `1` e o NPC for `elite melee_core`, o runtime ainda pode cair no fallback legado
- esse fallback legado usa `difficultyPercent / 100`, limitado entre `0.4` e `1.8`
- nesse fallback legado ainda existe scaling de vida e de alguns campos de combate por reflexao
- isso foi mantido para nao quebrar o comportamento antigo quando nenhuma curva real de `Health` esta empurrando o fator para fora de `1`

## `NpcScaling.MeleeDamageToPlayers`

- Aplica multiplicador no dano que NPCs do evento causam aos jogadores.
- So entra quando o role pode atacar jogadores.
- O multiplicador agora tambem respeita `CurveType`, `AboveNominalCurve` e `BelowNominalCurve`.

## `NpcScaling.MeleeDamageToCore`

- Aplica multiplicador no dano causado ao core.
- O dano final e:

```text
CoreDamageBase * multiplicadorDaRegra
```

- O multiplicador agora tambem respeita `CurveType`, `AboveNominalCurve` e `BelowNominalCurve`.

## Exemplo de configuracao com curvas novas

```json
{
  "PresetName": "curvas_avancadas",
  "NpcSelection": {
    "PreferHigherUnlockWeight": 0.35
  },
  "Difficulty": {
    "BasePercent": 100.0,
    "MinimumPercent": 1.0,
    "WaveCurve": {
      "Enabled": true,
      "CurveType": 5,
      "AmountPerWave": 4.0,
      "BaseSlope": 1.1,
      "InflectionPoint": 1.0,
      "SmoothingFactor": 0.18,
      "Bands": [
        { "FromInput": 6.0, "Multiplier": 1.15, "Additive": 0.0 }
      ]
    }
  },
  "WaveTiming": {
    "SecondsPer10DifficultyAboveNominal": -1.0,
    "AboveNominalCurve": {
      "CurveType": 7,
      "BaseSlope": 1.0,
      "SaturationFactor": 0.3
    }
  },
  "SpawnCount": {
    "DifficultyScaling": {
      "ReferenceDifficultyPercent": 40.0,
      "ExtraSpawnEveryDifficultyPercent": 10.0,
      "UseFloor": true,
      "GrowthCurve": {
        "CurveType": 6,
        "Exponent": 1.35,
        "BaseSlope": 1.0
      }
    }
  },
  "Roles": [
    {
      "Id": "ranged",
      "UnlockDifficultyPercent": 40.0,
      "BaseSpawnWeight": 1.0,
      "ExtraSpawnWeight": 1.0,
      "DifficultyGrowthWeight": 0.05,
      "DifficultyGrowthCurve": {
        "CurveType": 4,
        "HybridLinearPart": 0.35
      },
      "CanTargetPlayers": true,
      "CanTargetCore": false
    }
  ],
  "NpcScaling": {
    "Health": {
      "Enabled": true,
      "CurveType": 0,
      "PercentPer10DifficultyAboveNominal": 5.0,
      "PercentPer10DifficultyBelowNominal": 3.0,
      "AboveNominalCurve": {
        "CurveType": 6,
        "Exponent": 1.2,
        "BaseSlope": 1.0,
        "MaximumOutput": 8.0
      },
      "BelowNominalCurve": {
        "CurveType": 5,
        "SmoothingFactor": 0.22,
        "BaseSlope": 1.0
      },
      "MinimumMultiplier": 0.8,
      "MaximumMultiplier": 1.8
    }
  }
}
```

## Recomendacoes praticas

- Use `CurveType = 0` ou curvas omitidas quando quiser preservar o comportamento linear antigo.
- Use `Smooth` para progressao suave no inicio.
- Use `Aggressive` quando quiser escalada tardia mais forte.
- Use `Saturating` quando quiser crescimento que perde forca nas dificuldades muito altas.
- Use `Bands` para dar pequenos ajustes por faixa sem criar uma curva totalmente nova.
- Use `PreferHigherUnlockWeight` para empurrar a rolagem de pools mais avancados sem precisar inflar `SelectionWeight` manualmente.
- Lembre que `MeleeCoreMoveSpeed` continua reservado e ainda nao altera a runtime.
