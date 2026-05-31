# Engenharia-Reversa-Formato-ELF-no-Raspberry-Pi-4-mB-vol.2
*Program Headers e Segmentos: Como o Kernel Carrega seu Programa*


> **No episódio anterior (kkkk)**: Volume 1 — vimos o que é um ELF, criamos e compilamos um `hello_64` (é um `hello_32`), e lemos o ELF Header byte a byte.
>
> **Ambiente utilizado**: Raspberry Pi 4, modelo:B, ARM64 (AArch64) OS 64-bit.


## Sumário

1. [Duas visões do mesmo arquivo](#1-duas-visões-do-mesmo-arquivo)
2. [O comando `readelf -l`](#2-o-comando-readelf--l)
3. [Anatomia completa da saída](#3-anatomia-completa-da-saída)
4. [Tipos de segmento — um por um](#4-tipos-de-segmento--um-por-um)
   - PHDR — Program Header Table
   - INTERP — O intérprete do programa
   - LOAD — O coração do programa
   - DYNAMIC — Configuração do linker dinâmico
   - GNU_STACK — Declaração sobre a stack
   - GNU_RELRO — Read-Only After Relocation
   - NOTE — Metadados opcionais
5. [FileSiz vs MemSiz — o mistério do .bss](#5-filesiz-vs-memsiz--o-mistério-do-bss)
6. [Flags de permissão — R, W, E](#6-flags-de-permissão--r-w-e)
7. [PIE vs no-PIE — endereços fixos vs aleatórios](#7-pie-vs-no-pie--endereços-fixos-vs-aleatórios)
8. [Como o kernel carrega o ELF na memória](#8-como-o-kernel-carrega-o-elf-na-memória)
9. [Observando a memória em tempo real](#9-observando-a-memória-em-tempo-real)
10. [Exercícios práticos](#10-exercícios-práticos)
11. [Recapitulação e próximos passos](#11-recapitulação-e-próximos-passos)


# 1. Duas visões do mesmo arquivo

No Volume 1 você aprendeu que o ELF tem **seções** (`.text`, `.data`, `.rodata`...). Agora vamos descobrir que o mesmo arquivo é lido de duas formas completamente diferentes, dependendo de quem o está lendo.
> **Analogia**: pense em uma planta arquitetônica de um apartamento. O **arquiteto** a lê pensando em paredes, tomadas e encanamento (detalhes internos). O **caminhão de mudança** a lê pensando só em: quantos cômodos, qual a porta de entrada, onde encaixa o sofá. São dois "mapas" diferentes do mesmo espaço físico.

```
┌───────────────────────────────────────────────────────────
│               AS DUAS VISÕES DO ARQUIVO ELF                         │
├───────────────────────────────────────────────────────────
│  VISÃO DO LINKER         │  VISÃO DO KERNEL / LOADER                │
│  (tempo de compilação)   │  (tempo de execução)                     │
├───────────────────────────────────────────────────────────
│  Usa: Section Headers    │  Usa: Program Headers                    │
│  Unidade: Seção          │  Unidade: Segmento                       │.
│  Ex: .text, .data, .bss  │  Ex: LOAD, DYNAMIC, INTERP               │
│                          │                                          │
│  Responde: "onde estão   │  Responde: "o que precisa ser            │
│  os símbolos, onde       │  carregado na memória, com que           │
│  debugar?"               │  permissões, em que endereço?"           │
│                          │                                          │
│  Pode ser removido com   │  OBRIGATÓRIO para executar               │
│  strip (e o binário      │  (sem Program Headers o kernel           │
│  ainda executa)          │  não consegue rodar o programa)          │
└──────────────────────────┴────────────────────────────────

Um SEGMENTO pode conter várias SEÇÕES.
Um SEGMENTO é a "embalagem de entrega"; as seções são o "conteúdo".
```


# 2. O comando `readelf -l`

O `-l` (letra L minúscula, de *segments*) lista os **Program Headers** — os segmentos.

```bash
cd ~/estudos_elf
readelf -l hello_64
```

> `readelf -l` e `readelf --segments` são idênticos. Use qual preferir.

Saída completa no Raspberry Pi 4 (ARM64):

```
Elf file type is DYN (Position-Independent Executable file)
Entry point 0x640
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align

  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x000000000000001b 0x000000000000001b  R      0x1
      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000630 0x0000000000000630  R      0x10000
  LOAD           0x0000000000010000 0x0000000000010000 0x0000000000010000
                 0x00000000000001a4 0x00000000000001a4  R E    0x10000
  LOAD           0x0000000000020000 0x0000000000020000 0x0000000000020000
                 0x0000000000000028 0x0000000000000028  R      0x10000
  LOAD           0x0000000000020dc0 0x0000000000030dc0 0x0000000000030dc0
                 0x0000000000000268 0x0000000000000270  RW     0x10000
  DYNAMIC        0x0000000000020dd0 0x0000000000030dd0 0x0000000000030dd0
                 0x00000000000001e0 0x00000000000001e0  RW     0x8
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000020dc0 0x0000000000030dc0 0x0000000000030dc0
                 0x0000000000000240 0x0000000000000240  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version
          .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .data .bss
   06     .dynamic
   07
   08     .init_array .fini_array .dynamic .got
```

> **Atenção ao alinhamento**: No ARM64, o campo `Align` dos LOADs é `0x10000` (64 KB), não `0x1000` (4 KB) como no x86_64. Isso é o padrão do linker `ld` para aarch64. As páginas de memória física ainda são 4 KB, mas o linker reserva blocos de 64 KB para garantir compatibilidade com kernels que usam páginas maiores.


# 3. Anatomia completa da saída

Antes de mergulhar em cada tipo de segmento, vamos entender as colunas da tabela:

```
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  ────────────   ──────────────     ────────────────   ──────────────
  LOAD           0x0000000000010000 0x0000000000010000 0x0000000000010000
                 0x00000000000001a4 0x00000000000001a4  R E    0x10000
```

```
┌───────────────────────────────────────────────────────────────
│  Coluna       │  Significado                                             │
├───────────────────────────────────────────────────────────────
│  Type         │  Tipo do segmento (LOAD, INTERP, DYNAMIC...)             │
│  Offset       │  Onde o segmento começa DENTRO DO ARQUIVO (em bytes)     │
│  VirtAddr     │  Endereço VIRTUAL onde será carregado na memória         │
│  PhysAddr     │  Endereço físico — ignorado pelo kernel em user-space    │
│               │  (sempre igual ao VirtAddr em processos normais)         │
│  FileSiz      │  Quantos bytes ocupam NO ARQUIVO                         │
│  MemSiz       │  Quantos bytes ocupam NA MEMÓRIA (pode ser maior!)       │
│  Flags        │  Permissões: R=leitura, W=escrita, E=execução            │
│  Align        │  Alinhamento de memória exigido                          │
└───────────────────────────────────────────────────────────────
```

> **Por que PhysAddr existe se é ignorado?** É uma herança da spec ELF original, que precisava suportar sistemas embarcados sem MMU (sem memória virtual). Em sistemas com MMU — como o Raspberry Pi — o kernel gerencia a tradução virtual→físico e ignora este campo.

---

# 4. Tipos de segmento — um por um

Vamos dissecar cada linha da saída do `readelf -l`, como um médico legista examinando cada órgão.

---

## 4.1 PHDR — Program Header Table

```
PHDR  0x0000000000000040  0x0000000000000040  0x0000000000000040
      0x00000000000001f8  0x00000000000001f8   R   0x8
```

**O que é**: Descreve a própria tabela de Program Headers. É o ELF dizendo: *"eu mesmo estou aqui dentro do arquivo."*

```
┌───────────────────────────────────────────
│  ELF Header (64 bytes)                            │
│  ────────────────────                         │
│  Program Header Table  ← PHDR aponta para aqui   │
│  ├── Entry 0: PHDR                               │
│  ├── Entry 1: INTERP                             │
│  ├── Entry 2: LOAD                               │
│  └── ...                                         │
│                                                   │
│  [ conteúdo das seções... ]                       │
│                                                   │
│  Section Header Table                             │
└───────────────────────────────────────────┘
```

- **Offset `0x40`** = logo após o ELF Header (que tem 64 bytes = `0x40`)
- **Flag `R`** = somente leitura (faz sentido, ninguém deve modificar os headers)
- **Não é carregado como código ou dado do programa** — existe para que o loader (e o próprio programa, se necessário) saiba onde está a tabela

---

## 4.2 INTERP — O intérprete do programa

```
INTERP  0x0000000000000238  0x0000000000000238  ...
        0x000000000000001b  0x000000000000001b   R   0x1
    [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
```

**O que é**: Contém o caminho (string) do **dynamic linker** — o programa que vai carregar as bibliotecas `.so` que seu executável precisa.

```
Arquivo ELF
┌─────────────────────────────────────────────
│  ...                                               │
│  Offset 0x238: /lib/ld-linux-aarch64.so.1\0        │
│                ↑ 27 bytes de string ASCII + \0     │
└─────────────────────────────────────────────
```

> **Prática**: veja o conteúdo bruto do INTERP com hexdump:
> ```bash
> hexdump -C hello_64 | grep -A1 "ld-linux"
> # Ou diretamente pelo offset:
> dd if=hello_64 bs=1 skip=$((0x238)) count=27 2>/dev/null | cat
> ```

**Por que isso existe?** Quando você executa `./hello_64`, o kernel lê o ELF, encontra o INTERP, e **delega** o trabalho ao dynamic linker. O kernel não sabe como resolver `libc.so.6` — esse é o trabalho do `ld-linux-aarch64.so.1`.

> **Curiosidade de reversing**: malwares Linux às vezes modificam o INTERP para apontar para um "loader falso" que injeta código antes de chamar o programa real. Verificar se o INTERP é o esperado é uma etapa de análise de binários suspeitos.


## 4.3 LOAD — O coração do programa

Os segmentos `LOAD` são os mais importantes: são eles que o kernel realmente **copia para a memória**. Todo o resto é metadado.

No nosso `hello_64` temos 4 segmentos LOAD:

```
┌────────────────────────────────────────────────────────────────
│  SEGMENTOS LOAD DO hello_64 (ARM64, PIE)                                
├────────────────────────────────────────────────────────────────│
│  #   │  Offset      │  VirtAddr    │  Flags  │  Contém (seções)        
├────────────────────────────────────────────────────────────────│
│  [0] │  0x000000    │  0x000000    │   R     │  .interp, .note.*,      
│      │              │              │         │  .dynsym, .dynstr,      
│      │              │              │         │  .gnu.hash, .rela.*     
├────────────────────────────────────────────────────────────────│
│  [1] │  0x010000    │  0x010000    │   R E   │  .init, .plt, .text,   
│      │              │              │         │  .fini   ← CÓDIGO       
├────────────────────────────────────────────────────────────────│
│  [2] │  0x020000    │  0x020000    │   R     │  .rodata, .eh_frame*      
│      │              │              │         │  ← DADOS SOMENTE-LEITURA 
├────────────────────────────────────────────────────────────────│
│  [3] │  0x020dc0    │  0x030dc0    │   RW    │  .init_array, .dynamic,   
│      │              │              │         │  .got, .data, .bss        
│      │              │              │         │  ← DADOS LEITURA/ESCRITA
└────────────────────────────────────────────────────────────────┘
```

> **Princípio de segurança**: cada segmento LOAD tem permissões bem definidas. Código (`R E`) nunca é gravável. Dados (`RW`) nunca são executáveis (quando NX está ativo). Isso é o **W^X** (Write XOR Execute) — uma das proteções fundamentais contra exploits.


## 4.4 DYNAMIC — Configuração do linker dinâmico

```
DYNAMIC  0x0000000000020dd0  0x0000000000030dd0  ...
         0x00000000000001e0  0x00000000000001e0   RW  0x8
```

**O que é**: Uma tabela de pares `(tag, valor)` que o dynamic linker (`ld-linux-aarch64.so.1`) lê para saber o que fazer. É como um "manual de instruções" para o loader.

```bash
# Ver o conteúdo do segmento DYNAMIC:
readelf -d hello_64
```

Saída (resumida):

```
Dynamic section at offset 0x20dd0 contains 27 entries:
  Tag              Type              Name/Value
  0x0000000000000001 (NEEDED)        Shared library: [libc.so.6]
  0x000000000000000c (INIT)          0x10000
  0x000000000000000d (FINI)          0x10190
  0x0000000000000019 (INIT_ARRAY)    0x30db8
  0x000000000000001b (INIT_ARRAYSZ)  8 (bytes)
  0x0000000000000004 (HASH)          ...
  0x000000006ffffef5 (GNU_HASH)      ...
  0x0000000000000005 (STRTAB)        ...
  0x0000000000000006 (SYMTAB)        ...
  ...
```

```
┌────────────────────────────────────────────────────────
│  ENTRADAS MAIS IMPORTANTES DO DYNAMIC                            │
├──────────────────┬─────────────────────────────────────
│  Tag             │  Significado                                  │
├──────────────────┼─────────────────────────────────────
│  NEEDED          │  Nome da .so que precisa ser carregada        │
│  INIT            │  Endereço da função de inicialização (.init)  │
│  FINI            │  Endereço da função de finalização (.fini)    │
│  STRTAB          │  Tabela de strings de símbolos                │
│  SYMTAB          │  Tabela de símbolos dinâmicos                 │
│  RELA / RELASZ   │  Tabela de relocações                         │
│  PLTGOT          │  Endereço da GOT (Global Offset Table)        │
│  JMPREL          │  Relocações da PLT                            │
└──────────────────┴─────────────────────────────────────
```

> **Importante**: O segmento DYNAMIC está **contido dentro** do último LOAD (o RW). Ele não é carregado separadamente — o kernel carrega o LOAD inteiro, e o dynamic linker então localiza a seção DYNAMIC dentro desse bloco já mapeado.


## 4.5 GNU_STACK — Declaração sobre a stack

```
GNU_STACK  0x0000000000000000  0x0000000000000000  ...
           0x0000000000000000  0x0000000000000000   RW  0x10
```

**O que é**: Uma declaração de política — diz ao kernel quais permissões a **stack** do processo deve ter.

```
GNU_STACK com flags RW  → stack tem permissão de leitura e escrita
                          SEM execução → NX (No-Execute) HABILITADO 

GNU_STACK com flags RWE → stack é executável
                          NX DESABILITADO (perigoso em produção)
```

> **Por que isso importa em segurança?** Exploits clássicos de buffer overflow injetavam shellcode na stack e desviavam a execução para lá. O NX (equivalente ao DEP no Windows) previne isso. Se você vir `RWE` aqui num binário suspeito — levanta a bandeira vermelha.

```bash
# Verificar se NX está habilitado:
readelf -l hello_64 | grep "GNU_STACK"
# RW  → NX habilitado 
# RWE → NX desabilitado 

# Ou com checksec (se instalado):
# checksec --file=hello_64
```


## 4.6 GNU_RELRO — Read-Only After Relocation

```
GNU_RELRO  0x0000000000020dc0  0x0000000000030dc0  ...
           0x0000000000000240  0x0000000000000240   R  0x1
```

**O que é**: Marca uma região que começa como `RW` (durante o carregamento), mas é **remarcada como `R` (somente leitura)** depois que o dynamic linker termina de fazer as relocações.

```
Linha do tempo de um processo com RELRO:

Kernel carrega LOAD[3] (RW) ──────────────────────────►
                                     │
                    Dynamic linker preenche .got, .dynamic
                                     │
                    ld.so chama mprotect() na região RELRO
                                     │
                    .got, .dynamic viram somente-leitura (R)
                                     │
                              main() é chamado ──────────►

Resultado: atacante não consegue sobrescrever a GOT para redirecionar chamadas
```

> **RELRO Parcial vs Total**: RELRO parcial (padrão) protege `.dynamic` e `.got`. RELRO total (`gcc -Wl,-z,relro,-z,now`) também protege a `.got.plt`. Para CTFs e análise de exploits, saber se RELRO é parcial ou total é fundamental.


## 4.7 NOTE — Metadados opcionais

```
NOTE  0x0000000000000254  0x0000000000000254  ...
      0x0000000000000024  0x0000000000000024   R  0x4
```

**O que é**: Informações extras não-críticas para execução. Podem conter:

- **`gnu.build-id`**: hash SHA1 único deste build (útil para debug e símbolos remotos)
- **`ABI-tag`**: versão mínima do kernel Linux necessária
- **`gnu.property`**: propriedades de arquitetura (ex: suporte a BTI no ARM64)

```bash
# Ver as notas:
readelf -n hello_64
```


# 5. FileSiz vs MemSiz — o mistério do .bss

Esta é uma das partes mais elegantes do formato ELF. Repare no último LOAD:

```
LOAD  offset=0x020dc0  VirtAddr=0x030dc0  Flags=RW
      FileSiz=0x268    MemSiz=0x270
               ↑                ↑
          no arquivo        na memória
```

**MemSiz é maior que FileSiz.** A diferença são **8 bytes** neste caso. Por quê?

### A seção `.bss`

```
.bss = Block Started by Symbol
     = variáveis globais/estáticas NÃO inicializadas
```

```c
// Exemplo:
int contador;          // vai para .bss (não inicializado)
int valor = 42;        // vai para .data (inicializado com 42)
const char* msg = "Olá"; // vai para .rodata (somente-leitura)
```

**O truque**: variáveis não inicializadas devem valer zero por padrão (em C). Mas não faz sentido guardar milhares de bytes de zeros no arquivo em disco. Então o ELF faz:

```
No arquivo:   FileSiz bytes contêm os dados reais (.data, .got, etc.)
Na memória:   MemSiz bytes são reservados = FileSiz + tamanho do .bss
              Os bytes extras (o .bss) são zerados pelo kernel automaticamente
```

```
ARQUIVO EM DISCO:                    MEMÓRIA APÓS CARREGAMENTO:
┌──────────────────               ┌────────────────────
│ .init_array        │               │ .init_array           │
│ .fini_array        │     mmap()    │ .fini_array           │
│ .dynamic           │ ──────────► │ .dynamic              │
│ .got               │               │ .got                  │
│ .data              │               │ .data                 │
└──────────────────               │ .bss  ← 0x00 0x00    │
   FileSiz = 0x268                   │       ← 0x00 0x00     │
                                     └────────────────────
                                        MemSiz = 0x270
                                     (8 bytes extras = zeros)
```

> **Isso economiza espaço em disco**: um binário com `int buffer[1000000];` global não precisará de 4 MB no arquivo, apenas uma entrada no .bss dizendo "reserve 4 MB de zeros aqui".


# 6. Flags de permissão — R, W, E

Uma confusão muito comum: o `readelf` usa a letra **`E`** para executável, **não `X`**.

```
┌─────────────────────────────────────────────────────────
│  Flag │  Significado                                             │
├─────────────────────────────────────────────────────────
│  R    │  Readable   — pode ser lido                              │
│  W    │  Writable   — pode ser escrito                           │
│  E    │  Executable — pode ser executado como código             │
└─────────────────────────────────────────────────────────

Combinações típicas:
  R     → somente leitura (headers, metadados, .rodata)
  R E   → leitura + execução (.text, código do programa)
  RW    → leitura + escrita (.data, .bss, .got, stack)
  RWE   → tudo liberado ← RARO e perigoso (stack executável)
```

> **Atenção**: o `objdump -p` usa notação diferente — letras minúsculas com hífen para ausência: `r-x`, `rw-`, `r--`. Não confunda as duas ferramentas.

```bash
# readelf: flags com letras maiúsculas, espaço entre presentes
readelf -l hello_64 | grep "Flags\|R E\|RW\| R "

# objdump: flags com letras minúsculas, hífen para ausentes
objdump -p hello_64 | grep "flags"
# Saída: "flags r-x"  ou  "flags rw-"
```

### Como as flags se traduzem em proteções de memória

```
Flag ELF    →   Proteção mmap    →   syscall mprotect
─────────       ─────────────────    ──────────────────
R           →   PROT_READ        →   0x1
W           →   PROT_WRITE       →   0x2
E           →   PROT_EXEC        →   0x4
RW          →   PROT_READ|PROT_WRITE         → 0x3
R E         →   PROT_READ|PROT_EXEC          → 0x5
RWE         →   PROT_READ|PROT_WRITE|PROT_EXEC → 0x7
```

---

# 7. PIE vs no-PIE — endereços fixos vs aleatórios

Esta diferença aparece claramente ao comparar os VirtAddr dos segmentos LOAD.

```bash
# Compilar as duas versões
gcc -o hello_pie    hello.c           # PIE (padrão)
gcc -no-pie -o hello_nopie hello.c   # Endereços fixos

# Comparar os VirtAddr
readelf -l hello_pie   | grep "LOAD"
readelf -l hello_nopie | grep "LOAD"
```

```
PIE (hello_pie):
  LOAD  VirtAddr=0x0000000000000000  R
  LOAD  VirtAddr=0x0000000000010000  R E
  LOAD  VirtAddr=0x0000000000020000  R
  LOAD  VirtAddr=0x0000000000030dc0  RW
        ↑
        Começa em 0x0 → são OFFSETS, não endereços absolutos
        O kernel + ASLR escolherá a base real em tempo de execução

no-PIE (hello_nopie):
  LOAD  VirtAddr=0x0000000000400000  R
  LOAD  VirtAddr=0x0000000000401000  R E
  LOAD  VirtAddr=0x0000000000402000  R
  LOAD  VirtAddr=0x0000000000403dc0  RW
        ↑
        Endereços ABSOLUTOS e FIXOS
        Sempre carregado aqui, toda execução
```

```
┌───────────────────────────────────────────────────────────
│              PIE  vs  no-PIE na memória virtual                     │
│                                                                     │
│  PIE (com ASLR):          no-PIE (sem ASLR):                        │
│                                                                     │
│  Execução 1:              Toda execução:                            │
│  base = 0x5566ab000000    base = 0x0000000000400000                 │
│  .text em 0x5566ab010000  .text em 0x0000000000401000               │
│                                                                     │
│  Execução 2:              Previsível → mais fácil de explorar      
│  base = 0x7f3c12000000                                              │
│  .text em 0x7f3c12010000                                            │
│                                                                     │
│  Imprevisível → mais difícil de explorar                           
└───────────────────────────────────────────────────────────
```

> **Para análise e CTFs**: `hello_nopie` é mais fácil de analisar porque os endereços são sempre os mesmos. Em produção, prefira PIE. A combinação PIE + ASLR é uma proteção relevante contra ataques de memória.


# 8. Como o kernel carrega o ELF na memória

Agora que entendemos todos os segmentos, vamos ver o processo completo — da chamada `execve()` até a execução do `main()`. Esta é a sequência mais importante deste volume.

## A sequência completa

```
┌─────────────────────────────────────────────────────────────┐
│  JORNADA DE ./hello_64 DO SHELL ATÉ O main()                           │
└─────────────────────────────────────────────────────────────┘

  Você digita: ./hello_64
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Shell (bash/zsh) │  chama a syscall execve("./hello_64", argv, envp)
└─────────┬───────────────────────────────────────────────────┘
            │  syscall execve = ARM64 syscall nº 221 (0xDD)
            ▼
┌─────────────────────────────────────────────────────────────
│                         KERNEL LINUX                                  │
│                                                                       │
│  PASSO 1: Lê os primeiros bytes → verifica magic 7F 45 4C 46        
│           Confirma: é um ELF válido                                   │
│                                                                       │
│  PASSO 2: Lê o ELF Header → descobre e_phoff e e_phnum               
│           Encontra a Program Header Table                             │
│                                                                       │
│  PASSO 3: Varre os Program Headers                                    │
│           ├── Acha PT_INTERP → lê caminho do dynamic linker          │
│           │   (/lib/ld-linux-aarch64.so.1)                            │
│           └── Acha todos os PT_LOAD do binário principal              
│                                                                       │
│  PASSO 4: Mapeia os PT_LOAD do binário na memória virtual            
│           usando mmap() internamente                                  │
│           ├── LOAD[0] (R)   → mapeado como somente-leitura           │
│           ├── LOAD[1] (R E) → mapeado como leitura + execução        │
│           ├── LOAD[2] (R)   → mapeado como somente-leitura           │
│           └── LOAD[3] (RW)  → mapeado como leitura + escrita         │
│               (MemSiz > FileSiz → bytes extras = zeros para .bss)    
│                                                                       │
│  PASSO 5: Abre e lê o ELF do dynamic linker                           │
│           (/lib/ld-linux-aarch64.so.1)                                │
│           Mapeia os PT_LOADs do ld.so na memória                      │
│                                                                       │
│  PASSO 6: Monta a stack inicial do processo                           │
│           ├── argc, argv[], envp[]                                   
│           └── auxv (auxiliary vector):                               
│               AT_PHDR   → endereço da Phdr Table na memória          │
│               AT_PHENT  → tamanho de cada Phdr entry                 │
│               AT_PHNUM  → número de Phdr entries                     │
│               AT_ENTRY  → entry point do binário principal           │
│               AT_BASE   → base address do ld.so                      │
│               AT_RANDOM → 16 bytes aleatórios (para stack canary)    │
│                                                                       │
│  PASSO 7: Transfere controle para o entry point do ld.so              │
│           (não para o entry point do programa ainda!)                 │
└─────────────────────────────────────────────────────────────
          │
          ▼
┌─────────────────────────────────────────────────────────────
│                  DYNAMIC LINKER (ld-linux-aarch64.so.1)               │
│                                                                       │
│  PASSO 8: Lê o segmento PT_DYNAMIC do binário principal               │
│           ├── Lê entradas NEEDED → descobre: precisa de libc.so.6    │
│           └── Localiza e mapeia libc.so.6 (e outras .so necessárias)  
│                                                                       │
│  PASSO 9: Resolve relocações (preenche a GOT)                         │
│           Para cada função externa (ex: printf):                      │
│           ├── Encontra o endereço real de printf na libc.so.6        
│           └── Escreve esse endereço na GOT                           
│                                                                       │
│  PASSO 10: Aplica proteção GNU_RELRO                                  │
│            Chama mprotect() → .dynamic e .got → somente-leitura (R)  
│                                                                       │
│  PASSO 11: Chama funções de inicialização (.init, .init_array)        │
│            Configurações globais (ex: construtores C++)               │
│                                                                       │
│  PASSO 12: Salta para o entry point do binário principal              │
│            (o endereço em e_entry do ELF Header)                      │
└─────────────────────────────────────────────────────────────
          │
          ▼
┌─────────────────────────────────────────────────────────────
│                      BINÁRIO PRINCIPAL                                │
│                                                                       │
│  PASSO 13: Executa _start  (gerado pelo GCC/libc)                     │
│            ├── Configura o ambiente C (locale, etc.)                 
│            └── Chama __libc_start_main()                             
│                                                                       │
│  PASSO 14: __libc_start_main()                                        │
│            ├── Registra atexit() handlers                             
│            └── Chama main(argc, argv, envp)                           
│                                                                       │
│  PASSO 15: main() executa seu código                                  │
│            printf("Olá, ELF!\n")                                      │
│            return 0  
│                                                                       │
│  PASSO 16: __libc_start_main chama exit()                             │
│            ├── Chama funções de finalização (.fini, .fini_array)      
│            └── syscall exit_group() → processo termina               
└────────────────────────────────────────────────────────────┘
```

### Por que o kernel não chama main() diretamente?

```
Porque main() é uma convenção do C, não do sistema operacional.

O kernel só entende: "execute a partir deste endereço (e_entry)."
Quem chama main() é o runtime da libc (__libc_start_main).
Quem chama __libc_start_main é o _start.
Quem chama _start é o kernel (via e_entry).

Kernel → _start → __libc_start_main → main()
```


## O auxv — o bilhete secreto do kernel para o ld.so

O kernel passa informações cruciais para o dynamic linker através do **auxiliary vector (auxv)**, colocado na stack antes da execução começar.

```bash
# Ver o auxv do seu processo
LD_SHOW_AUXV=1 ./hello_64
```

Saída esperada no Pi:

```
AT_SYSINFO_EHDR: 0xffffb5670000   ← vDSO (virtual Dynamic Shared Object)
AT_HWCAP:        aa7              ← capacidades de hardware do CPU
AT_PAGESZ:       4096             ← tamanho da página: 4 KB
AT_CLKTCK:       100              ← ticks do relógio por segundo
AT_PHDR:         0x5594a2000040   ← onde estão os Program Headers na memória
AT_PHENT:        56               ← tamanho de cada Phdr entry (56 bytes = ELF64)
AT_PHNUM:        9                ← número de Program Headers
AT_BASE:         0xffffb4c90000   ← base address do ld.so
AT_FLAGS:        0x0
AT_ENTRY:        0x5594a2010640   ← entry point do binário principal
AT_UID:          1000
AT_EUID:         1000
AT_GID:          1000
AT_EGID:         1000
AT_RANDOM:       0xffffc8a4f298   ← 16 bytes aleatórios (stack canary seed)
AT_EXECFN:       ./hello_64       ← nome do executável
AT_PLATFORM:     aarch64          ← plataforma
```

> **Exercício**: compare o `AT_ENTRY` com o `e_entry` do `readelf -h`. São iguais? Devem ser, com o deslocamento da base ASLR somado.


# 9. Observando a memória em tempo real

Teoria é ótima. Mas ver a memória **ao vivo** é melhor ainda.

## Método 1 — /proc/PID/maps

```bash
# Terminal 1: execute o programa pausado com sleep ou gdb
sleep 100 &
PID=$!
cat /proc/$PID/maps
```

Ou, com seu próprio programa adicionando um `getchar()` para pausar:

```c
// hello_pause.c
#include <stdio.h>
int main(void) {
    printf("PID: %d — pressione Enter para continuar\n", getpid());
    getchar();   // pausa aqui
    printf("Olá, ELF!\n");
    return 0;
}
```

```bash
gcc -o hello_pause hello_pause.c
./hello_pause &
# Anota o PID exibido
cat /proc/<PID>/maps
```

Saída típica no Pi:

```
aaaac4560000-aaaac4561000 r--p 00000000 b3:02 12345  /home/pi/estudos_elf/hello_pause
aaaac4561000-aaaac4562000 r-xp 00010000 b3:02 12345  /home/pi/estudos_elf/hello_pause
aaaac4562000-aaaac4563000 r--p 00020000 b3:02 12345  /home/pi/estudos_elf/hello_pause
aaaac4573000-aaaac4574000 r--p 00020000 b3:02 12345  /home/pi/estudos_elf/hello_pause
aaaac4574000-aaaac4575000 rw-p 00021000 b3:02 12345  /home/pi/estudos_elf/hello_pause
...
fffff7f60000-fffff7f88000 r--p 00000000 b3:02 99999  /usr/lib/aarch64-linux-gnu/libc.so.6
fffff7f88000-fffff8100000 r-xp 00028000 b3:02 99999  /usr/lib/aarch64-linux-gnu/libc.so.6
fffff8100000-fffff8150000 r--p 001a0000 b3:02 99999  /usr/lib/aarch64-linux-gnu/libc.so.6
...
fffff813a000-fffff813c000 r--p 001d9000 b3:02 99999  /usr/lib/aarch64-linux-gnu/libc.so.6
fffff813c000-fffff813e000 rw-p 001db000 b3:02 99999  /usr/lib/aarch64-linux-gnu/libc.so.6
...
fffff8160000-fffff8190000 r--p 00000000 b3:02 88888  /usr/lib/aarch64-linux-gnu/ld-linux-aarch64.so.1
fffff8190000-fffff81c0000 r-xp 00030000 b3:02 88888  /usr/lib/aarch64-linux-gnu/ld-linux-aarch64.so.1
...
fffffffffb000-1000000000000 rwxp 00000000 00:00 0    [stack]
```

### Lendo o formato de /proc/PID/maps

```
aaaac4561000-aaaac4562000  r-xp  00010000  b3:02  12345  /home/pi/.../hello_pause
↑───────────────────┘  ↑───  ↑──────  ↑────  ↑───   ↑──────────────────────
     Range de memória      Perm.  Offset    Dev   Inode         Path do arquivo

                         Legenda das permissões:
                         r = read (leitura)
                         w = write (escrita)
                         x = execute (execução)
                         p = private (copy-on-write)
                         s = shared compartilhado)
```

> **Exercício**: compare os ranges de memória do `/proc/PID/maps` com os segmentos LOAD do `readelf -l`. Você verá que cada linha do maps corresponde a um segmento LOAD — as permissões devem bater!

## Método 2 — pmap

```bash
pmap -x <PID>
```

Mais legível que `/proc/PID/maps`:

```
Address           Kbytes     RSS   Dirty Mode  Mapping
aaaac4560000           4       4       0 r--   hello_pause
aaaac4561000           4       4       0 r-x   hello_pause
aaaac4562000           4       4       0 r--   hello_pause
aaaac4574000           4       4       4 rw-   hello_pause
...
fffff7f60000         160     160       0 r--   libc.so.6
fffff7f88000        2016    2016       0 r-x   libc.so.6
...
```

## Método 3 — strace (ver as syscalls de mmap)

```bash
strace -e mmap,mprotect ./hello_64 2>&1 | head -30
```

Você verá o kernel fazendo as chamadas `mmap()` para cada segmento LOAD, e depois o `mprotect()` do RELRO:

```
mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, 3, 0) = 0xffff...         ← LOAD[0] R
mmap(NULL, 4096, PROT_READ|PROT_EXEC, MAP_PRIVATE, 3, 0x10000) = 0xffff...  ← LOAD[1] RE
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE, 3, 0x20000) = 0xffff... ← LOAD[3] RW
mprotect(0xffff..., 2304, PROT_READ) = 0                           ← RELRO aplicado
```


# 10. Exercícios práticos

## Exercício 1 — Contando segmentos

```bash
# Quantos Program Headers tem o hello_64?
readelf -h hello_64 | grep "program headers"

# Quantos são do tipo LOAD?
readelf -l hello_64 | grep "^  LOAD"
```

## Exercício 2 — Achando a string do INTERP manualmente

```bash
# readelf já mostra, mas vamos confirmar no hex:
readelf -l hello_64 | grep "INTERP" | awk '{print $2}'
# Ex: offset = 0x238

# Ler do arquivo:
python3 -c "
with open('hello_64', 'rb') as f:
    f.seek(0x238)
    print(f.read(27))
"
```

## Exercício 3 — Comparar estático vs dinâmico

```bash
# Compilar versão estática
gcc -static -o hello_static hello.c

# Quantos LOAD tem o estático?
readelf -l hello_static | grep "^  LOAD" | wc -l

# Tem INTERP? Tem DYNAMIC?
readelf -l hello_static | grep "INTERP\|DYNAMIC"
# → Não deve ter nenhum! Por quê?
```

> **Resposta esperada**: binários estáticos não precisam de dynamic linker nem de INTERP — tudo já está embutido. Logo, sem PT_INTERP e sem PT_DYNAMIC.

## Exercício 4 — Verificar NX na stack

```bash
# NX habilitado (comportamento esperado):
readelf -l hello_64 | grep "GNU_STACK"
# Deve mostrar: RW  (sem E)

# Compilar sem NX e comparar:
gcc -o hello_nxoff hello.c -Wl,-z,execstack
readelf -l hello_nxoff | grep "GNU_STACK"
# Deve mostrar: RWE
```

## Exercício 5 — Observar ASLR em ação

```bash
# Execute duas vezes e veja os endereços mudando:
./hello_pause &; cat /proc/$!/maps | grep hello_pause | head -1; kill $!
./hello_pause &; cat /proc/$!/maps | grep hello_pause | head -1; kill $!

# Desabilitar ASLR temporariamente (precisa de sudo):
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
./hello_pause &; cat /proc/$!/maps | grep hello_pause | head -1; kill $!
./hello_pause &; cat /proc/$!/maps | grep hello_pause | head -1; kill $!
# Endereços iguais!

# Restaurar ASLR (importante!):
echo 2 | sudo tee /proc/sys/kernel/randomize_va_space
```


# 11. Recapitulando...

## O que você aprendeu neste volume:

• A diferença entre **seções** (visão do linker) e **segmentos** (visão do kernel)  
• Como usar `readelf -l` e interpretar cada coluna  
• O papel de cada tipo de segmento: PHDR, INTERP, LOAD, DYNAMIC, GNU_STACK, GNU_RELRO, NOTE  
• Por que **FileSiz < MemSiz** no LOAD de dados (seção `.bss`)  
• A notação de flags: `R`, `W`, `E` no readelf (não `X`!)  
• A diferença visual entre PIE e no-PIE nos endereços dos LOADs  
• A sequência completa: `execve` → kernel → ld.so → `_start` → `main()`  
• O que é o **auxv** e por que o kernel o passa para o ld.so  
• Como observar a memória ao vivo com `/proc/PID/maps`, `pmap` e `strace`  


## Referência rápida de comandos

```bash
# Listar Program Headers (segmentos)
readelf -l <binário>
readelf --segments <binário>      # idêntico ao acima

# Listar seções (para comparar)
readelf -S <binário>

# Ver seção DYNAMIC
readelf -d <binário>

# Ver auxv em tempo de execução
LD_SHOW_AUXV=1 ./<binário>

# Ver mapa de memória de um processo vivo
cat /proc/<PID>/maps
pmap -x <PID>

# Ver syscalls de carregamento
strace -e mmap,mprotect ./<binário>

# Verificar proteções (PIE, NX, RELRO, canary)
readelf -l <binário> | grep "GNU_STACK\|GNU_RELRO"
readelf -h <binário> | grep "Type"
```

*ELF Volume 2 — Program Headers e Segmentos*
*Ambiente: Raspberry Pi 4, ARM64 (AArch64), Raspberry Pi OS 64-bit*
