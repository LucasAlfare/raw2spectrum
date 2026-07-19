# MREP — Especificação do Formato de Representação MIDI

**Versão:** 1.0

**Status:** Experimental

---

# Objetivo

O **MREP (MIDI Representation)** é um formato binário baseado em matriz cuja finalidade é representar um arquivo MIDI de forma simples, determinística e eficiente para processamento computacional.

Enquanto um arquivo MIDI armazena **eventos musicais** (Note On, Note Off, Program Change, etc.), um arquivo MREP armazena o **estado de cada nota ao longo do tempo**, em intervalos temporais fixos.

O formato foi projetado para ser utilizado em aplicações como:

* Aprendizado de Máquina
* Transcrição Automática de Música (AMT)
* Análise Musical
* Processamento Numérico

O formato é completamente independente de qualquer modelo de IA, framework ou representação de áudio.

---

# Objetivos de Projeto

* formato binário simples;
* leitura e escrita determinísticas;
* acesso aleatório em tempo constante;
* fácil implementação em qualquer linguagem;
* independente de espectrogramas;
* independente de datasets;
* independente de frameworks de Deep Learning;
* otimizado para processamento numérico.

---

# Extensão

```text
.mrep
```

---

# Endianness

Todos os valores inteiros com mais de um byte são armazenados em **Little Endian**.

---

# Magic Number

ASCII

```text
MREP
```

Hexadecimal

```text
4D 52 45 50
```

---

# Estrutura Geral

```text
+----------------------+
| Cabeçalho (32 bytes) |
+----------------------+
| Frame 0              |
+----------------------+
| Frame 1              |
+----------------------+
| Frame 2              |
+----------------------+
| ...                  |
+----------------------+
```

---

# Cabeçalho

| Offset | Tamanho | Tipo    | Campo                             |
| -----: | ------: | ------- | --------------------------------- |
|      0 |       4 | char[4] | Magic                             |
|      4 |       2 | uint16  | Versão                            |
|      6 |       2 | uint16  | Reservado                         |
|      8 |       4 | uint32  | Duração do Frame (microssegundos) |
|     12 |       4 | uint32  | Quantidade de Frames              |
|     16 |       2 | uint16  | Quantidade de Notas               |
|     18 |       2 | uint16  | Tipo de Valor                     |
|     20 |      12 | bytes   | Reservado                         |

Tamanho total do cabeçalho:

```text
32 bytes
```

---

# Campos do Cabeçalho

## Magic

Sempre:

```text
MREP
```

---

## Versão

Versão atual:

```text
1
```

---

## Reservado

Reservado para futuras versões.

Todos os bytes devem ser preenchidos com zero.

---

## Duração do Frame

Tempo representado por cada frame.

Unidade:

```text
microssegundos
```

Exemplo:

```text
10000
```

equivale a

```text
10 milissegundos
```

Esse valor torna o arquivo completamente autossuficiente, permitindo que qualquer programa interprete corretamente sua resolução temporal.

---

## Quantidade de Frames

Quantidade total de frames armazenados no arquivo.

---

## Quantidade de Notas

Na versão 1:

```text
128
```

Existe uma coluna para cada nota MIDI (0–127).

---

## Tipo de Valor

Define o significado de cada célula da matriz.

Na versão 1:

```text
1 = Estado Binário da Nota
```

Versões futuras poderão adicionar outros tipos.

---

# Organização dos Dados

Após o cabeçalho são armazenados todos os frames, em sequência.

```text
Frame 0
128 bytes

Frame 1
128 bytes

Frame 2
128 bytes

...
```

Cada frame possui exatamente **128 bytes**.

Cada byte representa uma nota MIDI.

---

# Organização da Matriz

Os dados são organizados como:

```text
matriz[frame][nota]
```

onde:

* **frame** representa o tempo;
* **nota** representa uma das 128 notas MIDI.

---

# Valores das Células

Cada célula ocupa exatamente **1 byte sem sinal (UInt8)**.

Valores permitidos:

```text
0
```

A nota está desligada.

```text
1
```

A nota está ligada.

Nenhum outro valor é válido na versão 1.

---

# Exemplo

Suponha que durante determinado frame estejam soando:

```text
C4
E4
G4
```

Então o frame conterá:

```text
...

Nota 60 = 1

Nota 64 = 1

Nota 67 = 1

...
```

Todas as demais posições conterão:

```text
0
```

---

# Tamanho do Arquivo

O tamanho do arquivo é determinístico.

```text
32 + (QuantidadeDeFrames × 128)
```

bytes.

---

# Algoritmo de Escrita

## Entrada

```text
Arquivo MIDI
```

## Saída

```text
Arquivo MREP
```

---

### Passo 1

Ler o arquivo MIDI.

Reconstruir todas as notas.

Cada nota deve possuir:

```text
pitch

tempoInicial

tempoFinal
```

---

### Passo 2

Determinar a duração total da música.

---

### Passo 3

Calcular

```text
QuantidadeDeFrames =
ceil(duraçãoTotal / duraçãoDoFrame)
```

---

### Passo 4

Criar uma matriz

```text
QuantidadeDeFrames × 128
```

Inicializar todas as posições com zero.

---

### Passo 5

Para cada nota:

```text
frameInicial =
floor(tempoInicial / duraçãoDoFrame)

frameFinal =
ceil(tempoFinal / duraçãoDoFrame)
```

Então:

```text
para frame = frameInicial até frameFinal

    matriz[frame][pitch] = 1
```

---

### Passo 6

Escrever o cabeçalho.

---

### Passo 7

Escrever todos os frames em sequência.

Cada frame possui exatamente:

```text
128 bytes
```

Sem separadores.

Sem compressão.

Sem metadados entre os frames.

---

# Pseudocódigo de Escrita

```text
notas = lerMidi()

duracao = maiorTempoFinal(notas)

quantidadeDeFrames =
ceil(duracao / duracaoDoFrame)

matriz =
zeros(quantidadeDeFrames, 128)

para cada nota

    inicio =
    floor(nota.tempoInicial / duracaoDoFrame)

    fim =
    ceil(nota.tempoFinal / duracaoDoFrame)

    para frame = inicio até fim

        matriz[frame][nota.pitch] = 1

escreverCabecalho()

para cada frame

    escrever(frame)
```

---

# Algoritmo de Leitura

1. Abrir o arquivo.
2. Verificar o Magic Number.
3. Ler o cabeçalho.
4. Alocar a matriz.
5. Ler todos os frames.

---

# Pseudocódigo de Leitura

```text
cabecalho = lerCabecalho()

matriz =
novaMatriz(
    cabecalho.quantidadeDeFrames,
    cabecalho.quantidadeDeNotas
)

para frame

    para nota

        matriz[frame][nota] =
        lerUInt8()
```

---

# Complexidade

Escrita:

```text
O(número de notas + número de frames ativos)
```

Leitura:

```text
O(QuantidadeDeFrames × 128)
```

Acesso aleatório a qualquer nota em qualquer frame:

```text
O(1)
```

---

# Por que a Velocity não é armazenada?

A versão 1 do formato armazena **apenas o estado binário de cada nota** (ligada ou desligada).

Essa decisão foi tomada pelos seguintes motivos:

* o principal objetivo da Transcrição Automática de Música é determinar **quais notas estão ativas ao longo do tempo**;
* representar apenas o estado ligado/desligado simplifica significativamente a estrutura dos dados;
* a ausência da velocity reduz a complexidade da representação e do treinamento dos modelos;
* a estimativa da intensidade de cada nota pode ser tratada futuramente como uma tarefa independente.

Nada impede que versões futuras do formato adicionem suporte à velocity ou a outras informações expressivas, mantendo compatibilidade com a estrutura básica definida nesta especificação.
