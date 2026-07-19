# Formato `.SPEC` (Versão 3)

Gerado pela ferramenta `raw2spectrum`.

## Pra que serve

Guarda o "retrato sonoro" de um áudio ao longo do tempo — quanta energia cada frequência tem, em cada instante. É o dado de **entrada** do treino: é o que o modelo vai olhar para tentar adivinhar quais notas foram tocadas.

Esse formato não serve para reconstruir o áudio original de volta, e está tudo bem, porque não é essa a intenção. Ele só precisa ser bom o suficiente para o modelo aprender.

## Ideia central

Imagina uma tabela: cada linha é uma frequência (grave até agudo), cada coluna é um pedacinho de tempo. Cada célula da tabela diz "quanta energia essa frequência tinha nesse momento". Isso é o espectrograma.

```
           tempo 0   tempo 1   tempo 2  ...
grave         0.1       0.3       0.0
médio         0.8       0.9       0.7
agudo         0.0       0.1       0.2
```

O arquivo `.SPEC` é essa tabela, salva em bytes.

---

## Estrutura do arquivo

```
+------------+
| Cabeçalho  |   (informações sobre a tabela)
+------------+
| Tabela     |   (os números em si)
+------------+
```

## Cabeçalho — byte a byte

| Posição (offset) | Tamanho | O que é |
|---|---|---|
| 0 | 4 bytes | Letras "SPEC" (identifica o arquivo) |
| 4 | 1 byte | Número da versão do formato (`3`) |
| 5 | 4 bytes | Quantas linhas (frequências) a tabela tem |
| 9 | 4 bytes | Quantas colunas (instantes de tempo) a tabela tem |
| 13 | 4 bytes | Sample rate do áudio original (quantas amostras de som por segundo) |
| 17 | 4 bytes | Hop size (de quantas em quantas amostras de som pulamos pra cada coluna nova) |
| 21 | 1 byte | Tipo de energia guardada (ver tabela abaixo) |

Total do cabeçalho: **22 bytes**

### Tipo de energia (offset 21)

| Valor | Significado |
|---|---|
| 0 | Energia crua (magnitude) |
| 1 | Energia ao quadrado (potência) |
| 2 | Energia em decibéis |

Isso importa porque muda a escala dos números que o modelo vai ver. Recomenda-se fixar em `0` (o mais simples) a não ser que haja um motivo específico pra mudar.

### Sample rate e hop size — por que ficam no arquivo

Esses dois números, juntos, dizem quanto tempo (em segundos) cada coluna representa:

```
tempo_da_coluna_N (em segundos) = N * hop_size / sample_rate
```

Essa fórmula é o que permite mais tarde alinhar essa tabela com as notas do MIDI correspondente (arquivo `.ROLL`) — os dois arquivos usam a mesma fórmula, então sempre concordam sobre "o que aconteceu em cada momento".

Convenção fixa deste projeto: a coluna 0 sempre representa o tempo `0.0` segundos exato do áudio (sem nenhuma folga ou atraso no início).

---

## A tabela de dados

Depois do cabeçalho (a partir do byte 22), vêm os números da tabela, um após o outro, linha por linha:

```
linha 0: coluna 0, coluna 1, coluna 2, ...
linha 1: coluna 0, coluna 1, coluna 2, ...
...
```

Cada número ocupa **4 bytes** (tipo float32, o padrão para números com casas decimais).

---

## Como escrever um arquivo `.SPEC`

Você precisa ter em mãos: a tabela de energia já calculada, e os números de sample rate / hop size usados para calculá-la.

```
função escrever_spec(tabela, sample_rate, hop_size, tipo_de_energia):

    linhas = número de linhas da tabela
    colunas = número de colunas da tabela

    escreve "SPEC"
    escreve versão = 3
    escreve linhas
    escreve colunas
    escreve sample_rate
    escreve hop_size
    escreve tipo_de_energia

    para cada linha da tabela:
        para cada coluna dessa linha:
            escreve o número como float32
```

## Como ler um arquivo `.SPEC`

```
função ler_spec(arquivo):

    lê 4 bytes, confirma que é "SPEC"
    lê versão
    lê linhas
    lê colunas
    lê sample_rate
    lê hop_size
    lê tipo_de_energia

    cria uma tabela vazia de tamanho [linhas][colunas]

    para cada linha:
        para cada coluna:
            lê 4 bytes, interpreta como float32
            guarda na tabela

    devolve tabela, sample_rate, hop_size, tipo_de_energia
```

---

## Resultado final esperado

Uma matriz numérica (array 2D) representando o som ao longo do tempo, pronta para ser recortada em janelas fixas e alimentada num modelo de treino como entrada. Nada além disso — não é pensado para virar áudio de novo.
