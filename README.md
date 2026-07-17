# Especificação do Formato `.SPEC` (Versão 1)

## Objetivo

O formato `.SPEC` foi criado para armazenar um espectrograma de maneira simples.

Um espectrograma é uma matriz onde:

- cada **linha** representa uma faixa de frequência;
- cada **coluna** representa um instante no tempo;
- cada célula representa a intensidade (energia) daquela frequência naquele instante.

O arquivo é composto apenas por bytes.

Todos os números inteiros são armazenados em **Little Endian**.

Todos os números de ponto flutuante são armazenados no padrão **IEEE-754 Float32**.

---

# Organização Geral

```
+----------------------+
| Cabeçalho            |
+----------------------+
| Matriz de Intensidade|
+----------------------+
```

---

# Cabeçalho

| Offset | Tamanho | Tipo | Descrição |
|---------|----------|------|-----------|
|0|4 bytes|ASCII|Assinatura `"SPEC"`|
|4|1 byte|uint8|Versão|
|5|4 bytes|uint32|Quantidade de linhas|
|9|4 bytes|uint32|Quantidade de colunas|
|13|4 bytes|uint32|Sample Rate original|
|17|4 bytes|uint32|FFT Size|
|21|4 bytes|uint32|Window Size|
|25|4 bytes|uint32|Hop Size|
|29|1 byte|uint8|Tipo do dado|

Total do cabeçalho:

```
30 bytes
```

---

# Campo: Assinatura

Offset:

```
0
```

Valor:

```
53 50 45 43
```

ASCII:

```
SPEC
```

Serve apenas para identificar o arquivo.

---

# Campo: Versão

Offset:

```
4
```

Valor:

```
01
```

---

# Campo: Quantidade de Linhas

Offset:

```
5
```

Tipo:

```
uint32
```

Representa quantas frequências existem.

Exemplo:

```
04 00 00 00
```

significa

```
4 linhas
```

---

# Campo: Quantidade de Colunas

Offset:

```
9
```

Tipo:

```
uint32
```

Representa quantos instantes existem.

Exemplo:

```
03 00 00 00
```

significa

```
3 colunas
```

---

# Campo: Sample Rate

Offset:

```
13
```

Tipo:

```
uint32
```

Exemplo:

```
44 AC 00 00
```

Representa

```
44100 Hz
```

---

# Campo: FFT Size

Offset:

```
17
```

Tipo:

```
uint32
```

Exemplo:

```
00 08 00 00
```

Representa

```
2048
```

---

# Campo: Window Size

Offset:

```
21
```

Tipo:

```
uint32
```

Exemplo:

```
00 08 00 00
```

Representa

```
2048
```

---

# Campo: Hop Size

Offset:

```
25
```

Tipo:

```
uint32
```

Exemplo:

```
00 02 00 00
```

Representa

```
512
```

---

# Campo: Tipo do Dado

Offset:

```
29
```

Tipo:

```
uint8
```

Valores possíveis:

|Valor|Significado|
|------|-----------|
|1|Float32 IEEE-754|
|2|Float64 IEEE-754|

Nesta versão recomenda-se utilizar apenas:

```
1
```

---

# Dados

Após o byte 29 começam os dados da matriz.

Os valores são gravados em ordem de linhas (Row-major).

A ordem é:

```
linha 0
    coluna 0
    coluna 1
    coluna 2
    ...

linha 1
    coluna 0
    coluna 1
    coluna 2
    ...

linha 2
...
```

---

# Exemplo

Matriz:

|Freq\Tempo|t0|t1|t2|
|-----------|--|--|--|
|f0|0.0|0.5|1.0|
|f1|0.2|0.4|0.6|

Ela será gravada exatamente nesta ordem:

```
0.0
0.5
1.0
0.2
0.4
0.6
```

Cada valor é convertido para Float32 IEEE-754.

Por exemplo:

|Valor|Bytes (Hex)|
|------|-----------|
|0.0|00 00 00 00|
|0.5|00 00 00 3F|
|1.0|00 00 80 3F|
|0.2|CD CC 4C 3E|
|0.4|CD CC CC 3E|
|0.6|9A 99 19 3F|

---

# Como Escrever

Entrada:

- Matriz `M`
- Número de linhas
- Número de colunas
- Sample Rate
- FFT Size
- Window Size
- Hop Size

Algoritmo:

```
escreva "SPEC"

escreva versão

escreva quantidade de linhas

escreva quantidade de colunas

escreva sample rate

escreva fft size

escreva window size

escreva hop size

escreva tipo do dado

para linha de 0 até linhas-1

    para coluna de 0 até colunas-1

        valor = M[linha][coluna]

        converta valor para Float32 IEEE-754

        escreva os 4 bytes
```

---

# Como Ler

```
leia assinatura

verifique se é "SPEC"

leia versão

leia quantidade de linhas

leia quantidade de colunas

leia sample rate

leia fft size

leia window size

leia hop size

leia tipo do dado

crie uma matriz M[linhas][colunas]

para linha de 0 até linhas-1

    para coluna de 0 até colunas-1

        leia 4 bytes

        interprete como Float32 IEEE-754

        M[linha][coluna] = valor
```

Após o término do algoritmo, a matriz reconstruída será exatamente igual à matriz originalmente gravada.

---

# Interpretação da Matriz

Cada elemento da matriz representa a energia observada em uma determinada frequência durante um determinado intervalo de tempo.

```
M[linha][coluna]
```

significa:

```
energia da frequência correspondente à "linha"

durante o intervalo de tempo correspondente à "coluna"
```

A correspondência entre índice e frequência é determinada pelos parâmetros armazenados no cabeçalho (`Sample Rate`, `FFT Size`, `Window Size` e `Hop Size`), permitindo que qualquer implementação reconstrua corretamente os eixos de frequência e de tempo do espectrograma.
