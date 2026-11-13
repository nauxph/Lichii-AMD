<!--
# SPDX-FileCopyrightText: Copyright 2026 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later
-->

# Lichii-AMD

Lichii-AMD is [Lichii](https://github.com/zortos293/Lichii) with fixes for **AMD Radeon** GPUs, tested with
**The Legend of Zelda: Ocarina of Time** for Switch 2 (`0400486024DEC000`). It is built on Lichii's `nx2`
branch at `8ddb4b9a38` plus two later fixes (Lichii #1576 and #1592), which in turn builds on Eglantine,
Eden, yuzu and Sudachi. This fork is not affiliated with Nintendo, AMD or the upstream projects.

*Português abaixo.*

## Download and setup

Download the zip from the [releases page](https://github.com/nauxph/Lichii-AMD/releases), extract all
of it and run `lichii.exe`.

You need:

- Windows 10 or 11 (64-bit) and an AMD Radeon GPU with a current driver. This build was tested on an
  RX 9070 XT with driver 2.0.395.
- The latest **Microsoft Visual C++ Redistributable (x64)**:
  <https://aka.ms/vc14/vc_redist.x64.exe>. The build uses MSVC 14.51, so an older runtime can stop
  it from starting.
- Your own keys, firmware and games, dumped from your own console. None of them are included.

For Ocarina of Time, set **GPU Mode** to **Fast** in the graphics settings, either globally or for the
game. All measurements below used Fast.

### Coming from an earlier version

This version keeps its data in `%APPDATA%\lichii` instead of `%APPDATA%\eden`. On the first launch a
**Migration** window offers to import the old data:

1. Close any other copy of the emulator.
2. Leave **Keep Old Data** selected. It copies, so the `eden` folder stays as it was and older
   versions keep working.
3. Leave **Clear Shader Cache** unticked, so your shader cache comes along.
4. Pick **Eden** in the list and click **Import**.

From then on this version and older ones keep separate saves.

The first launch converts the shader cache to a new format and compiles Ocarina's shaders again (the
driver's cache does not carry over), so loading takes longer and the game stutters for a while in
each new area. **Build all cached shaders before game start** (Configure > Graphics > Advanced) builds
every cached shader on the loading screen instead; loading then takes longer and uses more memory.

Shader stutter needs no extra setting: while a very large shader compiles for the first time, its
draws are skipped instead of freezing the game. Asynchronous shader building can stay off. To turn
the skipping off, put `EDEN_NX2_HEAVY_SKIP=0` in `nx2_switches.txt` (see Switches).

## Changes from Lichii

The commits are on the `amd` branch.

- **Shader stutter in new places.** While a shader of 4,000 or more instructions compiles for the
  first time, its draws are skipped unless that could lose data, and its build moves ahead of the
  background work. This is the heavy skip from Lichii #579, on by default here and also without
  asynchronous shaders. Draws held per new pipeline went from 111-124 ms to 2-20 ms, and the worst
  freeze from 7-11 s to under 1 s, with no missing objects seen. A large object can appear a moment
  late in a new place.
- **A draw waiting for its pipeline builds it right away** instead of waiting behind the whole queue
  (port of Lichii #321, now on upstream's shader build scheduler).
- **Occlusion query copies (frame rate).** On AMD drivers, the occlusion query results the game reads
  are copied after one barrier instead of waiting on each query (`VK_QUERY_RESULT_WAIT_BIT`). AMD's
  driver spent about 22 µs on each wait. With Ocarina's ~2,000 queries a frame, that came to 45 ms of
  a 61 ms GPU frame; the copies now take 0.4 ms.

  Measured at the Kokiri village sign (RX 9070 XT, docked, 1440p), in the same scene and session,
  over three 600-frame windows each:

  | | Before | After |
  |---|---|---|
  | Frame rate | 19.7 fps | 34.7 fps |
  | p95 frame time | 55.9 ms | 34.2 ms |
  | p99 frame time | 66.0 ms | 42.0 ms |
  | 1% low | 15.2 fps | 23.9 fps |

- **Flickering HUD, minimap and dialogs.** A full memory barrier runs around every render pass
  (`EDEN_NX2_RP_BARRIER=1`, on by default in this build). It costs about 0.05 ms a frame.
- **Diagnostics.** GPU time per recording line (`EDEN_DKB_FRAME_TRACE=1`), frame time percentiles in
  the log, and live `rp_barrier` switching on the debug server (`docs/nx2/debug-mcp.md`).

### Lichii `nx2` up to `8ddb4b9a38`

This version moves the base from `65e979adb3` to `8ddb4b9a38` (220 commits). Among the changes:

- #1253 moves the data folder to `%APPDATA%\lichii`, with the Migration window above.
- #1036 replaces the shader cache with a new format and builds shaders on a prioritized scheduler.
- #1334 and #1337 keep pipelines up to a memory limit and rebuild them from kept SPIR-V. They replace
  this fork's port of #313.
- #949 (semaphores without draining the GPU) and #1028 and #1056 (macros on the GPU split's back)
  replace this fork's ports of #204, #215 and #243.
- #659 removed the throttle on game file reads, so `EDEN_NX2_FS_READ_MBPS=0` is no longer set.
- #1060 and #1062 add AMD FSR 3.1 and FSR 4 for Ocarina.
- Ocarina fixes: #1485 (water grid), #1313 (freeze at the guard house), #1196 (ocarina crackle) and
  #820 (line in the tomb).
- #818 skips stale texture uploads.
- Added on top: #1576 waits for an emulation teardown that still makes progress instead of
  terminating it (in this version's first test builds, closing the game hung for 10 s and then
  crashed), and #1592 fixes a crash when another game starts after Ocarina was stopped.

Fixes from earlier releases are still in: green sky, fog and water (#351); black water and foliage,
then a yellow-white image (#432); Ocarina's image filter on the GPU's matrix units (#583); the Hyrule
Field horizon blocks (#353); movies in sync with their sound (#251); and #550, #397, #446, #410, #580,
#584 and #585.

## Switches

The `nx2_switches.txt` next to `lichii.exe` sets this build's defaults: the GPU split, fence
batching, 128-draw submissions and the render pass barrier. To change one, copy its line into
`%APPDATA%\lichii\nx2_switches.txt`, which wins over the file next to the executable, then restart.
`EDEN_NX2_HEAVY_SKIP=0` there turns off the skipping of heavy shaders.

## Known issues

- This is a pre-release with a much newer upstream base. Only Ocarina of Time on one PC was tested
  with it, for about four hours.
- The frame rate moves between about 40 and 60 fps while walking. It is limited by the round trips
  between the emulated CPU and the GPU emulation threads, not by the graphics card (the RX 9070 XT
  is about 13% busy) or by one saturated thread.
- Loading a new area can pause the game for a few seconds.
- `EDEN_NX2_PREFLUSH=1` corrupts water and grass on AMD.

## License and source

Lichii-AMD is GPL-3.0-or-later; see `LICENSE.txt` and `LICENSES/`. The source for each release is
its tagged commit on the `amd` branch of this repository.

---

## Português

O Lichii-AMD é o [Lichii](https://github.com/zortos293/Lichii) com correções para placas **AMD Radeon**,
testado com **The Legend of Zelda: Ocarina of Time** de Switch 2 (`0400486024DEC000`). Ele parte do
branch `nx2` do Lichii no commit `8ddb4b9a38`, mais duas correções posteriores (PRs #1576 e #1592), e o
Lichii por sua vez vem do Eglantine, do Eden, do yuzu e do Sudachi. Não tem ligação com a Nintendo, com
a AMD nem com os projetos originais.

### Como usar

Baixe o zip na [página de releases](https://github.com/nauxph/Lichii-AMD/releases), extraia tudo e
abra o `lichii.exe`.

Você precisa de:

- Windows 10 ou 11 (64 bits) e uma placa AMD Radeon com driver atualizado. Testado numa RX 9070 XT
  com o driver 2.0.395.
- O **Microsoft Visual C++ Redistributable (x64)** mais recente:
  <https://aka.ms/vc14/vc_redist.x64.exe>. Sem ele atualizado, o emulador pode nem abrir.
- Suas próprias chaves, firmware e jogos, extraídos do seu console. Nada disso vem incluído.

No Ocarina of Time, deixe o **GPU Mode** em **Fast** nas opções de gráficos, globais ou do jogo.

### Vindo de uma versão anterior

Esta versão guarda os dados em `%APPDATA%\lichii`, e não mais em `%APPDATA%\eden`. Na primeira vez
que abrir, aparece a janela **Migration** para importar os dados antigos:

1. Feche qualquer outra cópia do emulador.
2. Deixe **Keep Old Data** marcado. Ele copia, então a pasta `eden` fica como estava e as versões
   antigas continuam funcionando.
3. Deixe **Clear Shader Cache** desmarcado, para o cache de shaders vir junto.
4. Escolha **Eden** na lista e clique em **Import**.

Daí em diante, esta versão e as antigas têm saves separados.

Na primeira vez, o cache de shaders é convertido para o formato novo e os shaders do Ocarina são
compilados de novo (o cache do driver não aproveita os antigos), então o carregamento demora mais e o
jogo engasga um pouco em cada área nova. A opção **Build all cached shaders before game start**
(Emulação > Configurar > Gráficos > Avançado) compila todos os shaders do cache na tela de
carregamento; o carregamento fica mais longo e usa mais memória.

As travadas de shader não precisam de nenhum ajuste: enquanto um shader muito pesado compila pela
primeira vez, os desenhos dele são pulados em vez de travar o jogo. A compilação assíncrona pode
ficar desligada. Para desligar o pulo, coloque `EDEN_NX2_HEAVY_SKIP=0` no `nx2_switches.txt`.

### O que muda em relação ao Lichii

- **Travadas de shader em lugares novos:** enquanto um shader de 4.000 instruções ou mais compila pela
  primeira vez, os desenhos dele são pulados, se isso não perder dados, e a compilação dele passa na
  frente do trabalho em segundo plano (PR #579, ligado por padrão aqui). A espera por pipeline novo
  caiu de 111–124 ms para 2–20 ms, e a pior travada de 7–11 s para menos de 1 s, sem objetos
  faltando. Um objeto grande pode aparecer um instante depois num lugar novo.
- **O desenho que espera um pipeline compila esse pipeline na hora** (port da PR #321).
- **Cópia das queries de oclusão:** nas placas AMD, os resultados são copiados depois de uma única
  barreira, sem esperar query por query. Na mesma cena, o FPS passou de 19,7 para 34,7, o p95 de
  55,9 para 34,2 ms e o p99 de 66,0 para 42,0 ms.
- **HUD, minimapa e diálogos piscando:** corrigidos com uma barreira em volta de cada passe de
  renderização (`EDEN_NX2_RP_BARRIER=1`), que custa ~0,05 ms por quadro.

### Novidades do Lichii (`nx2` até `8ddb4b9a38`)

- Pasta de dados nova em `%APPDATA%\lichii`, com a janela de migração (PR #1253).
- Cache de shaders num formato novo e compilação com prioridades (PR #1036).
- Os pipelines ficam guardados até um limite de memória e são recompilados a partir do SPIR-V
  guardado (PRs #1334 e #1337), no lugar do port da PR #313.
- Semáforos sem esvaziar a GPU (PR #949) e macros executadas no fundo da GPU dividida (PRs #1028 e
  #1056), no lugar dos ports das PRs #204, #215 e #243.
- Sem limite de velocidade na leitura dos arquivos do jogo (PR #659), por isso o
  `EDEN_NX2_FS_READ_MBPS=0` saiu.
- AMD FSR 3.1 e FSR 4 no Ocarina (PRs #1060 e #1062).
- Correções do Ocarina: grade na água (#1485), travamento na casa da guarda (#1313), chiado da ocarina
  (#1196) e linha na tumba (#820).
- Envio de texturas desatualizadas pulado (PR #818).
- Adicionadas por cima: a PR #1576 espera o encerramento da emulação enquanto ele avança, em vez de
  matá-lo (nas primeiras builds de teste desta versão, fechar o jogo travava 10 s e depois dava erro),
  e a PR #1592 corrige um erro ao abrir outro jogo depois de parar o Ocarina.

Continuam valendo as correções das versões anteriores: céu, névoa e água verdes (#351); água e plantas
pretas e a imagem amarelada (#432); o filtro de imagem do Ocarina nas matrizes da placa (#583); os
blocos coloridos no horizonte de Hyrule Field (#353); e os vídeos em sincronia com o som (#251).

Deixe `EDEN_NX2_PREFLUSH` desligado: na AMD ele deixa a água e a grama pretas.

### Problemas conhecidos

- Esta é uma versão de teste (pré-lançamento), com uma base do Lichii bem mais nova. Só o Ocarina of
  Time foi testado, num único PC, por cerca de quatro horas.
- Andando, o FPS varia entre ~40 e 60. O limite são as idas e voltas entre a CPU emulada e as threads
  que emulam a GPU, não a placa de vídeo (a RX 9070 XT fica em ~13% de uso).
- Carregar uma área nova pode pausar o jogo por alguns segundos.

### Licença

GPL-3.0-or-later; veja `LICENSE.txt` e `LICENSES/`. O código de cada release é o commit marcado com a
tag dele no branch `amd`.
