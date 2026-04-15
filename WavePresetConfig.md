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

Observacao importante:

- O nome operacional do preset e o nome do arquivo.
- O campo `PresetName` e metadata interna e nao e usado como chave principal de busca.

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

## Campos raiz

### `SchemaVersion`

- Tipo: `int`
- Uso atual: informativo.
- No codigo atual nao existe branch de comportamento por versao.
- Recomendacao: manter `1` enquanto o formato atual nao mudar.

### `PresetName`

- Tipo: `string`
- Uso atual: metadata do preset.
- Se vier vazio, o plugin preenche com o nome de fallback durante a normalizacao.
- Nao substitui o nome do arquivo nem o nome usado em `/node.start`.

### `Description`

- Tipo: `string`
- Uso atual: metadata descritiva.
- Pode ser qualquer texto para identificar o objetivo do preset.

### `WaveMode`

- Tipo: `enum` numerico.
- Valores:
  - `0 = Finite`
  - `1 = Endless`
- `Finite` encerra o evento ao atingir `FiniteWaveCount`.
- `Endless` ignora `FiniteWaveCount` como limite final e o evento continua indefinidamente.

### `FiniteWaveCount`

- Tipo: `int`
- Uso: quantidade maxima de waves quando `WaveMode = 0`.
- Valor minimo real: `1`.
- Se vier menor que `1`, o plugin corrige para `1`.
- Quando `WaveMode = 1`, este campo deixa de ser o limitador da sessao.

### `InterWaveDelaySeconds`

- Tipo: `float`
- Uso: tempo entre o fim de uma wave e o inicio da proxima.
- Quanto maior o valor, maior a pausa entre waves.
- Valores praticos:
  - `5` a `10`: ritmo rapido
  - `10` a `20`: ritmo padrao
  - `20+`: ritmo mais lento

## `NpcSelection`

### `PreferHigherUnlockWeight`

- Tipo: `float`
- Uso atual: declarado no preset, mas nao esta sendo consumido pela selecao de NPCs no codigo atual.
- E seguro manter o valor padrao, mas hoje ele nao muda o comportamento da runtime.

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
- Valores abaixo de `100` tornam a wave base mais leve.
- Valores acima de `100` tornam a wave base mais pesada.

### `MinimumPercent`

- Tipo: `float`
- Uso: piso da dificuldade final.
- O codigo garante minimo real de `1`.
- Mesmo se a soma der menor, a dificuldade nao cai abaixo deste valor.

### `MaximumPercent`

- Tipo: `float`
- Uso: teto da dificuldade final.
- Se for `0` ou menor, o preset fica sem teto maximo.
- Se for maior que `0`, a dificuldade final e limitada a esse valor.

## `Difficulty.WaveCurve`

Usa a wave atual para somar dificuldade ao longo da sessao.

Internamente:

- `wave 1` usa `steps = 0`
- `wave 2` usa `steps = 1`
- `wave 3` usa `steps = 2`

### `Enabled`

- Tipo: `bool`
- Liga ou desliga o crescimento da dificuldade por wave.

### `CurveType`

- Tipo: `enum` numerico
- Valores:
  - `0 = Linear`
  - `1 = Exponential`
  - `2 = Logarithmic`
  - `3 = Step`
  - `4 = Hybrid`

Comportamento atual:

- `Linear`: `AmountPerWave * steps`
- `Exponential`: `AmountPerWave * (exp(steps * 0.1) - 1)`
- `Logarithmic`: `AmountPerWave * log(1 + steps)`
- `Step`: `AmountPerWave * floor(steps / StepEveryWaves)`
- `Hybrid`: mistura uma parte linear com uma parte logaritmica

### `AmountPerWave`

- Tipo: `float`
- Intensidade da curva.
- Quanto maior, mais rapido a dificuldade sobe.

### `StepEveryWaves`

- Tipo: `int`
- Usado apenas em `CurveType = 3`.
- A cada quantas waves a curva em degraus soma mais dificuldade.
- O codigo corrige para pelo menos `1`.

### `HybridLinearPart`

- Tipo: `float`
- Usado apenas em `CurveType = 4`.
- O valor real e limitado para o intervalo `0..1`.
- `0`: totalmente logaritmico
- `1`: totalmente linear
- `0.5`: metade linear, metade logaritmico

## `Difficulty.AliveCarryOver`

- Soma dificuldade com base em quantos NPCs sobraram vivos no fim da wave anterior.

### `Enabled`

- Tipo: `bool`
- Liga ou desliga esse fator.

### `AmountPerUnit`

- Tipo: `float`
- Quanto cada NPC restante soma na dificuldade seguinte.
- Exemplo: se `AliveAtWaveEndLast = 3` e `AmountPerUnit = 5`, soma `15`.

## `Difficulty.Players`

- Soma dificuldade por participantes validos alem do primeiro.

### `Enabled`

- Tipo: `bool`
- Liga ou desliga esse fator.

### `AmountPerUnit`

- Tipo: `float`
- Quanto cada jogador valido extra soma.
- Exemplo: 3 jogadores validos com `AmountPerUnit = 4` gera `8`.

## `Difficulty.Pressure`

- Soma dificuldade com base no `PressureLevel`.
- O `PressureLevel` sobe quando objetivos falham.

### `Enabled`

- Tipo: `bool`

### `AmountPerUnit`

- Tipo: `float`
- Quanto cada nivel de pressure soma.

## `Difficulty.ObjectiveFailed`

- Soma dificuldade pelo total acumulado de objetivos falhados.

### `Enabled`

- Tipo: `bool`

### `AmountPerUnit`

- Tipo: `float`
- Quanto cada objetivo falhado soma.

## `Difficulty.ObjectiveCompleted`

- Reduz a dificuldade pelo total acumulado de objetivos concluidos.

### `Enabled`

- Tipo: `bool`

### `AmountPerUnit`

- Tipo: `float`
- Quanto cada objetivo concluido reduz da dificuldade final.

## `WaveTiming`

Controla a duracao da wave.

Duracao atual:

```text
duration = valor base da wave
```

Depois ajusta por dificuldade:

- se dificuldade > `100`, aplica `SecondsPer10DifficultyAboveNominal`
- se dificuldade < `100`, aplica `SecondsPer10DifficultyBelowNominal`

Depois aplica:

- minimo em `MinimumDurationSeconds`
- maximo apenas se `MaximumDurationSeconds > 0`

### `BaseDurationSeconds`

- Tipo: `float`
- Duracao base da wave quando nao existe uma entrada mais especifica em `Entries`.

### `Entries`

- Tipo: `List<WavePresetWaveValueEntry>`
- Estrutura de cada item:
  - `FromWave`: a partir de qual wave o valor passa a valer
  - `Value`: duracao em segundos

Regra de resolucao:

- o plugin procura todas as entradas com `FromWave <= waveAtual`
- a entrada com maior `FromWave` vence

Exemplo:

```json
[
  { "FromWave": 1, "Value": 60.0 },
  { "FromWave": 4, "Value": 55.0 },
  { "FromWave": 8, "Value": 50.0 }
]
```

Resultado:

- waves `1-3`: `60`
- waves `4-7`: `55`
- wave `8+`: `50`

Observacao:

- Se `Entries` vier vazio, a normalizacao atual preenche com um exemplo padrao.
- A `wave 1` passa a repetir `BaseDurationSeconds`.

### `SecondsPer10DifficultyAboveNominal`

- Tipo: `float`
- Quanto alterar a duracao para cada `+10` pontos acima de `100` de dificuldade.
- Valor negativo encurta a wave quando a dificuldade sobe.
- Valor positivo alonga a wave quando a dificuldade sobe.

Exemplo:

- dificuldade `130`
- delta `+30`
- com `-1.0`, a duracao perde `3` segundos

### `SecondsPer10DifficultyBelowNominal`

- Tipo: `float`
- Quanto alterar a duracao para cada `-10` pontos abaixo de `100` de dificuldade.
- Valor positivo alonga waves mais faceis.
- Valor negativo encurta waves mais faceis.

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
- Quantidade base de spawns da wave quando nao existe uma entrada mais especifica em `Entries`.

### `Entries`

- Tipo: `List<WavePresetWaveIntEntry>`
- Estrutura de cada item:
  - `FromWave`: a partir de qual wave o valor passa a valer
  - `Value`: quantidade base daquela faixa

Regra de resolucao:

- o plugin usa a entrada com maior `FromWave` que ainda seja menor ou igual a wave atual

Observacao:

- Se `Entries` vier vazio, a normalizacao atual preenche com um exemplo padrao.
- A `wave 1` passa a repetir `BaseCount`.

## `SpawnCount.PlayerScaling`

### `ExtraSpawnsPerExtraValidPlayer`

- Tipo: `int`
- Quantos spawns extras entram para cada participante valido alem do primeiro.
- Exemplo: com valor `2` e 3 jogadores validos, entram `4` NPCs extras.

## `SpawnCount.DifficultyScaling`

### `ReferenceDifficultyPercent`

- Tipo: `float`
- Ponto de referencia a partir do qual a dificuldade passa a adicionar NPCs extras.
- Abaixo ou igual a esse valor, nao entra bonus por dificuldade.

### `ExtraSpawnEveryDifficultyPercent`

- Tipo: `float`
- Quantos pontos de dificuldade sao necessarios para ganhar `+1` spawn.
- So funciona se for maior que `0`.

Exemplo:

- dificuldade `70`
- referencia `40`
- intervalo `10`
- bonus bruto: `3`

### `UseFloor`

- Tipo: `bool`
- Define como arredondar o bonus de dificuldade:
  - `true`: usa `floor`
  - `false`: usa `round`

Exemplo com referencia `40`, intervalo `10`, dificuldade `46`:

- `UseFloor = true`: `0`
- `UseFloor = false`: `1`

## `SpawnFlow`

### `SpawnIntervalSeconds`

- Tipo: `float`
- Intervalo base entre um spawn e outro dentro da mesma wave.
- Quanto menor, mais rapido os NPCs entram.

### `MinimumSpawnIntervalSeconds`

- Tipo: `float`
- Piso do intervalo real.
- O plugin faz `max(MinimumSpawnIntervalSeconds, SpawnIntervalSeconds)`.
- Em runtime o intervalo final ainda nao cai abaixo de `0.1`.

### `MeleeCoreHitIntervalSeconds`

- Tipo: `float`
- Intervalo entre os golpes de NPCs melee contra o core.
- Quanto menor, mais rapido o core toma dano.

### `MeleeCoreMoveSpeed`

- Tipo: `float`
- Uso atual: declarado e normalizado no preset, mas nao esta sendo aplicado pela runtime atual.
- Pode ser mantido como metadata futura, mas hoje nao muda a movimentacao.

## `Roles`

Define os papeis logicos escolhidos antes do pool de NPC.

Cada spawn escolhe um role elegivel por peso.

Elegibilidade:

- `difficultyPercent >= UnlockDifficultyPercent`

Peso atual:

```text
weight = baseWeight + (growth * DifficultyGrowthWeight)
```

Onde:

- `baseWeight = BaseSpawnWeight` para spawns da base
- `baseWeight = ExtraSpawnWeight` para spawns acima do `baseCount`
- `growth = max(0, difficulty - unlock) / 10`

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
- Peso usado nos spawns que pertencem ao bloco base da wave.

### `ExtraSpawnWeight`

- Tipo: `float`
- Peso usado nos spawns extras, ou seja, aqueles acima do `baseCount`.
- Bom para fazer reforcos extras tenderem a outro role.

### `DifficultyGrowthWeight`

- Tipo: `float`
- Quanto o role ganha de peso a cada `+10` de dificuldade acima do unlock dele.

### `CanTargetPlayers`

- Tipo: `bool`
- Se `false`, o NPC desse role nao causa dano em jogador.

### `CanTargetCore`

- Tipo: `bool`
- Se `true`, o role pode operar como atacante do core.
- Para NPC melee de core este campo precisa estar habilitado.

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
weight = SelectionWeight + advancedBoost + (growth * DifficultyGrowthWeight)
```

Onde:

- `advancedBoost = UnlockDifficultyPercent * 0.001`
- `growth = max(0, difficulty - unlock) / 10`

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
- Quanto o peso cresce a cada `+10` de dificuldade acima do unlock do pool.

### `CoreDamageBase`

- Tipo: `float`
- Dano base aplicado no core por esse NPC.
- E relevante para NPCs que realmente atacam o core.
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
- `MinimumMultiplier`
- `MaximumMultiplier`

Formula base do multiplicador:

```text
delta = difficultyPercent - 100
steps = abs(delta) / 10
percent = steps * PercentPer10...
factor = 1 + percent / 100
```

Depois o fator e limitado por:

- `MinimumMultiplier`
- `MaximumMultiplier` se for maior que `0`

### `Enabled`

- Tipo: `bool`
- Liga ou desliga a regra.
- Se `false`, o multiplicador vira `1`.

### `CurveType`

- Tipo: `enum` numerico
- Uso atual nestas regras: declarado, mas nao esta sendo usado no calculo do multiplicador.
- Hoje o scaling destas regras e sempre linear por degraus de `10` de dificuldade.

### `PercentPer10DifficultyAboveNominal`

- Tipo: `float`
- Quanto aumentar por cada `+10` acima de `100`.

Exemplo:

- valor `5`
- dificuldade `130`
- multiplicador bruto `1.15`

### `PercentPer10DifficultyBelowNominal`

- Tipo: `float`
- Quanto reduzir por cada `-10` abaixo de `100`.

Exemplo:

- valor `5`
- dificuldade `80`
- multiplicador bruto `0.90`

### `MinimumMultiplier`

- Tipo: `float`
- Piso do multiplicador final.

### `MaximumMultiplier`

- Tipo: `float`
- Teto do multiplicador final.
- Se for `0` ou menor, nao existe teto maximo.

## `NpcScaling.Health`

Uso atual no codigo:

- a regra e consultada no spawn do NPC
- se o multiplicador calculado for diferente de `1`, o plugin chama `ApplyEliteMeleeScaling`

Cuidado importante:

- no comportamento atual, o fator realmente aplicado na vida nao usa diretamente `PercentPer10DifficultyAboveNominal` e `PercentPer10DifficultyBelowNominal`
- o fator aplicado hoje e `difficultyPercent / 100`, limitado entre `0.4` e `1.8`
- alem da vida, esse metodo tambem tenta escalar alguns campos internos de dano e velocidade de ataque via reflexao

Em outras palavras:

- esta regra hoje funciona mais como gatilho para ligar scaling de vida/ataque conforme a dificuldade muda
- mas a intensidade final nao segue exatamente os percentuais configurados na regra

## `NpcScaling.MeleeDamageToPlayers`

- Aplica multiplicador no dano que NPCs do evento causam aos jogadores.
- So entra quando o role pode atacar jogadores.
- Se `CanTargetPlayers = false`, o dano em player e bloqueado antes.

## `NpcScaling.MeleeDamageToCore`

- Aplica multiplicador no dano causado ao core.
- O dano final e:

```text
CoreDamageBase * multiplicadorDaRegra
```

## Recomendacoes praticas

- Use `WaveMode = 0` para eventos fechados por numero de waves.
- Use `WaveMode = 1` para eventos endless.
- Mantenha `Entries` com `wave 1` repetindo o valor base para deixar o preset mais legivel.
- Use `UnlockDifficultyPercent` em `Roles` e `NpcPools` para introduzir inimigos por fase de progressao.
- Se quiser waves mais curtas quando a dificuldade sobe, use `SecondsPer10DifficultyAboveNominal` negativo.
- Se quiser mais mobs em servidor cheio, ajuste `ExtraSpawnsPerExtraValidPlayer`.
- Se quiser mais mobs quando a dificuldade explode, ajuste `DifficultyScaling`.
- Trate `NpcSelection.PreferHigherUnlockWeight` e `SpawnFlow.MeleeCoreMoveSpeed` como reservados no estado atual.
- Trate `NpcScaling.*.CurveType` como reservado no estado atual.
