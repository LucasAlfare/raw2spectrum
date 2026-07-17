# CONTEXTO DO PROJETO

Este documento serve para contextualizar uma conversa contínua sobre um projeto de Inteligência Artificial voltado para transcrição automática de música (Automatic Music Transcription - AMT). Considere todas as informações abaixo como decisões já tomadas ou conceitos já compreendidos, evitando repetir explicações introdutórias.

---

# Objetivo Final

O objetivo do projeto é desenvolver, do zero, um sistema capaz de receber um arquivo de áudio (MP3, WAV, FLAC, OGG, etc.) e produzir um MIDI correspondente. A geração de partitura (MusicXML, LilyPond e PDF) será tratada como uma etapa posterior e separada.

O pipeline desejado é:

```text
Áudio
    ↓
Pré-processamento
    ↓
Representação para IA
    ↓
Modelo de Deep Learning
    ↓
Eventos MIDI
    ↓
MusicXML / LilyPond
    ↓
PDF
```

O foco atual **não é** a geração da partitura.

O foco atual é compreender completamente a transformação:

```text
Áudio → MIDI
```

---

# Objetivo de Aprendizado

O objetivo principal não é apenas construir um modelo funcional.

Quero compreender profundamente toda a arquitetura envolvida.

Prefiro entender cada transformação antes de utilizar bibliotecas prontas.

Meu aprendizado é orientado por compreensão dos conceitos e da matemática, não apenas pelo uso de frameworks.

---

# Meu Perfil

- Desenvolvedor profissional.
- Linguagem principal: Kotlin.
- Conhecimento avançado de programação.
- Conhecimento de música.
- Já estudei cálculo, estatística e álgebra linear (embora precise revisar diversos conceitos).
- Não tenho experiência prática com Deep Learning.

Não preciso de explicações básicas de programação.

---

# Estado Atual do Projeto

Ainda não comecei a desenvolver o modelo de IA.

Neste momento estou construindo toda a etapa de pré-processamento e entendendo as representações dos dados.

Ainda estou antes da etapa de treinamento.

---

# O que já compreendi

Durante as conversas anteriores compreendi os seguintes conceitos.

## O modelo não deve receber MP3 diretamente.

O áudio normalmente é convertido para uma representação mais adequada antes do treinamento.

Essa representação normalmente é um espectrograma.

---

## O espectrograma

Compreendi que:

- espectrograma não é um formato de arquivo;
- espectrograma é apenas uma matriz;
- cada linha representa uma frequência;
- cada coluna representa um instante no tempo;
- cada célula representa a intensidade daquela frequência naquele instante.

Em outras palavras:

```text
matriz[freq][tempo]
```

Onde

```text
matriz[linha][coluna]
```

representa energia.

Não representa notas.

Não representa acordes.

Não representa música.

A interpretação musical será aprendida posteriormente pela IA.

---

## Formato .SPEC

Foi criada uma especificação fictícia apenas para facilitar o entendimento do armazenamento do espectrograma.

Esse formato não é um padrão existente.

Ele foi criado apenas para compreender:

- organização dos dados;
- cabeçalho;
- armazenamento binário;
- representação IEEE-754;
- leitura;
- escrita;
- ordem row-major.

O objetivo era entender exatamente quais bytes seriam gravados em disco.

---

## O papel do pré-processamento

Antes eu imaginava começar treinando uma IA.

Hoje compreendo que existe uma etapa anterior extremamente importante.

Essa etapa transforma:

```text
Áudio
```

em

```text
Representação matemática
```

Essa representação será a entrada do modelo.

---

# Projeto Atual

O primeiro projeto criado chama-se:

```
raw2spectrum
```

Objetivo:

Converter arquivos de áudio em espectrogramas.

Esse projeto NÃO possui inteligência artificial.

Ele apenas realiza o pré-processamento necessário para futuros modelos.

Descrição do projeto:

> A simple and educational audio preprocessing library that converts raw audio (WAV, MP3, FLAC, OGG, etc.) into spectrogram representations suitable for machine learning, audio analysis, and automatic music transcription (AMT). Designed as the first stage of an audio-to-MIDI pipeline.

Licença:

```
MIT
```

---

# Arquitetura imaginada atualmente

A arquitetura geral atualmente pensada é:

```text
raw2wav

↓

raw2spectrum

↓

midi2events

↓

trainer

↓

inference
```

Cada ferramenta possui responsabilidade única.

Ainda não sei se essa arquitetura será definitiva.

---

# Como prefiro aprender

Prefiro explicações extremamente concretas.

Sempre que possível:

- partir do baixo nível;
- mostrar estruturas de dados;
- mostrar bytes;
- mostrar formatos;
- mostrar algoritmos;
- explicar o motivo da existência de cada etapa.

Não gosto de respostas que simplesmente dizem "a biblioteca faz isso".

Prefiro entender primeiro.

Depois utilizar bibliotecas.

---

# Como responder

Sempre considere que:

- já compreendo programação;
- já compreendo música;
- ainda estou aprendendo processamento de sinais;
- ainda estou aprendendo Machine Learning.

Evite voltar para explicações genéricas.

Prefira construir o conhecimento passo a passo.

Quando apresentar um conceito novo:

1. explique o problema que ele resolve;
2. explique intuitivamente;
3. explique tecnicamente;
4. mostre como ele se encaixa na arquitetura do projeto.

---

# O que ainda preciso aprender

Os próximos assuntos provavelmente serão estudados nesta ordem:

1. Representação digital do áudio.
2. FFT.
3. STFT.
4. Mel Spectrogram.
5. Como a STFT gera um espectrograma.
6. Como escolher Window Size.
7. Como escolher Hop Size.
8. Como interpretar visualmente um espectrograma.
9. Como representar MIDI para treinamento.
10. Estrutura do dataset.
11. Arquitetura do modelo.
12. Função de perda.
13. Backpropagation.
14. Processo completo de treinamento.
15. Inferência.

---

# Objetivo das próximas conversas

Assuma que quero construir conhecimento sólido.

Não estou procurando apenas uma implementação pronta.

Quero compreender profundamente cada etapa do pipeline.

Sempre que possível, estabeleça conexões entre os conceitos já aprendidos e os novos conceitos.

Evite respostas superficiais.

Prefiro uma explicação longa, incremental e tecnicamente correta do que uma resposta curta que esconda detalhes importantes.

Considere este documento como o estado atual do projeto e continue a partir dele.
