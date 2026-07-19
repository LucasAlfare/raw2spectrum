# Formato `.ROLL` (Versão 3)

Gerado pela ferramenta `midi2roll`.

---

# Parte 1 — Visão geral: como sanitizar um MIDI antes de virar `.ROLL`

Antes de chegar na especificação binária, vale entender o que precisa acontecer com o MIDI cru até ele virar dado limpo o suficiente pra treino. Essa etapa é tão importante quanto o formato do arquivo em si — um `.ROLL` gerado de um MIDI mal separado vai ensinar coisa errada pro modelo, mesmo que o arquivo esteja tecnicamente correto.

## 1. Separar as faixas em categorias

Um MIDI é organizado em faixas (tracks), e cada faixa tem duas informações que dizem "o que ela é":

- Um número de instrumento (`program number`, de 0 a 127, padrão General MIDI)
- Uma marcação de "isso é bateria" (`is_drum`, verdadeiro ou falso)

Com isso, toda faixa cai em uma de três categorias:

| Categoria | Como identificar |
|---|---|
| **Percussão** | `is_drum = verdadeiro` |
| **Baixo** | `is_drum = falso` e `program number` entre 32 e 39 |
| **Acompanhamento** | `is_drum = falso` e qualquer outro `program number` (tudo que não é baixo nem bateria: piano, cordas, sopro, melodia, etc) |

Isso é uma escolha de projeto — "acompanhamento" vira um balaio único com tudo que sobra. Se no futuro for necessário separar melodia principal de harmonia de fundo, precisa de mais categorias e mais regras (o `program number` sozinho não resolve essa distinção).

## 2. Juntar faixas da mesma categoria

Uma música pode ter mais de uma faixa de "acompanhamento" (ex: piano e cordas, ambos não-baixo, não-bateria). Nesse caso, todas as notas de todas as faixas dessa categoria são despejadas juntas numa única lista, como se fosse um instrumento só. O mesmo vale se houver mais de uma faixa de bateria.

## 3. Reduzir os sons de percussão a um conjunto fixo

Bateria no MIDI não representa "nota grave/aguda" como no piano — cada número de pitch é um **som diferente** (bumbo, caixa, chimbal, etc), sem relação de escala musical entre eles. O padrão General MIDI define uns 47 sons possíveis, o que é demais e redundante pro que precisamos (muitos são variações do mesmo som, tipo 3 tipos de chimbal aberto).

Reduz-se isso pra um conjunto fixo de **sons canônicos**, agrupando variações parecidas:

| Som canônico | Pitches do GM agrupados nele |
|---|---|
| Bumbo (kick) | 35, 36 |
| Caixa (snare) | 38, 40 |
| Chimbal fechado | 42, 44 |
| Chimbal aberto | 46 |
| Tom grave | 41, 43, 45 |
| Tom médio | 47, 48 |
| Tom agudo | 50 |
| Prato de ataque (crash) | 49, 57 |
| Prato de condução (ride) | 51, 59 |
| Pandeiro/shaker | 54, 70, 82 |
| Palma (clap) | 39 |
| Outro/outros | qualquer pitch de percussão não listado acima |

Isso é uma tabela de referência — pode ser ajustada, mas precisa existir e ser fixa, porque é ela que define quantas colunas a camada de percussão do `.ROLL` vai ter (nesse exemplo, 11 sons canônicos).

## 4. Encaixar tudo no mesmo grid de tempo

As três categorias (baixo, acompanhamento, percussão) usam exatamente a mesma divisão de tempo (mesma fórmula de `sample_rate`/`hop_size` já usada no `.SPEC`). Isso garante que, ao olhar a coluna de tempo N em qualquer uma das três camadas, todas estão falando do mesmo instante exato da música.

## 5. Limpezas finais, comuns às três categorias

- Descarta notas com pitch fora do intervalo esperado (21–108 pra baixo/acompanhamento; fora da tabela de sons canônicos pra percussão)
- Descarta notas de duração zero (erros de exportação do MIDI às vezes geram isso)
- Se duas notas idênticas (mesmo pitch, mesma categoria) tocarem coladas sem silêncio entre elas, cada uma gera seu próprio "começo" (`1`) na matriz — isso já é resolvido pela lógica onset/sustain que a spec usa, então não precisa de tratamento especial extra

Depois dessas 5 etapas, o MIDI está pronto para ser convertido no arquivo `.ROLL` descrito a seguir.

---

# Parte 2 — Especificação do formato

## Pra que serve

Guarda, numa única estrutura, o que cada uma das três partes de uma música está tocando: **baixo**, **acompanhamento** (o resto dos instrumentos melódicos/harmônicos) e **percussão**. É o dado de **saída** do treino — o que se espera que o modelo aprenda a prever a partir do `.SPEC`.

## ⚠️ Isto é um formato intermediário, não o resultado final

O modelo treinado com este formato nunca gera um `.mid` diretamente. Ele aprende a prever três matrizes de números. Virar um arquivo `.mid` de verdade exige um passo extra depois da previsão (explicado na Parte 4).

```
MP3 novo
   ↓ (gera o espectro)
.SPEC
   ↓ (modelo treinado faz a previsão)
3 matrizes .ROLL previstas (baixo, acompanhamento, percussão)  ← o modelo para aqui
   ↓ (passo de conversão — Parte 4)
arquivo .mid final, com 3 faixas
```

Nada essencial se perde nesse processo (diferente do `.SPEC`, que abre mão da fase do som de propósito e não tem volta) — só falta reorganizar os dados no formato de arquivo `.mid`, tarefa mecânica coberta adiante.

O `.mid` final não terá: força da tecla (velocity) real, pedal, nem o instrumento exato original de cada faixa (usa-se um instrumento padrão por categoria). Terá as notas certas, nas categorias certas, nos tempos certos.

## Ideia central

Em vez de uma tabela só, o `.ROLL` agora guarda **três tabelas**, uma por categoria, todas alinhadas na mesma linha do tempo:

```
Tabela de Baixo:          [tempo] x [88 teclas]
Tabela de Acompanhamento:  [tempo] x [88 teclas]
Tabela de Percussão:       [tempo] x [11 sons canônicos]
```

Cada célula usa os mesmos três valores de antes:

```
0 = silêncio
1 = a nota/som começou exatamente agora
2 = a nota/som continua tocando
```

---

## Estrutura do arquivo

```
+---------------------+
| Cabeçalho           |
+---------------------+
| Tabela de Baixo     |
+---------------------+
| Tabela de Acompanhamento |
+---------------------+
| Tabela de Percussão |
+---------------------+
```

## Cabeçalho — byte a byte

| Posição (offset) | Tamanho | O que é |
|---|---|---|
| 0 | 4 bytes | Letras "ROLL" (identifica o arquivo) |
| 4 | 1 byte | Número da versão do formato (`3`) |
| 5 | 4 bytes | Quantas colunas de tempo as tabelas têm (as três têm o mesmo número) |
| 9 | 4 bytes | Sample rate (mesmo valor do `.SPEC` correspondente) |
| 13 | 4 bytes | Hop size (mesmo valor do `.SPEC` correspondente) |
| 17 | 1 byte | Quantos sons canônicos de percussão existem (no exemplo desta spec, `11`) |

Total do cabeçalho: **18 bytes**

### Por que percussão tem seu próprio contador de colunas

Baixo e acompanhamento sempre usam 88 colunas (as 88 teclas de piano, valor fixo, não precisa gravar). Percussão usa um número de colunas que depende da tabela de sons canônicos escolhida (Parte 1, item 3) — por isso esse número fica gravado no cabeçalho, pra qualquer leitor saber o tamanho exato da terceira tabela sem precisar adivinhar.

---

## As três tabelas de dados

Aparecem nesta ordem fixa: baixo, depois acompanhamento, depois percussão. Cada uma segue o mesmo padrão — tempo por tempo, e dentro de cada tempo, uma coluna por vez:

```
Tabela de baixo (88 colunas):
  tempo 0: tecla 21, tecla 22, ..., tecla 108
  tempo 1: tecla 21, tecla 22, ..., tecla 108
  ...

Tabela de acompanhamento (88 colunas):
  (mesmo padrão acima)

Tabela de percussão (N colunas, N = valor do cabeçalho):
  tempo 0: som 0, som 1, ..., som N-1
  tempo 1: som 0, som 1, ..., som N-1
  ...
```

Cada valor ocupa **1 byte** (0, 1 ou 2).

---

# Parte 3 — Como escrever e como ler

## Como escrever um arquivo `.ROLL`

Você precisa ter em mãos: as notas já separadas e limpas nas três categorias (Parte 1), o sample rate e hop size do `.SPEC` correspondente, o total de colunas de tempo, e a tabela de sons canônicos de percussão escolhida.

```
função escrever_roll(notas_baixo, notas_acompanhamento, notas_percussao,
                      sample_rate, hop_size, total_colunas, tabela_sons_percussao):

    tabela_baixo = monta_tabela(notas_baixo, total_colunas, 88, pitch_minimo=21)
    tabela_acompanhamento = monta_tabela(notas_acompanhamento, total_colunas, 88, pitch_minimo=21)
    tabela_percussao = monta_tabela_percussao(notas_percussao, total_colunas, tabela_sons_percussao)

    escreve "ROLL"
    escreve versão = 3
    escreve total_colunas
    escreve sample_rate
    escreve hop_size
    escreve quantidade_de_sons_percussao = tamanho de tabela_sons_percussao

    escreve tabela_baixo (linha por linha, cada valor em 1 byte)
    escreve tabela_acompanhamento (linha por linha, cada valor em 1 byte)
    escreve tabela_percussao (linha por linha, cada valor em 1 byte)


função monta_tabela(notas, total_colunas, n_colunas_de_saida, pitch_minimo):

    cria tabela vazia [total_colunas][n_colunas_de_saida], tudo em 0

    para cada nota (pitch, inicio, fim) em notas:

        se pitch fora do intervalo válido:
            ignora essa nota

        coluna = pitch - pitch_minimo
        frame_inicio = arredonda(inicio * sample_rate / hop_size)
        frame_fim = arredonda(fim * sample_rate / hop_size)

        tabela[frame_inicio][coluna] = 1
        para frame de (frame_inicio + 1) até (frame_fim - 1):
            tabela[frame][coluna] = 2

    devolve tabela


função monta_tabela_percussao(notas, total_colunas, tabela_sons_percussao):

    cria tabela vazia [total_colunas][tamanho de tabela_sons_percussao], tudo em 0

    para cada nota (pitch_original, inicio, fim) em notas:

        som_canonico = busca pitch_original em tabela_sons_percussao

        se não encontrou:
            ignora essa nota

        // sons de percussão costumam ser curtos e sem sustain longo,
        // mas seguimos a mesma lógica onset/sustain por consistência
        frame_inicio = arredonda(inicio * sample_rate / hop_size)
        frame_fim = arredonda(fim * sample_rate / hop_size)

        tabela[frame_inicio][som_canonico] = 1
        para frame de (frame_inicio + 1) até (frame_fim - 1):
            tabela[frame][som_canonico] = 2

    devolve tabela
```

## Como ler um arquivo `.ROLL`

```
função ler_roll(arquivo):

    lê 4 bytes, confirma que é "ROLL"
    lê versão
    lê total_colunas
    lê sample_rate
    lê hop_size
    lê quantidade_de_sons_percussao

    tabela_baixo = lê_tabela(total_colunas, 88)
    tabela_acompanhamento = lê_tabela(total_colunas, 88)
    tabela_percussao = lê_tabela(total_colunas, quantidade_de_sons_percussao)

    devolve tabela_baixo, tabela_acompanhamento, tabela_percussao, sample_rate, hop_size


função lê_tabela(n_linhas, n_colunas):

    cria tabela vazia [n_linhas][n_colunas]

    para cada linha:
        para cada coluna:
            lê 1 byte
            guarda na tabela

    devolve tabela
```

---

# Parte 4 — Convertendo a saída do modelo em MIDI

Depois que o modelo prevê as três matrizes a partir de um MP3 novo, cada uma delas passa pelo mesmo processo de decodificação (matriz → lista de notas), igual ao usado numa única tabela:

```
função matriz_para_notas(tabela, sample_rate, hop_size, deslocamento_de_pitch):

    notas = lista vazia
    nota_ativa = {}

    para cada frame (linha) da tabela:
        para cada coluna da tabela:

            valor = tabela[frame][coluna]

            se valor == 1:
                se coluna já estava em nota_ativa:
                    fecha_nota(notas, nota_ativa, coluna, frame, sample_rate, hop_size, deslocamento_de_pitch)
                nota_ativa[coluna] = frame

            senão se valor == 0 e coluna está em nota_ativa:
                fecha_nota(notas, nota_ativa, coluna, frame, sample_rate, hop_size, deslocamento_de_pitch)

    para cada coluna restante em nota_ativa:
        fecha_nota(notas, nota_ativa, coluna, total_de_frames, sample_rate, hop_size, deslocamento_de_pitch)

    devolve notas
```

Para baixo e acompanhamento, `deslocamento_de_pitch = 21` (soma-se à coluna pra recuperar o pitch original de piano). Para percussão, em vez de somar um deslocamento, usa-se a tabela de sons canônicos ao contrário, pra transformar "coluna 3" de volta em, por exemplo, "pitch 38 (caixa)" — pega-se um pitch representante de cada grupo (o primeiro da lista da Parte 1, item 3).

Depois de decodificadas as três listas de notas, monta-se o `.mid` com três faixas:

```
importa pretty_midi

midi = novo PrettyMIDI()

faixa_baixo = novo Instrument(programa=33)          // baixo elétrico, dedo
faixa_acompanhamento = novo Instrument(programa=0)  // piano acústico
faixa_percussao = novo Instrument(programa=0, is_drum=verdadeiro)

para cada (pitch, inicio, fim) em notas_baixo:
    faixa_baixo.notas.adiciona(Note(velocity=100, pitch=pitch, start=inicio, end=fim))

para cada (pitch, inicio, fim) em notas_acompanhamento:
    faixa_acompanhamento.notas.adiciona(Note(velocity=100, pitch=pitch, start=inicio, end=fim))

para cada (pitch_representante, inicio, fim) em notas_percussao:
    faixa_percussao.notas.adiciona(Note(velocity=100, pitch=pitch_representante, start=inicio, end=fim))

midi.instrumentos.adiciona(faixa_baixo)
midi.instrumentos.adiciona(faixa_acompanhamento)
midi.instrumentos.adiciona(faixa_percussao)

midi.salva("resultado_final.mid")
```

`velocity=100` é um valor fixo padrão — o modelo não prevê força da tecla, então usa-se um valor médio constante pra o arquivo MIDI ser válido e soar de forma equilibrada.

---

## Resultado final esperado

**Do formato `.ROLL` em si:** três matrizes numéricas (valores 0/1/2), uma para baixo, uma para acompanhamento e uma para percussão, todas alinhadas ao mesmo eixo de tempo do `.SPEC` correspondente. É o alvo que o modelo tenta prever durante o treino.

**Do pipeline completo (MP3 novo → resultado):** um arquivo `.mid` com três faixas separadas (baixo, acompanhamento, percussão), cada uma com notas nos tempos certos. O `.ROLL` sozinho, produzido pelo modelo, não é o entregável final — é uma etapa intermediária necessária antes de chegar no arquivo `.mid` completo.
