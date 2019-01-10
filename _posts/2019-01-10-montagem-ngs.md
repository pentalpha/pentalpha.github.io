---
layout: post
title: Montagem NGS de Genomas
tags: [montagem, ngs, genoma, spades, abyss, velvet, fastq, fasta, quast, redundans]
---

# O que é Montagem de Sequências?

Em bioinformática, montagem de sequências se refere a alinhar e juntar fragmentos de sequências de DNA com o objetivo de reconstruir a sequência original. Isso é necessário, pois as técnologias de sequenciamento não podem ler um genoma inteiro de uma vez, muito pelo contrário, elas lêem pequenos fragmentos. As técnologias mais recentes (NGS), lêem fragmentos ainda menores que as Sanger.

> O problema da montagem de sequencias pode ser comparado a pegar várias cópias de um livro, passar cada página delas por um triturador e então tentar reconstruir uma cópia do livro a partir dos fragmentos de texto restantes. (Wikipedia)

Como é de se esperar, essa é uma tarefa computacionalmente complexa e intensiva. Existem muitos software disponíveis para fazer isso. Aqui, vamos dar instruções sobre como usar cada um deles.

# Um guia para montagem De Novo a partir de dados de NGS

As principais ferramentas que vamos utilizar aqui:

1. [prefetch](#1-prefetch) 
2. [fastq-dump](#2-fastq-dump)  
3. [Option A: spades](#3-option-a-spades)
4. [Option B: abyss](#4-option-b-abyss)
5. [Option C: SOAPdenovo](#5-option-c-SOAPdenovo)
5. [Option C: Velvet](#6-option-d-velvet)
6. [Quast e Redundans: Analizando e melhorando os resultados](#7-quast-e-redundans)

Antes de prosseguir, abra um terminal para uma máquina Linux e entre na sua pasta pessoal. Então, crie uma pasta chamada "ngs-assembly" para guardar os arquivos gerados nesta tarefa.

# 1. prefetch
Caso o prefetch não esteja instalado, use o seguinte comando para instalar ele, juntamente a todo o sra-toolkit:

```sh
    $ sudo apt-get install sra-toolkit
```

## 1.1. Aplicação

Neste exemplo, vamos baixar os dados SRA SRR522243 e ......245.

### 1.1.1. SRR552224X

> Illumina whole genome shotgun sequencing of genomic DNA paired-end library 'Solexa-42867' containing sample Rhodobacter LW1. [URL](https://trace.ncbi.nlm.nih.gov/Traces/sra/sra.cgi?view=run_browser&run=SRR522243)

### 1.1.2. Baixando os dados.

```sh
    $ prefetch SRR522243
    $ prefetch SRR522245
```
Os arquivos não foram baixados para o seu diretório atual, eles provavelmente estão em alguma pasta do próprio sra-toolkit. Você precisa recuperar eles com o fastq-dump.

[For more info on prefatch](prefatch.md)

# 2. fastq-dump

Basicamente é uma das ferramentas '*-dump' do sra-toolkit.
```sh
    $ fastq-dump --split-files SRR522243
    $ fastq-dump --split-files SRR522245
```
Isso vai criar os seguintes arquivos .fastq no seu diretório atual:
SRR522243_2.fastq  SRR522245_2.fastq
SRR522243_1.fastq  SRR522245_1.fastq

Por causa do --split-files arg., há arquivos _1 e _2 files. Esses são os reads pareados.

Agora vamos concatenar os arquivos fastq em dois arquivos, o das reads esquerdas e o das reads no sentido direito:
```sh
    $ cat SRR522243_1.fastq  SRR522245_1.fastq > Rhodo_Hiseq_read1.fastq
    $ cat SRR522243_2.fastq  SRR522245_2.fastq > Rhodo_Hiseq_read2.fastq
    $ mkdir raw 
    $ mv *_[12].fastq raw/
    $ tree .
```

O último comando, "tree .", é uma dica de como ver não só os arquivos e sub-diretórios no diretório atual, como o conteúdo dos sub-diretórios.

# 3. Option A: spades

Spades é um montador de genomas russo escrito em Python, bastante utilizado. Como todo montador deve(ria) fazer, ele recebe os fastqs e cria um arquivo .fasta com os contigs montados.
[Official Webpage](http://cab.spbu.ru/software/spades/)

## 3.1 Installation

Caso o spades já não esteja instalado, execute o comando na sua máquina:
```sh
    $ sudo apt-get install spades
```

## 3.2 Using it

```sh
    $ spades.py -t 8 --pe1-1 Rhodo_Hiseq_read1.fastq --pe1-2 Rhodo_Hiseq_read2.fastq -o SpadesOut

    $ more SpadesOut/scaffolds.fasta

    $ more SpadesOut/contigs.fasta
```

Vai demorar alguns minutinhos. Ele vai criar em torno de 3150 contigs a partir dos reads passados. Também há um arquivo de scaffolds, porém o número de sequências (3122) não é muito menor.

[For information on scaffolds](https://genome.jgi.doe.gov/help/scaffolds.jsf)

O seguinte comando é um truque rápido para calcular o número de sequências em um arquivo .fasta, sem ter que recorrer ao quast:
```sh
    cat <nome_do_arquivo>.fasta | grep ">" | wc -l
```

# 4. Option B: ABySS

O montador com o nome mais legal.

> ABySS is a de novo, parallel, paired-end sequence assembler that is designed for short reads. (http://www.bcgsc.ca/platform/bioinfo/software/abyss)

## 4.1. Installation

Caso ainda não esteja instalado:

```sh
    $ sudo apt-get install abyss
```

## 4.2. Using it

```sh
    $ mkdir abyss
    $ cd abyss/
    $ abyss-pe k=31 l=1 n=5 s=100 np=8 \
          name=Rhodo \
          lib='reads' \
          reads='../Rhodo_Hiseq_read1.fastq ../Rhodo_Hiseq_read2.fastq' \
          aligner=bowtie
```

É possível que ocorram problemas no MPI caso você esteja tentando usar paralelismo. Caso isso aconteça, tire o "np=8".

Foram produzidos 19998 contigs (Rhodo-contigs.fa) e 19842 scaffolds (Rhodo-scaffolds.fa).

# 5. Option C: SOAPdenovo

## 5.1. Installation

#TODO

## 5.2. Using it

Primeiramente, para configurar a montagem, escreva um arquivo chamado "soap.config":

```sh
    $ mkdir soap
    $ cd soap
    $ nano soap.config
    $ SOAPdenovo-63mer all -s soap.config -o Rhodo -F -R -E -w -u -K 55 -p 8 >> SOAPdenovo.log
```

O conteúdo é o seguinte:
```sh
[LIB]
avg_ins=220

reverse_seq=0

asm_flags=3

rank=1

q1=../Rhodo_Hiseq_read1.fastq

q2=../Rhodo_Hiseq_read2.fastq
```

O comando vai demorar cerca de 2 minutos, produzindo 3408 scaffolds (Rhodo.scafSeq) e 3426 contigs (Rhodo.contig).

# 6. Option D: Velvet
Sequence assembler for very short reads.

## 6.1. Installation

On Ubuntu, install using apt-get:
```sh
    $ sudo apt-get install velvet
```
## 6.2. Using it

Usar o velvet consiste de 2 passos: velveth para fazer o "hashing" dos reads e velvetg para resolver o grafo de bruijin, construindo assim a montagem.

```sh
    $ velveth velvet/ 31 -shortPaired -fastq -separate Rhodo_Hiseq_read1.fastq Rhodo_Hiseq_read2.fastq
    $ velvetg velvet/ -cov_cutoff auto
    $ more velvet/contigs.fa
```

# 7. Quast e Redundans
Copie tudo para um diretório só:
```sh
    $ mkdir genomes/
    $ cp soap/Rhodo.scafSeq genomes/soap.scaff.fasta
    $ cp SpadesOut/scaffolds.fasta genomes/spades.scaff.fasta
    $ cp abyss/Rhodo-scaffolds.fa genomes/abyss.scaff.fasta
    $ cp velvet/contigs.fa genomes/velvet.contig.fasta
```

## 7.1. Quast.py
> QUAST-a quality assessment tool for evaluating and comparing genome assemblies. This tool improves on leading assembly comparison software with new ideas and quality metrics. QUAST can evaluate assemblies both with a reference genome, as well as without a reference. QUAST produces many reports, summary tables and plots to help scientists in their research and in their publications. [QUAST: quality assessment tool for genome assemblies](https://www.ncbi.nlm.nih.gov/pubmed/23422339);

Você pode baixar a partir da [sourceforge](http://quast.sourceforge.net/quast).

```sh
    $ quast.py -o quast_genomes/ genomes/*
    $ cd quast_genomes
    $ cat report.txt
```

Ele vai gerar "reports" em diversos formatos (html, tsv, txt...). Inclusive é gerada uma página do icarus, que pode ser aberta usando um web-browser.

## 7.2. redundans.py:
O redundans.py é uma pipeline para fechar gaps e reduzir a redundância dos contigs de uma montagem.

> Program takes as input assembled contigs, sequencing libraries and/or reference sequence and returns scaffolded homozygous genome assembly. Final assembly should be less fragmented and with total size smaller than the input contigs. In addition, Redundans will automatically close the gaps resulting from genome assembly or scaffolding. ([Github](https://github.com/lpryszcz/redundans))

Hora de executar ele para cada uma das nossas montagens:

```sh
    $ redundans.py -v -i ../*.fastq -f abyss.scaff.fasta -o abyss.gap -t 8
    $ redundans.py -v -i ../*.fastq -f spades.scaff.fasta -o spades.gap -t 8
    $ redundans.py -v -i ../*.fastq -f soap.scaff.fasta -o soap.gap -t 8
    $ redundans.py -v -i ../*.fastq -f velvet.contig.fasta -o velvet.gap -t 8
```

Ao final da execução, o redundans.py exibe uma pequena tabela com estatísticas (semelhante às do quast) sobre as sequências do arquivo de entrada e de cada arquivo gerado. Leia ela para ver o quanto cada montagem melhorou depois de passar pelo redundans.py.

Agora, vamos copiar os scaffolds produzidos pelo redundans.py para uma pasta propria e excluir os outros arquivos gerados:

```sh
    $ mkdir final_scaffolds
    $ mv abyss.gap/_gapcloser.1.2.fa final_scaffolds/abyss.fasta
    $ mv soap.gap/_gapcloser.1.2.fa final_scaffolds/soap.fasta
    $ mv spades.gap/_gapcloser.1.2.fa final_scaffolds/spades.fasta
    $ mv velvet.gap/_gapcloser.1.2.fa final_scaffolds/velvet.fasta
    $ rm -Rf *.gap/
    $ rm *.fai
```
PS: O resultado "oficial" é o *.gap/scaffolds.filled.fa, porém ele é um symlink para o arquivo _gapcloser.1.2.fa, por isso escrevi comandos para copiar o arquivo do _gapcloser, senão poderiam acontecer problemas depois de excluir os arquivos originais.

Agora, tente rodar o quast.py com os novos resultados em "final_scaffolds" e decida qual foi o melhor!