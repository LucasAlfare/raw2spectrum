# Formato `.ROLL` (Versão 2)

Gerado pela ferramenta `midi2roll`.

## Pra que serve

Guarda, de forma simplificada, quais notas foram tocadas e quando — extraído de um arquivo MIDI. É o dado de **saída** do treino: é o que queremos que o modelo aprenda a prever a partir do `.SPEC`.

Esse formato não guarda tudo que existe num MIDI (instrumento, força da tecla, pedal, etc). Guarda só o essencial: **qual nota, quando começou, até quando ficou tocando**.

## ⚠️ Importante: isto é um formato intermediário, não o resultado final

**O `.ROLL` não é um arquivo MIDI, e o modelo treinado com este formato nunca vai gerar um `.mid` diretamente.**

Quando você treina um modelo usando `.SPEC` como entrada e `.ROLL` como saída esperada, o que o modelo aprende a produzir é **uma matriz de números** (0, 1 ou 2), no mesmo formato deste arquivo — não um arquivo `.mid`.

Ou seja, o caminho completo, do MP3 novo até o MIDI final, tem uma etapa a mais depois do modelo:

```
MP3 novo
   ↓ (gera o espectro)
.SPEC
   ↓ (modelo treinado faz a previsão)
matriz .ROLL prevista   ← é AQUI que o modelo para. Isto NÃO é um MIDI ainda.
   ↓ (passo de conversão — ver seção "Convertendo a saída do modelo em MIDI", mais abaixo)
arquivo .mid final
```

Esse passo de conversão é obrigatório e precisa ser feito por fora do modelo, depois que ele termina de prever. Sem esse passo, você fica só com uma matriz de números — não é possível abrir isso num player de música ou numa DAW.

A boa notícia: esse passo de conversão é simples, direto, e não tem perda de informação essencial no meio do caminho — a matriz `.ROLL` já contém tudo que é necessário (pitch, início, fim de cada nota) para montar um MIDI correto. É diferente do que acontece com o `.SPEC`: lá, informação necessária pra reconstruir o áudio (a fase da onda) foi descartada de propósito e não tem como recuperar. Aqui, nada essencial foi perdido — só falta reorganizar os dados no formato de arquivo `.mid`, que é uma tarefa mecânica, coberta na seção abaixo.

O que o `.mid` final **não** vai ter, porque foram descartados de propósito na hora de gerar o `.ROLL`: instrumento, força da tecla (velocity), pedal de sustain. O MIDI final terá as notas certas, no tempo certo — só não terá a "textura" de interpretação do MIDI original usado no treino.

## Ideia central

Pega cada nota do MIDI e simplifica ela pra 3 informações: qual tecla, em que momento começou a tocar, em que momento parou. Junta isso numa tabela parecida com a do `.SPEC` (mesmas colunas de tempo, uma coluna por tecla do piano), onde cada célula guarda um destes três valores:

```
0 = silêncio (nenhuma nota tocando)
1 = a nota começou exatamente agora (ataque)
2 = a nota continua tocando (mas não é o começo)
```

Exemplo, olhando só a coluna da tecla "Dó central":

```
tempo:     0    1    2    3    4    5    6
valor:     0    1    2    2    0    1    2
                ↑ nota começa      ↑ nota começa de novo
```

O valor `1` sempre marca o instante exato do ataque — isso é importante mesmo em uma matriz só, porque se duas notas iguais tocarem coladas (sem silêncio entre elas), o `1` ainda avisa "começou uma nova aqui", em vez de parecer uma nota só, mais longa.

---

## Estrutura do arquivo

```
+------------+
| Cabeçalho  |
+------------+
| Tabela     |
+------------+
```

## Cabeçalho — byte a byte

| Posição (offset) | Tamanho | O que é |
|---|---|---|
| 0 | 4 bytes | Letras "ROLL" (identifica o arquivo) |
| 4 | 1 byte | Número da versão do formato (`2`) |
| 5 | 4 bytes | Quantas colunas de tempo a tabela tem |
| 9 | 4 bytes | Sample rate (mesmo valor do `.SPEC` correspondente) |
| 13 | 4 bytes | Hop size (mesmo valor do `.SPEC` correspondente) |

Total do cabeçalho: **17 bytes**

### Sobre as teclas representadas

A tabela sempre cobre as 88 teclas de um piano (da nota 21 até a nota 108, numeração MIDI padrão). Esse intervalo é fixo e não muda de arquivo pra arquivo, por isso não precisa ser gravado no cabeçalho — é uma regra combinada do projeto.

### Sobre sample rate e hop size

Servem só como conferência: garantem que este `.ROLL` foi construído usando a mesma escala de tempo do `.SPEC` correspondente. Antes de usar os dois arquivos juntos no treino, vale checar se esses dois números batem entre eles.

---

## A tabela de dados

Depois do cabeçalho (a partir do byte 17), vem a tabela, uma coluna de tempo por vez, e dentro dela, uma tecla por vez:

```
tempo 0: tecla 21, tecla 22, ..., tecla 108
tempo 1: tecla 21, tecla 22, ..., tecla 108
...
```

Cada valor ocupa **1 byte** (só precisa guardar 0, 1 ou 2).

---

## Preparando os dados do MIDI antes de gravar

Antes de virar `.ROLL`, os dados do MIDI passam por uma limpeza simples:

- Ignora faixas de bateria
- Ignora instrumento, força da tecla (velocity), pedal e qualquer outro detalhe — fica só pitch, início e fim de cada nota
- Se várias notas tocam em faixas diferentes ao mesmo tempo, todas entram juntas na mesma tabela

---

## Como escrever um arquivo `.ROLL`

Você precisa ter em mãos: a lista de notas já limpa (pitch, início em segundos, fim em segundos), o sample rate e o hop size do `.SPEC` correspondente, e quantas colunas de tempo esse `.SPEC` tem.

```
função escrever_roll(notas, sample_rate, hop_size, total_colunas):

    cria uma tabela vazia [total_colunas][88], tudo preenchido com 0

    para cada nota (pitch, inicio, fim) na lista de notas:

        se pitch for menor que 21 ou maior que 108:
            ignora essa nota

        coluna_da_tecla = pitch - 21

        frame_inicio = arredonda(inicio * sample_rate / hop_size)
        frame_fim    = arredonda(fim * sample_rate / hop_size)

        tabela[frame_inicio][coluna_da_tecla] = 1

        para frame de (frame_inicio + 1) até (frame_fim - 1):
            tabela[frame][coluna_da_tecla] = 2

    escreve "ROLL"
    escreve versão = 2
    escreve total_colunas
    escreve sample_rate
    escreve hop_size

    para cada linha de tempo da tabela:
        para cada uma das 88 teclas:
            escreve o valor (0, 1 ou 2) como 1 byte
```

## Como ler um arquivo `.ROLL`

```
função ler_roll(arquivo):

    lê 4 bytes, confirma que é "ROLL"
    lê versão
    lê total_colunas
    lê sample_rate
    lê hop_size

    cria uma tabela vazia [total_colunas][88]

    para cada linha de tempo:
        para cada uma das 88 teclas:
            lê 1 byte
            guarda na tabela

    devolve tabela, sample_rate, hop_size
```

---

## Convertendo a saída do modelo em MIDI

Depois que o modelo já foi treinado e você passa um MP3 novo por ele, o resultado é uma matriz no formato `.ROLL` (valores 0/1/2). Esta seção explica como transformar essa matriz numa lista de notas, e depois num arquivo `.mid` de verdade.

### Passo 1 — matriz vira lista de notas

Percorre a matriz procurando, para cada tecla, blocos contínuos que começam com `1` (onset) e seguem com `2` (sustain), até voltar pra `0` (silêncio) ou até aparecer um novo `1` (nova nota, sem silêncio no meio).

```
função matriz_para_notas(tabela, sample_rate, hop_size):

    notas = lista vazia
    nota_ativa = {}  // guarda, por tecla, em que frame a nota atual começou

    para cada frame (linha) da tabela:
        para cada tecla (coluna) da tabela:

            valor = tabela[frame][tecla]

            se valor == 1:
                // uma nota nova está começando
                se tecla já estava em nota_ativa:
                    // a nota anterior nessa tecla termina aqui, uma nova começa
                    fecha_nota(notas, nota_ativa, tecla, frame_atual=frame, sample_rate, hop_size)
                nota_ativa[tecla] = frame

            senão se valor == 0 e tecla está em nota_ativa:
                // a nota que estava tocando parou
                fecha_nota(notas, nota_ativa, tecla, frame_atual=frame, sample_rate, hop_size)

    // fecha qualquer nota que ainda estava tocando no último frame
    para cada tecla restante em nota_ativa:
        fecha_nota(notas, nota_ativa, tecla, frame_atual=total_de_frames, sample_rate, hop_size)

    devolve notas


função fecha_nota(notas, nota_ativa, tecla, frame_atual, sample_rate, hop_size):

    frame_inicio = nota_ativa[tecla]

    inicio_segundos = frame_inicio * hop_size / sample_rate
    fim_segundos = frame_atual * hop_size / sample_rate
    pitch = tecla + 21

    notas.adiciona( (pitch, inicio_segundos, fim_segundos) )

    remove tecla de nota_ativa
```

O resultado é uma lista simples: `[(pitch, início_em_segundos, fim_em_segundos), ...]`.

### Passo 2 — lista de notas vira arquivo `.mid`

Isso é feito com uma biblioteca pronta (`pretty_midi`, em Python), sem necessidade de lidar com bytes manualmente:

```
importa pretty_midi

midi = novo PrettyMIDI()
instrumento = novo Instrument(programa=0)  // 0 = piano acústico

para cada (pitch, inicio, fim) na lista de notas:
    instrumento.notas.adiciona(
        Note(velocity=100, pitch=pitch, start=inicio, end=fim)
    )
    // velocity fixa em 100 porque o modelo não prevê força da tecla — 
    // é só um valor padrão pra o arquivo MIDI ser válido

midi.instrumentos.adiciona(instrumento)
midi.salva("resultado_final.mid")
```

Pronto — esse é o arquivo `.mid` que o usuário final recebe, com as notas na hora certa.

---

## Resultado final esperado

**Do formato `.ROLL` em si:** uma matriz numérica (array 2D, valores 0/1/2) representando quais notas soam em cada instante, alinhada exatamente com a mesma linha do tempo do `.SPEC` correspondente. É o alvo que o modelo tenta prever durante o treino.

**Do pipeline completo (MP3 novo → resultado):** um arquivo `.mid` de verdade, com as notas certas nos tempos certos, gerado em duas etapas — primeiro o modelo prevê a matriz `.ROLL`, depois essa matriz é convertida em MIDI pelo processo descrito acima. O `.ROLL` sozinho, produzido pelo modelo, **não é o entregável final** — é uma etapa intermediária necessária antes de chegar no arquivo `.mid`.
