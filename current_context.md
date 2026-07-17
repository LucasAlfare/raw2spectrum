# Contexto do Projeto

Estou iniciando um projeto pessoal para aprender profundamente sobre Inteligência Artificial aplicada à música. Meu objetivo **não é apenas utilizar um modelo existente**, mas compreender toda a cadeia de processamento desde o áudio bruto até uma representação simbólica (MIDI) capaz de gerar uma partitura.

Sou desenvolvedor profissional, especializado em Kotlin, e possuo sólida formação musical. Já estudei cálculo, estatística e álgebra linear no passado, embora esses conhecimentos estejam enferrujados. Não quero apenas aprender a usar frameworks; quero entender a matemática, os conceitos e a arquitetura por trás das decisões.

Durante esta conversa ficou claro que meu objetivo não é:

```
MP3 → PDF
```

mas sim:

```
Áudio
    ↓
Representação adequada
    ↓
Modelo de IA
    ↓
Eventos MIDI
    ↓
MusicXML / LilyPond
    ↓
PDF
```

Essa mudança de perspectiva foi um dos maiores esclarecimentos da conversa.

---

# Objetivo principal

Quero construir um modelo capaz de aprender a seguinte função:

```
f(áudio) = MIDI
```

Onde a entrada pode ser:

- WAV
- MP3
- FLAC
- OGG
- qualquer formato de áudio

E a saída será um MIDI representando a música.

No futuro esse modelo poderá lidar com múltiplos instrumentos, mas inicialmente quero focar em piano.

Não quero pensar na geração da partitura neste momento porque ela passou a ser vista apenas como uma etapa posterior de renderização.

---

# Como passei a enxergar o problema

Inicialmente eu imaginava que o desafio seria algo parecido com:

```
áudio
↓

detectar notas

↓

partitura
```

Durante a conversa compreendi que isso é simplificar demais o problema.

Hoje passo a enxergar a arquitetura assim:

```
Áudio

↓

Pré-processamento

↓

Representação matemática

↓

Modelo

↓

Eventos MIDI

↓

Renderização
```

Essa mudança de mentalidade foi extremamente importante.

---

# O maior aprendizado

Percebi que a IA não deve receber diretamente um MP3.

Ela normalmente recebe uma representação muito mais organizada do áudio.

Essa representação é o espectrograma.

Passei a entender que:

- o áudio bruto é apenas uma sequência enorme de amplitudes;
- o espectrograma torna explícita a distribuição das frequências ao longo do tempo;
- isso facilita muito o trabalho da rede neural.

---

# O que mais me confundiu

Minha maior dificuldade foi entender o espectrograma.

Inicialmente imaginei que existisse algum "arquivo de espectrograma" padronizado.

Depois compreendi que:

- espectrograma é apenas uma matriz;
- essa matriz pode ser armazenada em inúmeros formatos;
- o importante é a informação, não o formato.

Mesmo depois disso continuei com dúvidas sobre:

- como essa matriz seria gravada em disco;
- como floats poderiam existir em um arquivo;
- como seria possível reconstruir a matriz durante a leitura.

---

# O que foi esclarecido

Foi construída uma especificação completa de um formato fictício chamado:

```
.SPEC
```

Esse formato serviu apenas para entendimento.

A especificação definiu:

- cabeçalho;
- assinatura;
- versão;
- quantidade de linhas;
- quantidade de colunas;
- sample rate;
- FFT size;
- window size;
- hop size;
- tipo dos dados;
- ordem de armazenamento (row-major);
- algoritmo de escrita;
- algoritmo de leitura;
- significado de cada campo;
- representação dos floats em IEEE-754;
- interpretação da matriz.

Essa explicação finalmente tornou claro que:

"o arquivo contém apenas bytes; quem transforma esses bytes em números é o programa durante a leitura."

Esse foi outro momento importante da conversa.

---

# Como agora enxergo o espectrograma

Hoje penso nele como:

```
matriz[freq][tempo]
```

Onde cada célula representa:

```
energia
```

daquela frequência naquele instante.

Ou seja,

```
M[linha][coluna]
```

não representa uma nota.

Representa apenas intensidade.

A IA aprenderá posteriormente a interpretar padrões dessa matriz.

---

# O primeiro software do projeto

Antes eu pensava em começar treinando uma IA.

Agora compreendi que existe uma etapa muito anterior.

Meu primeiro software será um pré-processador.

Inicialmente pensei em chamá-lo de:

```
raw2spectrum
```

Ele será responsável por:

```
MP3
WAV
FLAC
OGG
...

↓

Espectrograma
```

Sem qualquer inteligência artificial.

Seu único objetivo será produzir uma representação adequada para treinamento.

---

# Arquitetura inicial imaginada

Hoje imagino o projeto dividido em ferramentas independentes.

```
raw2wav
```

Padroniza todos os formatos de áudio.

↓

```
wav2spectrogram
```

Gera espectrogramas.

↓

```
midi2events
```

Transforma MIDI em eventos de treinamento.

↓

```
trainer
```

Treina o modelo.

↓

```
inference
```

Recebe um áudio novo e produz MIDI.

---

# Tecnologia

Embora Python seja o ecossistema dominante para IA, meu foco principal continua sendo Kotlin.

A intenção não é fugir do Python, mas compreender profundamente os conceitos antes de depender das ferramentas.

Quero ser capaz de entender exatamente o que cada biblioteca faz.

---

# Projeto criado

Foi criado o projeto:

```
raw2spectrum
```

Descrição:

> A simple and educational audio preprocessing library that converts raw audio (WAV, MP3, FLAC, OGG, etc.) into spectrogram representations suitable for machine learning, audio analysis, and automatic music transcription (AMT). Designed as the first stage of an audio-to-MIDI pipeline.

Licença:

```
MIT
```

Tags:

- audio
- audio-processing
- spectrogram
- stft
- fft
- signal-processing
- music
- music-information-retrieval
- automatic-music-transcription
- amt
- midi
- machine-learning
- deep-learning
- preprocessing
- kotlin

---

# Forma como gosto de aprender

Durante toda a conversa ficou evidente que prefiro compreender profundamente cada conceito antes de prosseguir.

Não gosto de tratar bibliotecas como caixas-pretas.

Sempre que surge uma abstração, procuro entender:

- exatamente o que existe em memória;
- exatamente quais bytes estão sendo gravados;
- exatamente como os dados são interpretados.

Explicações excessivamente abstratas tendem a gerar mais dúvidas.

Explicações concretas, próximas do baixo nível, funcionam muito melhor para mim.

---

# O que ainda falta entender

Ainda preciso compreender profundamente:

- Transformada de Fourier (FFT)
- STFT
- Mel Spectrogram
- Como exatamente uma janela da STFT produz um vetor de frequências.
- Como escolher Window Size.
- Como escolher Hop Size.
- Como o espectrograma muda visualmente conforme esses parâmetros.
- Como um modelo realmente "enxerga" essa matriz.
- Como representar corretamente o alvo (MIDI) para treinamento.
- Como funciona o treinamento propriamente dito.
- Como definir a função de perda.
- Como acontece o backpropagation.
- Como organizar um dataset grande.
- Como ocorre a inferência.

---

# Próximos passos

O próximo passo esperado é estudar profundamente a geração do espectrograma.

Não quero apenas utilizar uma biblioteca.

Quero compreender:

- por que a FFT funciona;
- por que a STFT existe;
- por que aparecem linhas horizontais;
- por que surgem harmônicos;
- como acordes aparecem na matriz;
- como diferentes instrumentos alteram o espectrograma.

Somente depois disso pretendo avançar para Deep Learning.

---

# Filosofia deste aprendizado

A intenção não é aprender IA rapidamente.

A intenção é construir conhecimento sólido.

Quero ser capaz de implementar praticamente todo o pipeline entendendo o motivo de cada etapa.

Sempre que possível, prefiro construir pequenas ferramentas próprias para compreender o funcionamento interno, mesmo que posteriormente utilize bibliotecas consolidadas em produção.

Meu objetivo final é que, ao terminar essa jornada, eu seja capaz de olhar para um sistema de Automatic Music Transcription e entender completamente cada bloco, desde o primeiro byte do áudio até o último símbolo da partitura.
