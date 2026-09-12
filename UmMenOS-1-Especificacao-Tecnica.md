# UmMenOS 1 — Especificação Técnica

**Versão do documento:** 2.1
**Nome do projeto:** UmMenOS
**Versão do sistema:** 1
**Leitura matemática:** `UmMenOS 1` = `1 − 1` = `0`
**Público-alvo:** implementação por LLM / engenheiro de sistemas
**Alvo:** x86-64 (long mode), bare metal, QEMU e hardware genérico
**Idioma de implementação:** C17 + Assembly (NASM/GAS)
**Licença:** MIT

---

## §0 Sobre o nome e a filosofia do projeto

**UmMenOS** é um nome que opera em três camadas simultâneas:

1. **Camada portuguesa:** `Um` (numeral 1) + `Men` (de "menos") + `OS` (*Operating System*).
2. **Camada matemática:** `UmMenOS N` lê-se como `1 − N`. Para `N = 1`, o resultado é `0`.
3. **Camada inglesa:** `UmMenOS` soa como *"one less OS"* — "um sistema operacional a menos".

A filosofia do projeto é coerente com o nome:

- Um OS **mínimo**, que faz apenas o necessário para ser usável.
- Um OS que se define por **subtração**, não por adição.
- Um OS que, ironicamente, se declara **zero** (`UmMenOS 1` = `1 − 1` = `0`), mas que funciona.

O nome é uma declaração de intenção: simplicidade radical, com humor.

**Convenção de versionamento:**

- `UmMenOS N`, com `N` inteiro positivo.
- Cada versão é uma nova afirmação matemática:
  - `UmMenOS 0` = `1 − 0` = `1` (unitário).
  - `UmMenOS 1` = `1 − 1` = `0` (zero).
  - `UmMenOS 2` = `1 − 2` = `−1` (deve um).
- Codinomes opcionais, sem valor vinculante: `Nada`, `Vazio`, `Nulo`, `Ausência`, `Silêncio`, `Zero`.
- Minor versions (`UmMenOS 1.1`) são permitidas por pragmatismo, com a consciência de que quebram o trocadilho puro.

---

## §1 Escopo

### §1.1 Objetivo

Construir o **UmMenOS**, um sistema operacional monousuário-multitarefa (mas com suporte a múltiplos usuários), monolítico, com:

- boot em BIOS/UEFI via Multiboot2 (GRUB) ou Limine;
- kernel em C com pontos isolados em assembly;
- gerenciamento de memória virtual com paginação;
- escalonamento preemptivo de processos e threads;
- sistema de arquivos próprio (**UmFS**) em disco ATA PIO ou VirtIO-blk;
- terminal (TTY) com disciplina de linha e sequências ANSI mínimas;
- shell interativo (`msh`) com pipes, redirecionamento e builtins;
- usuários, grupos e permissões POSIX-like;
- isolamento ring 0 / ring 3;
- chamadas de sistema estáveis e versionadas.

### §1.2 Não-objetivos (v1)

- SMP / multicore (v1 é monoprocessado; deve ser projetado para extensão futura).
- Rede (opcional; ver §10.6).
- USB.
- Áudio, gráficos acelerados.
- Drivers além de: teclado PS/2, timer PIT/HPET, serial 16550, ATA PIO, framebuffer VBE simples.
- ASLR, sandboxing avançado, capabilities finas.
- Suporte a ELF com bibliotecas dinâmicas (apenas ELF64 estático).

### §1.3 Metas de aceitação (definição de "usável")

O sistema é considerado **usável** quando, em QEMU:

1. Inicializa em < 3 segundos.
2. Apresenta prompt de login `ummenos login:` em um terminal.
3. Permite autenticar como `root` e como usuário comum.
4. `msh` executa: `ls`, `cat`, `echo`, `mkdir`, `rm`, `cp`, `mv`, `ps`, `kill`, `cd`, `pwd`, `whoami`, `login`, `shutdown`.
5. Suporta pipes (`ls | cat`), redirecionamento (`echo x > f`), background (`prog &`).
6. Persiste arquivos entre reinicializações em disco (via **UmFS**).
7. Impede que processo em ring 3 acesse memória do kernel sem causar pânico controlado.
8. Encerra o sistema com `shutdown` de forma limpa (sync + halt).

### §1.4 Licença

O projeto **DEVE** adotar a licença **MIT**. O arquivo `LICENSE` na raiz do repositório **DEVE** conter o texto padrão da MIT com o detentor dos direitos autorais. Justificativa: é permissiva, curta, compatível com uso comercial e educacional, e facilita a reutilização de trechos.

### §1.5 Requisitos não funcionais

| Requisito | Meta | Como medir |
|-----------|------|------------|
| Tempo de boot | < 3 s em QEMU com KVM | `time` no host até prompt de login |
| Latência de syscall (`getpid`) | < 1 µs | `rdtsc` em torno de `syscall` |
| Throughput de disco (ATA PIO) | > 10 MB/s leitura sequencial | `dd if=/dev/hda of=/dev/null bs=64k` |
| MTBF em uso normal | > 24 h sem pânico | Execução contínua em QEMU |
| Uso de memória em idle | < 32 MiB | `free` reportado pelo kernel |
| Tamanho do kernel (binário) | < 1 MiB | `size kernel.elf` |
| Tempo de compilação limpa | < 30 s | `make clean && time make` |

### §1.6 Análise de riscos

| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Bugs em paginação | Alta | Crítico | Testes unitários + QEMU debug + asserções |
| Corrupção de disco | Média | Alto | Cache write-back com `sync()` periódico |
| Vazamento de memória | Alta | Médio | Contadores em `allocate_kernel_memory`/`free_kernel_memory` |
| Complexidade do COW | Média | Alto | Começar com `fork` sem COW; otimizar depois |
| Feature creep | Alta | Médio | Escopo v1 congelado; backlog para v2 |
| Corrida em spinlocks | Média | Crítico | Revisão de código + testes de estresse |
| Regressão em syscalls | Média | Alto | Testes de integração na CI |

---

## §2 Arquitetura geral

```
+------------------------------------------------------+
|                    Userspace (ring 3)                |
|  init | login | msh | coreutils | libc (libum)       |
+------------------------------------------------------+
|                  Syscall ABI (int 0x80 / syscall)    |
+------------------------------------------------------+
|                    Kernel (ring 0)                   |
|  +--------+ +---------+ +--------+ +--------------+  |
|  |  VFS   | | Process | | Memory | |   Drivers    |  |
|  +--------+ +---------+ +--------+ +--------------+  |
|  |  UmFS  | | Sched   | | Pager  | | ATA | Kbd    |  |
|  |  TTY   | | Signals | | Heap   | | PIT | Serial |  |
|  |  IPC   | | Syscall | | Slab   | | FB  |        |  |
|  +--------+ +---------+ +--------+ +--------------+  |
|                   HAL (x86-64)                       |
+------------------------------------------------------+
```

**Modelo:** monolítico, kernel em higher-half, espaço de endereçamento do kernel compartilhado entre todos os processos, mapa de memória do usuário trocado a cada troca de contexto.

**Princípios:**

- Kernel nunca confia em ponteiros do usuário sem validação.
- Toda entrada de syscall é copiada com `copy_from_user` / `copy_to_user`.
- Toda estrutura compartilhada com o usuário é versionada.

---

## §3 Boot e inicialização

### §3.1 Cadeia de boot

1. **Bootloader:** Multiboot2 (GRUB2) carrega `kernel.elf` e um módulo `initrd` opcional.
2. Kernel recebe: mapa de memória Multiboot2, framebuffer info, módulos.
3. Ponto de entrada em assembly: `_start` → configura GDT, IDT, paging, pilha do kernel.
4. Salta para `kernel_main(multiboot2_information)`.

### §3.2 Requisitos do carregador

- Kernel **DEVE** ser ELF64, `ET_EXEC`, carregado em endereço físico `0x100000` e mapeado em virtual `0xFFFFFFFF80000000`.
- **DEVE** conter header Multiboot2 com tags: `FRAMEBUFFER`, `MEMORY_MAP`, `MODULE`.

### §3.3 Sequência de inicialização (obrigatória)

```
1.  Desabilitar interrupções (cli)
2.  Configurar GDT (ring 0 e ring 3)
3.  Configurar IDT com 256 entradas
4.  Configurar paging (PML4, PDPT, PD, PT) — identity map temporário
5.  Habilitar long mode e paging
6.  Remover identity map, mapear higher-half
7.  Inicializar PMM (bitmap físico)
8.  Inicializar heap do kernel (slab)
9.  Inicializar serial (COM1), framebuffer, VGA fallback
10. Inicializar PIT (100 Hz) e/ou HPET
11. Inicializar drivers de bloco (ATA/VirtIO)
12. Montar UmFS da partição primária 1 do disco 0
13. Inicializar VFS, TTY, process, scheduler
14. Criar processo `init` (PID 1) a partir de `/sbin/init`
15. Habilitar interrupções (sti)
16. Entrar no loop ocioso (hlt)
```

**Critério de aceitação:** se qualquer passo falhar, o kernel **DEVE** imprimir erro em serial e tela, e entrar em `panic()`.

### §3.4 Mapa de memória virtual do kernel

| Faixa virtual | Uso |
|---------------|-----|
| `0x0000000000000000`–`0x00007FFFFFFFFFFF` | espaço do usuário (128 TB) |
| `0xFFFF800000000000`–`0xFFFFBFFFFFFFFFFF` | mapa direto da RAM física |
| `0xFFFFFFFF80000000`–`0xFFFFFFFFFFFFFFFF` | código e dados do kernel |

### §3.5 Gerenciamento de energia (ACPI)

#### §3.5.1 Detecção

Após inicializar o PMM, o kernel **DEVE**:

1. Procurar a assinatura `"RSD PTR "` nos primeiros 1 KiB da EBDA ou na faixa `0xE0000`–`0xFFFFF`.
2. Validar o checksum.
3. Parsear RSDT/XSDT para localizar MADT (CPUs) e FADT (shutdown/reboot).

#### §3.5.2 Shutdown

Ordem de tentativas:

1. Se ACPI disponível: escrever `\_S5` no método `SMI_CMD` ou usar `RESET_REG` do FADT.
2. Se QEMU: escrever `0x2000` na porta `0x604` ou usar `isa-debug-exit`.
3. Fallback: `cli; hlt` em loop.

#### §3.5.3 Reboot

Ordem de tentativas:

1. `0xCF9` (reset via chipset).
2. Controlador de teclado 8042 (comando `0xFE`).
3. Triple fault.

#### §3.5.4 Estados de energia da CPU

- Loop ocioso **DEVE** usar `hlt`.
- Se `mwait` disponível (via CPUID), **PODE** ser usado.

---

## §4 Estilo de código, nomenclatura e documentação inline

Esta seção é **normativa**. Todo código do projeto (kernel, userspace, ferramentas) **DEVE** seguir estas regras. Desvios só são aceitos com justificativa em comentário `NOTE:` no local.

### §4.1 Princípios

1. **Legibilidade acima de brevidade.** Um nome longo e claro é preferível a um curto que exija decodificação mental.
2. **Consistência absoluta.** O mesmo conceito tem o mesmo nome em todo o código.
3. **Documentação inline é obrigatória.** Todo arquivo, struct, função e campo não-trivial tem comentário explicativo. O código deve ser entendível por leitura direta, sem consultar documentação externa.

### §4.2 Nomenclatura

#### §4.2.1 Proibição de abreviações

Palavras **NÃO DEVEM** ser abreviadas. A única exceção são siglas e acrônimos consagrados (§4.2.2).

| Proibido | Correto |
|----------|---------|
| `str` | `string` |
| `mem` | `memory` |
| `buf` | `buffer` |
| `ptr` | `pointer` |
| `len` | `length` |
| `arg` | `argument` |
| `addr` | `address` |
| `num` | `number` |
| `idx` | `index` |
| `cnt` | `count` |
| `tmp` | `temporary` |
| `msg` | `message` |
| `err` | `error` |
| `req` | `request` |
| `res` | `response` |
| `val` | `value` |
| `dest` | `destination` |
| `src` | `source` |
| `alloc` | `allocate` |
| `init` | `initialize` |
| `deinit` | `deinitialize` |
| `config` | `configuration` |
| `cmd` | `command` |
| `opt` | `option` |
| `param` | `parameter` |
| `proc` | `process` |
| `info` | `information` |
| `attr` | `attribute` |
| `pos` | `position` |
| `prev` | `previous` |
| `curr` | `current` |
| `env` | `environment` |
| `dir` | `directory` |
| `reg` | `register` |
| `desc` | `descriptor` |

A regra se aplica a **funções, variáveis, campos de struct, parâmetros, macros e tipos**.

#### §4.2.2 Siglas e acrônimos

Siglas consagradas **DEVEM** ser mantidas como sigla, tanto isoladas quanto dentro de nomes compostos. Nunca expandir para o significado literal.

**Lista canônica (não exaustiva):**

- **Hardware:** CPU, MMU, GDT, IDT, TSS, PIC, APIC, IOAPIC, PIT, HPET, RTC, ACPI, DMA, PCI, ATA, PIO, VGA, PS/2, IRQ, ELF, MADT, FADT, RSDT, XSDT, RSDP, EBDA.
- **Software:** VFS, PCB, TTY, FD, PID, PPID, UID, GID, EUID, EGID, ABI, API, IPC, COW, LRU, FIFO, LIFO, IO.
- **Sistema de arquivos:** UmFS, LBA, inode (ver nota abaixo).
- **Rede (v2):** IP, TCP, UDP, ICMP, ARP, MAC.

**Exemplos de uso:**

- `initialize_gdt()` — correto.
- `initialize_graphics_descriptor_table()` — **proibido**.
- `read_from_vga()` — correto.
- `read_from_video_graphics_array()` — **proibido**.
- `process->pid` — correto.
- `process->process_id` — aceitável, mas redundante; prefira `pid`.

**Nota sobre `inode`:** embora seja contração histórica de *index node*, é termo consolidado em sistemas de arquivos. **DEVE** ser mantido como `inode`.

#### §4.2.3 Convenções de caixa

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Funções | `snake_case`, verbo primeiro | `compare_string`, `allocate_page` |
| Variáveis | `snake_case` | `process_count`, `current_process` |
| Campos de struct | `snake_case` | `process->pid`, `inode->size` |
| Tipos (`typedef`) | `snake_case` + sufixo `_t` | `process_t`, `inode_t`, `block_device_t` |
| Macros e constantes | `UPPER_SNAKE_CASE` | `MAX_PROCESSES`, `PAGE_SIZE` |
| Valores de enum | `UPPER_SNAKE_CASE` com prefixo | `PROCESS_STATE_READY` |
| Arquivos | `snake_case.c` / `snake_case.h` | `physical_memory.c`, `process.h` |

#### §4.2.4 Estrutura de nomes de função

Formato: `verbo_objeto` ou `verbo_objeto_contexto`.

**Verbos canônicos:**

| Verbo | Uso |
|-------|-----|
| `initialize` | Inicialização de subsistema |
| `allocate` / `free` | Alocação e liberação |
| `create` / `destroy` | Ciclo de vida de objeto |
| `open` / `close` | Aquisição e liberação de recurso |
| `read` / `write` | Transferência de dados |
| `get` / `set` | Acesso a campo |
| `compare` | Comparação |
| `copy` | Duplicação |
| `find` / `search` | Busca |
| `insert` / `remove` | Modificação de coleção |
| `send` / `receive` | Comunicação |
| `handle` | Tratamento de evento |
| `check` / `validate` | Verificação |
| `convert` / `parse` | Transformação |
| `print` | Saída |

**Exemplos:**

```c
void  initialize_physical_memory_manager(void);
void *allocate_kernel_memory(size_t size);
void  free_kernel_memory(void *pointer);
int   compare_string(const char *first, const char *second);
int   print_formatted(const char *format, ...);
void  copy_memory(void *destination, const void *source, size_t count);
void  set_memory(void *destination, int value, size_t count);
size_t length_string(const char *string);
pid_t process_create(process_t *parent);
void  process_destroy(process_t *process);
int   signal_send(process_t *process, int signal_number);
```

**Evitar:**

- Nomes genéricos: `do_thing()`, `handle()`, `run()`.
- Prefixos redundantes: `function_compare_string()`.
- Sufixos redundantes: `compare_string_function()`.
- Negativas: `not_ready()` → prefira `is_ready()` com retorno booleano.

#### §4.2.5 Prefixos de namespace

| Subsistema | Prefixo | Exemplo |
|------------|---------|---------|
| Memória física | `physical_memory_` ou `pmm_` | `pmm_allocate_frame()` |
| Memória virtual | `virtual_memory_` ou `vmm_` | `vmm_map_page()` |
| Heap do kernel | `kernel_memory_` | `kernel_memory_allocate()` |
| Processos | `process_` | `process_create()` |
| Threads | `thread_` | `thread_create()` |
| Escalonador | `scheduler_` | `scheduler_tick()` |
| Sinais | `signal_` | `signal_send()` |
| VFS | `vfs_` | `vfs_open()` |
| Inodes | `inode_` | `inode_read()` |
| Diretórios | `directory_` | `directory_lookup()` |
| Bloco | `block_` | `block_read()` |
| UmFS | `umfs_` | `umfs_mount()` |
| TTY | `tty_` | `tty_write()` |
| Teclado | `keyboard_` | `keyboard_handle_interrupt()` |
| Serial | `serial_` | `serial_write()` |
| Framebuffer | `framebuffer_` | `framebuffer_draw_character()` |
| ATA | `ata_` | `ata_read_sector()` |
| ACPI | `acpi_` | `acpi_shutdown()` |
| Syscalls | `syscall_` | `syscall_open()` |

**Regra:** se o subsistema é conhecido por sigla (PMM, VMM, VFS, ACPI, ATA), usar a sigla como prefixo. Se é palavra comum (process, thread, signal), usar a palavra por extenso.

### §4.3 Comentários e documentação inline

#### §4.3.1 Regra geral

Todo elemento não-trivial **DEVE** ter comentário. "Não-trivial" significa:

- Qualquer função (exceto getters/setters de uma linha).
- Qualquer struct e seus campos cuja função não seja óbvia pelo nome.
- Qualquer macro ou valor de enum não autoexplicativo.
- Qualquer bloco de código com lógica não-imediata.
- Qualquer invariante mantida por um módulo.

#### §4.3.2 Cabeçalho de arquivo

Todo arquivo `.c` e `.h` **DEVE** começar com:

```c
/**
 * @file    physical_memory.c
 * @brief   Gerenciador de memória física (PMM).
 *
 * Mantém um bitmap de frames de 4 KiB. Um bit em 1 indica frame em uso;
 * um bit em 0 indica frame livre. O frame 0 nunca é alocado (reservado
 * para o BIOS/real mode).
 *
 * Invariantes:
 *   - O bitmap cobre toda a RAM detectada no boot.
 *   - Regiões do kernel e módulos Multiboot2 estão marcadas como usadas.
 *   - Toda alocação é atômica (protegida por spinlock).
 */
```

#### §4.3.3 Cabeçalho de função

```c
/**
 * Compara duas strings terminadas em NUL, byte a byte.
 *
 * @param first   Primeira string.
 * @param second  Segunda string.
 * @return        Valor negativo se first < second;
 *                zero se first == second;
 *                valor positivo se first > second.
 *
 * @note Não considera locale. Para ordenação lexicográfica com
 *       locale, usar uma função futura compare_string_localized().
 */
int compare_string(const char *first, const char *second);
```

Para funções triviais, uma linha basta:

```c
/** Retorna o PID do processo. */
pid_t process_get_id(const process_t *process);
```

#### §4.3.4 Cabeçalho de struct

```c
/**
 * Bloco de Controle de Processo (PCB).
 *
 * Representa um processo no sistema. Um processo possui:
 *   - um espaço de endereçamento próprio (PML4);
 *   - uma tabela de descritores de arquivo;
 *   - credenciais (UID, GID, EUID, EGID);
 *   - estado de escalonamento e sinais pendentes.
 *
 * O ciclo de vida é gerenciado por process_create() e process_destroy().
 * O reference_count permite que múltiplas referências existam (pai,
 * filhos, tabela de processos global).
 */
typedef struct process {
    pid_t pid;                       /**< Identificador único. */
    pid_t parent_pid;                /**< PID do pai (0 se for init). */
    uid_t user_id;                   /**< UID real. */
    uid_t effective_user_id;         /**< UID efetivo (usado em permissões). */
    gid_t group_id;                  /**< GID real. */
    gid_t effective_group_id;        /**< GID efetivo. */
    /* ... */
} process_t;
```

#### §4.3.5 Comentários inline

- Use `/* ... */` ou `// ...` com moderação.
- Explique **por que**, não **o que**.
- **Ruim:** `index++; // incrementa index`
- **Bom:** `index++; // avança para o próximo frame livre no bitmap`

#### §4.3.6 Marcação de pendências

| Marca | Uso |
|-------|-----|
| `TODO(nome):` | Algo a implementar |
| `FIXME(nome):` | Bug conhecido |
| `NOTE:` | Observação importante |
| `WARNING:` | Armadilha ou comportamento não-óbvio |
| `HACK:` | Solução temporária |

```c
/* TODO(allek): suportar páginas de 2 MiB para reduzir uso de memória. */
/* FIXME(allek): race condition se dois processos fizerem fork simultâneo. */
/* WARNING: esta função não pode ser chamada com interrupções habilitadas. */
```

### §4.4 Tabela comparativa (antes / depois)

| Antes (proibido) | Depois (obrigatório) |
|------------------|----------------------|
| `strcmp(first, second)` | `compare_string(first, second)` |
| `printf(format, ...)` | `print_formatted(format, ...)` |
| `memcpy(destination, source, count)` | `copy_memory(destination, source, count)` |
| `memset(destination, value, count)` | `set_memory(destination, value, count)` |
| `strlen(string)` | `length_string(string)` |
| `kmalloc(size)` | `allocate_kernel_memory(size)` |
| `kfree(pointer)` | `free_kernel_memory(pointer)` |
| `init_pmm()` | `initialize_physical_memory_manager()` |
| `alloc_frame()` | `physical_memory_allocate_frame()` |
| `printk(...)` | `print_kernel(...)` |
| `struct proc { ... }` | `struct process { ... }` |
| `p->pid` | `process->pid` |
| `mmap(...)` | `map_memory_region(...)` |
| `init_tty()` | `initialize_terminal()` |
| `dev` | `device` |
| `blkdev` | `block_device` |
| `dentry` | `directory_entry` |
| `attr` | `attribute` |
| `argv` | `argument_vector` (em contexto interno; `argv` é padrão POSIX) |

**Exceção:** nomes de syscalls POSIX (`open`, `read`, `write`, `close`, `fork`, `execve`, `mmap`, `brk`) **DEVEM** permanecer como estão — são ABI pública. Os handlers internos usam prefixo `syscall_` e seguem o estilo: `syscall_open`, `syscall_read` etc.

### §4.5 Ferramentas de verificação

- **`clang-format`** com arquivo `.clang-format` na raiz, baseado em LLVM style, indentação 4 espaços, limite 100 colunas.
- **`clang-tidy`** com checks habilitados para legibilidade e segurança.
- **`cppcheck`** em CI.
- **`grep` no CI** para detectar violações óbvias.

Exemplo de regra de CI:

```bash
if grep -rnE '\b(str[a-z]|mem[a-z]|ptr|buf|len|alloc|init)\b' src/ include/; then
    echo "Violação de nomenclatura detectada."
    exit 1
fi
```

---

## §5 Gerenciamento de memória

### §5.1 Tipos base e convenções

```c
typedef unsigned char      u8;
typedef unsigned short     u16;
typedef unsigned int       u32;
typedef unsigned long long u64;
typedef signed long long   i64;
typedef u64                size_t;
typedef i64                ssize_t;
typedef u64                paddr_t;   // endereço físico
typedef u64                vaddr_t;   // endereço virtual
typedef i32                pid_t;
typedef u32                uid_t;
typedef u32                gid_t;
typedef i32                fd_t;
```

- Todos os structs em disco ou compartilhados com userspace **DEVEM** ter `_Static_assert` de tamanho.
- Ordenação little-endian, alinhamento natural (sem padding implícito — usar `__attribute__((packed))` quando o layout for fixo).

### §5.2 PMM (Physical Memory Manager)

- **Granularidade:** 4 KiB.
- **Estrutura:** bitmap, 1 bit por frame.
- **Funções:**

```c
paddr_t physical_memory_allocate_frame(void);   // retorna 0 se OOM
void    physical_memory_free_frame(paddr_t frame);
void    physical_memory_mark_used(paddr_t address, size_t size);
void    physical_memory_mark_free(paddr_t address, size_t size);
```

- **Invariantes:** frame 0 nunca alocado; região do kernel e módulos Multiboot2 marcados como usados.

### §5.3 VMM (Virtual Memory Manager)

- **Paginação:** 4 níveis (PML4 → PDPT → PD → PT), páginas de 4 KiB.
- **Funções:**

```c
void    initialize_virtual_memory(void);
void    virtual_memory_map(pml4_t *page_map_level_4, vaddr_t virtual_address,
                           paddr_t physical_address, u32 flags);
void    virtual_memory_unmap(pml4_t *page_map_level_4, vaddr_t virtual_address);
paddr_t virtual_memory_translate(pml4_t *page_map_level_4, vaddr_t virtual_address);
pml4_t *virtual_memory_create_address_space(void);
void    virtual_memory_destroy_address_space(pml4_t *page_map_level_4);
```

- **Flags:** `PAGE_PRESENT`, `PAGE_WRITE`, `PAGE_USER`, `PAGE_NO_EXECUTE`, `PAGE_COPY_ON_WRITE`, `PAGE_DIRTY`.

### §5.4 Heap do kernel

- Alocador **slab** para objetos pequenos (< 2 KiB) + **buddy** para páginas.
- Interface:

```c
void *allocate_kernel_memory(size_t size);
void *allocate_kernel_memory_zeroed(size_t size);
void *reallocate_kernel_memory(void *pointer, size_t new_size);
void  free_kernel_memory(void *pointer);
```

- **Alinhamento mínimo:** 16 bytes.
- Deve suportar detecção de corrupção (magic number no cabeçalho) em builds `DEBUG`.

### §5.5 Espaço do usuário

Layout fixo por processo:

| Faixa | Conteúdo |
|-------|----------|
| `0x0000000000400000` | código ELF (RX) |
| ... | dados/bss (RW) |
| ... | heap (cresce via `brk`/`mmap`) |
| `0x00007F0000000000` | pilha principal (8 MiB, cresce para baixo) |
| `0x00007FFFF0000000` | vDSO (opcional) |

- `brk` implementado, `mmap` apenas anônimo (v1).
- Stack guard page (não mapeada) abaixo da pilha.

---

## §6 Interrupções, exceções e timer

### §6.1 IDT

- 256 entradas, preenchidas em `initialize_interrupt_descriptor_table()`.
- Exceções 0–31 com handlers nomeados; interrupções 32+ para IRQs.

### §6.2 Exceções tratadas

| # | Nome | Ação |
|---|------|------|
| 0 | Divide Error | Envia SIGFPE ao processo |
| 6 | Invalid Opcode | SIGILL |
| 8 | Double Fault | panic + dump |
| 13 | General Protection Fault | SIGSEGV (se origem em ring 3) senão panic |
| 14 | Page Fault | Demanda de página, COW, ou SIGSEGV |

### §6.3 Timer

- **Fonte primária:** PIT a **100 Hz** (período 10 ms).
- **Fallback:** HPET se disponível.
- Cada tick:
  1. incrementa `jiffies` (u64);
  2. acorda processos adormecidos (`sleep`);
  3. decrementa quantum do processo corrente;
  4. se quantum = 0, chama `scheduler_tick()`.

### §6.4 Relógio de parede

- RTC (CMOS) lido no boot para inicializar `wall_time`.
- `gettimeofday` retorna `{wall_time + jiffies/HZ, ns}`.

### §6.5 SMP e concorrência

#### §6.5.1 Detecção

- Parsear MADT para contar CPUs.
- Armazenar em `cpu_information_t` (uma por CPU).

#### §6.5.2 Modo monoprocessado (v1)

- Apenas a CPU 0 (BSP) executa o kernel.
- APs **DEVEM** ser colocadas em `hlt` via INIT-SIPI-SIPI, ou ignoradas se o bootloader já as desabilitou.

#### §6.5.3 Primitivas de sincronização

- `spinlock_t`: implementado com `lock cmpxchg` ou `xchg`.
- `atomic_int`: usar `__atomic_*` do GCC/Clang.
- `spin_lock_irqsave` / `spin_unlock_irqrestore`: desabilitam interrupções ao adquirir.

#### §6.5.4 Preparação para SMP futuro

- Estruturas globais **DEVEM** ser protegidas por spinlock.
- Variáveis por CPU **DEVEM** usar `__thread` ou `.data..percpu`.
- Documentar que migrar para SMP exigirá reescrita do escalonador.

---

## §7 Processos e threads

### §7.1 Modelo

- **Processo:** unidade de espaço de endereçamento + tabela de FDs + credenciais.
- **Thread:** unidade de escalonamento dentro do processo.
- Um processo tem no mínimo 1 thread.
- v1 **PODE** restringir criação de threads extras via syscall `clone` (opcional), mas o modelo interno **DEVE** ser thread-aware.

### §7.2 Estados

```
NEW → READY → RUNNING → (ZOMBIE | TERMINATED)
                  ↓
               BLOCKED (I/O, sleep, wait, pipe)
```

### §7.3 PCB (Process Control Block)

```c
/**
 * Bloco de Controle de Processo (PCB).
 *
 * Ver §4.3.4 para descrição completa.
 */
typedef struct process {
    pid_t pid;                       /**< Identificador único. */
    pid_t parent_pid;                /**< PID do pai (0 se init). */
    uid_t user_id;                   /**< UID real. */
    uid_t effective_user_id;         /**< UID efetivo. */
    gid_t group_id;                  /**< GID real. */
    gid_t effective_group_id;        /**< GID efetivo. */
    int   state;                     /**< Estado (ver PROCESS_STATE_*). */
    pml4_t *page_map_level_4;        /**< Espaço de endereçamento. */
    vaddr_t program_break;           /**< Fim do segmento de dados. */
    file_descriptor_entry_t file_descriptors[FILE_DESCRIPTOR_MAX];
    inode_t *current_working_directory;
    signal_action_t signal_actions[SIGNAL_COUNT];
    u64 pending_signals;             /**< Bitmap de sinais pendentes. */
    u64 blocked_signals;             /**< Bitmap de sinais bloqueados. */
    int exit_code;                   /**< Código de saída. */
    u64 start_time;                  /**< jiffies no momento da criação. */
    u64 cpu_time;                    /**< jiffies consumidos. */
    char name[PROCESS_NAME_MAX];     /**< Nome legível. */
    struct process *parent;          /**< Processo pai. */
    struct list_head children;       /**< Lista de filhos. */
    struct list_head siblings;       /**< Lista de irmãos. */
    spinlock_t lock;                 /**< Protege campos mutáveis. */
    atomic_int reference_count;      /**< Contagem de referências. */
} process_t;
```

### §7.4 Escalonador

- **Tipo:** round-robin com prioridade estática (3 níveis: `HIGH`, `NORMAL`, `LOW`).
- **Quantum:** 10 ticks (100 ms) para `NORMAL`; 5 para `HIGH`; 20 para `LOW`.
- **Fila:** uma fila circular por nível de prioridade, com aging de 1 nível a cada 500 ticks.
- **Troca de contexto:** salva/restaura `rsp`, `rip`, `cr3`, `rbx`, `rbp`, `r12`–`r15`.
- **Idle:** se nenhum processo em READY, executa `hlt`.

### §7.5 Criação e término

- `fork()`: cópia de espaço de endereçamento com **copy-on-write**.
- `execve(path, argument_vector, environment_vector)`: substitui imagem; **DEVE** liberar páginas antigas.
- `exit(code)`: fecha FDs, libera memória, vira ZOMBIE até `wait()` do pai; se o pai morreu, reparent para PID 1.
- `wait()`: bloqueia até filho terminar; retorna status.

### §7.6 Sinais

| Sinal | Nº | Ação padrão |
|-------|----|-------------|
| SIGHUP | 1 | Terminar |
| SIGINT | 2 | Terminar |
| SIGQUIT | 3 | Terminar + core |
| SIGKILL | 9 | Terminar (não capturável) |
| SIGSEGV | 11 | Terminar + core |
| SIGPIPE | 13 | Terminar |
| SIGTERM | 15 | Terminar |
| SIGCHLD | 17 | Ignorar |
| SIGCONT | 18 | Continuar |
| SIGSTOP | 19 | Parar (não capturável) |

- `kill(pid, signal_number)`, `signal(signal_number, handler)`, `sigaction`, `sigprocmask`, `sigsuspend`.
- Entrega: ao retornar de interrupção/timer, se há sinal pendente não bloqueado, empilha frame no espaço do usuário.

---

## §8 Chamadas de sistema (ABI)

### §8.1 Convenção

- Instrução: `syscall` (preferida) com fallback `int 0x80`.
- Registradores: `rax` = número, `rdi, rsi, rdx, r10, r8, r9` = argumentos.
- Retorno em `rax`; erros codificados como `-errno`.
- `syscall` **DEVE** preservar todos os registradores exceto `rax`, `rcx`, `r11`.

### §8.2 Tabela de syscalls (v1)

A coluna **ABI** é o contrato binário com userspace e não muda. A coluna **Handler** é o nome interno no kernel, seguindo §4.

| # | ABI (POSIX) | Handler | Assinatura ABI |
|---|-------------|---------|----------------|
| 0 | `exit` | `syscall_exit` | `void exit(int status)` |
| 1 | `fork` | `syscall_fork` | `pid_t fork(void)` |
| 2 | `execve` | `syscall_execute` | `int execve(const char *path, char **argv, char **envp)` |
| 3 | `wait4` | `syscall_wait` | `pid_t wait4(pid_t, int *status, int, void*)` |
| 4 | `getpid` | `syscall_get_process_id` | `pid_t getpid(void)` |
| 5 | `getppid` | `syscall_get_parent_process_id` | `pid_t getppid(void)` |
| 6 | `sleep` | `syscall_sleep` | `int sleep(u32 milliseconds)` |
| 7 | `read` | `syscall_read` | `ssize_t read(int fd, void *buffer, size_t count)` |
| 8 | `write` | `syscall_write` | `ssize_t write(int fd, const void *buffer, size_t count)` |
| 9 | `open` | `syscall_open` | `int open(const char *path, int flags, mode_t mode)` |
| 10 | `close` | `syscall_close` | `int close(int fd)` |
| 11 | `lseek` | `syscall_seek` | `off_t lseek(int fd, off_t offset, int whence)` |
| 12 | `stat` | `syscall_stat` | `int stat(const char *path, struct stat *)` |
| 13 | `fstat` | `syscall_file_stat` | `int fstat(int fd, struct stat *)` |
| 14 | `unlink` | `syscall_unlink` | `int unlink(const char *path)` |
| 15 | `mkdir` | `syscall_create_directory` | `int mkdir(const char *path, mode_t)` |
| 16 | `rmdir` | `syscall_remove_directory` | `int rmdir(const char *path)` |
| 17 | `readdir` | `syscall_read_directory` | `int readdir(int fd, struct dirent *)` |
| 18 | `chdir` | `syscall_change_directory` | `int chdir(const char *path)` |
| 19 | `getcwd` | `syscall_get_working_directory` | `char *getcwd(char *buffer, size_t size)` |
| 20 | `chmod` | `syscall_change_mode` | `int chmod(const char *path, mode_t)` |
| 21 | `chown` | `syscall_change_owner` | `int chown(const char *path, uid_t, gid_t)` |
| 22 | `dup` | `syscall_duplicate` | `int dup(int fd)` |
| 23 | `dup2` | `syscall_duplicate_to` | `int dup2(int old, int new)` |
| 24 | `pipe` | `syscall_pipe` | `int pipe(int fds[2])` |
| 25 | `brk` | `syscall_set_program_break` | `void *brk(void *address)` |
| 26 | `mmap` | `syscall_map_memory` | `void *mmap(void *address, size_t length, int protection, int flags, int fd, off_t offset)` |
| 27 | `munmap` | `syscall_unmap_memory` | `int munmap(void *address, size_t length)` |
| 28 | `kill` | `syscall_send_signal` | `int kill(pid_t, int signal_number)` |
| 29 | `signal` | `syscall_set_signal_handler` | `sighandler_t signal(int, sighandler_t)` |
| 30 | `sigprocmask` | `syscall_set_signal_mask` | `int sigprocmask(int, const sigset_t*, sigset_t*)` |
| 31 | `getuid` | `syscall_get_user_id` | `uid_t getuid(void)` |
| 32 | `geteuid` | `syscall_get_effective_user_id` | `uid_t geteuid(void)` |
| 33 | `getgid` | `syscall_get_group_id` | `gid_t getgid(void)` |
| 34 | `setuid` | `syscall_set_user_id` | `int setuid(uid_t)` |
| 35 | `ioctl` | `syscall_io_control` | `int ioctl(int fd, unsigned long request, void *argument)` |
| 36 | `gettimeofday` | `syscall_get_time_of_day` | `int gettimeofday(struct timeval *, void *)` |
| 37 | `reboot` | `syscall_reboot` | `int reboot(int command)` |
| 38 | `sync` | `syscall_sync` | `void sync(void)` |
| 39 | `yield` | `syscall_yield` | `void yield(void)` |
| 40 | `getdents` | `syscall_get_directory_entries` | `int getdents(int fd, void *buffer, size_t size)` |

### §8.3 Validação de argumentos

Toda syscall **DEVE**:

1. Verificar que ponteiros do usuário estão em `[0x400000, 0x00007FFFFFFFFFFF]`.
2. Copiar dados com `copy_from_user` / `copy_to_user` (nunca dereferenciar direto).
3. Retornar `-EFAULT` em falha de cópia.
4. Verificar permissões antes de operações sensíveis.

### §8.4 errno

| Valor | Nome | Significado |
|-------|------|-------------|
| 1 | EPERM | Operação não permitida |
| 2 | ENOENT | Arquivo/diretório não existe |
| 5 | EIO | Erro de I/O |
| 9 | EBADF | FD inválido |
| 11 | EAGAIN | Recurso temporariamente indisponível |
| 12 | ENOMEM | Memória insuficiente |
| 13 | EACCES | Permissão negada |
| 14 | EFAULT | Endereço inválido |
| 17 | EEXIST | Já existe |
| 20 | ENOTDIR | Não é diretório |
| 21 | EISDIR | É um diretório |
| 22 | EINVAL | Argumento inválido |
| 28 | ENOSPC | Sem espaço |
| 32 | EPIPE | Pipe quebrado |
| 36 | ENAMETOOLONG | Nome muito longo |
| 38 | ENOSYS | Syscall não implementada |
| 39 | ENOTEMPTY | Diretório não vazio |

### §8.5 Caminho de headers públicos

Headers públicos do userspace ficam em:

```
include/ummenos/syscall.h
include/ummenos/types.h
include/ummenos/stat.h
include/ummenos/signal.h
include/ummenos/time.h
include/ummenos/ioctl.h
```

Uso no userspace:

```c
#include <ummenos/syscall.h>
#include <ummenos/stat.h>
```

---

## §9 Armazenamento e sistema de arquivos

### §9.1 Camada de bloco

- Dispositivo de bloco com setor de 512 bytes; UmFS usa blocos de 4096 bytes (8 setores).
- Interface:

```c
typedef struct block_device {
    char name[32];
    u64  number_of_blocks;      // em blocos de 4096
    int  (*read)(struct block_device*, u64 lba, void *buffer, size_t count);
    int  (*write)(struct block_device*, u64 lba, const void *buffer, size_t count);
    void *private_data;
} block_device_t;
```

- Driver ATA PIO: canais primário/secundário, master/slave, LBA28.
- Cache de blocos: LRU, 256 entradas de 4096 B (1 MiB). Escrita **write-back** com `sync()`.

### §9.2 UmFS — layout em disco

**Bloco 0:** reservado para boot (não usado pelo UmFS).
**Bloco 1:** superbloco.

#### §9.2.1 Superbloco (4096 bytes)

| Offset | Tam | Campo | Valor |
|--------|-----|-------|-------|
| 0 | 4 | `magic` | `0x554D4653` ("UMFS") |
| 4 | 4 | `version` | `1` |
| 8 | 4 | `block_size` | `4096` |
| 12 | 8 | `total_blocks` | |
| 20 | 8 | `inode_count` | |
| 28 | 8 | `inode_table_lba` | |
| 36 | 8 | `inode_table_length` | em blocos |
| 44 | 8 | `inode_bitmap_lba` | |
| 52 | 8 | `inode_bitmap_length` | |
| 60 | 8 | `data_bitmap_lba` | |
| 68 | 8 | `data_bitmap_length` | |
| 76 | 8 | `data_region_lba` | |
| 84 | 8 | `root_inode` | tipicamente `1` |
| 92 | 8 | `free_blocks` | |
| 100 | 8 | `free_inodes` | |
| 108 | 8 | `modification_time` | epoch Unix |
| 116 | 3980 | `reserved` | zeros |

#### §9.2.2 Inode (256 bytes)

```c
/**
 * Inode do UmFS.
 *
 * Total: 256 bytes. Descreve um arquivo, diretório ou dispositivo
 * especial. O campo mode segue a convenção POSIX (S_IFMT + permissões).
 */
typedef struct umfs_inode {
    u16 mode;                     /**< Tipo + permissões. */
    u16 link_count;               /**< Número de links. */
    u32 user_id;                  /**< UID do dono. */
    u32 group_id;                 /**< GID do dono. */
    u32 flags;                    /**< Flags reservadas. */
    u64 size;                     /**< Tamanho em bytes. */
    u64 access_time;              /**< Último acesso. */
    u64 modification_time;        /**< Última modificação. */
    u64 change_time;              /**< Última mudança de metadados. */
    u64 block_count;              /**< Blocos alocados. */
    u64 direct_blocks[8];         /**< Blocos diretos. */
    u64 indirect_block;           /**< Bloco indireto simples. */
    u64 double_indirect_block;    /**< Bloco duplamente indireto. */
    u8  reserved[120];            /**< Zeros. */
} umfs_inode_t;
```

- Capacidade máxima: 8 × 4 KiB + 1024 × 4 KiB + 1024² × 4 KiB = 4 GiB + 32 KiB.
- `mode`: `S_IFMT 0xF000`, `S_IFREG 0x8000`, `S_IFDIR 0x4000`, `S_IFCHR 0x2000`, `S_IFIFO 0x1000`; permissões nos 9 bits baixos.

#### §9.2.3 Entrada de diretório (128 bytes)

```c
/**
 * Entrada de diretório do UmFS.
 *
 * Total: 128 bytes. Diretórios são arquivos cujo conteúdo é uma
 * sequência dessas entradas.
 */
typedef struct umfs_directory_entry {
    u32 inode;                    /**< Número do inode (0 = livre). */
    u8  type;                     /**< Tipo (ver UMFS_TYPE_*). */
    u8  name_length;              /**< Comprimento do nome. */
    u8  name[122];                /**< Nome, sem NUL. */
} umfs_directory_entry_t;
```

- Diretórios são arquivos cujo conteúdo é uma sequência de entradas.
- `.` e `..` **DEVEM** existir como entradas inode 1 e inode do pai.

### §9.3 VFS

- Camada de abstração sobre UmFS (e futuros FS).
- Estruturas: `inode_t`, `directory_entry_t`, `file_t`.
- Operações de inode: `lookup`, `create`, `mkdir`, `unlink`, `read`, `write`, `truncate`, `read_directory`.
- Resolução de caminho: absoluto (`/`) ou relativo (`current_working_directory`).
- Máximo de componentes por caminho: 32; comprimento total: 4096.

### §9.4 Permissões

- Checagem clássica `rwx` para `owner`, `group`, `other`.
- `root` (UID 0) ignora permissões exceto o bit de execução para arquivos não-executáveis.
- Diretórios: `r` permite listar, `w` permite criar/remover, `x` permite atravessar.

### §9.5 Estrutura de diretórios padrão

```
/
├── bin/        → utilitários essenciais
├── sbin/       → init, login, shutdown
├── dev/        → dispositivos (nós especiais)
│   ├── console
│   ├── tty0
│   ├── null
│   ├── zero
│   └── hda
├── etc/
│   ├── passwd
│   └── group
├── home/
│   └── <user>/
├── tmp/
└── var/
    └── log/
```

---

## §10 Dispositivos e drivers

### §10.1 Serial (COM1)

- 115200 8N1, IRQ 4.
- Saída usada para log do kernel e console remoto.
- **DEVE** funcionar antes de qualquer driver complexo (para debug).

### §10.2 Framebuffer

- Modo texto VGA 80×25 quando disponível; caso contrário, framebuffer linear do Multiboot2.
- Renderização de fonte bitmap 8×16 embutida.
- Suporte a cursor, scroll, cores ANSI (8 cores + brilho).

### §10.3 Teclado PS/2

- IRQ 1, scancode set 1.
- Tradução para ASCII com shift/caps.
- Combinações: `Ctrl+C` → SIGINT ao foreground, `Ctrl+D` → EOF, `Ctrl+Z` → SIGSTOP.

### §10.4 ATA PIO

- Modo LBA28, PIO read/write de setores.
- Detecção de drives master/slave em canais primário/secundário.
- Timeouts de 5 s; erros reportados via `EIO`.

### §10.5 Console e dispositivos especiais

- `/dev/null`: descarta escritas, retorna EOF em leituras.
- `/dev/zero`: retorna zeros.
- `/dev/console`: terminal principal (tty0).

### §10.6 Pilha de rede (opcional)

Se for implementar:

- **Loopback:** interface `loopback` com IP `127.0.0.1`.
- **ARP:** resolução de endereços MAC.
- **IP:** IPv4 apenas, sem fragmentação.
- **ICMP:** echo request/reply (ping).
- **UDP:** sockets simples.
- **Driver:** NE2000 ou RTL8139 (QEMU suporta ambos).

Se **não** for implementar, listar explicitamente como não-objetivo em §1.2.

---

## §11 Terminal (TTY)

### §11.1 Responsabilidade

- Recebe bytes do teclado (ou serial) e do processo, aplica disciplina de linha, gerencia buffer e eco.

### §11.2 Modos

- **Canônico** (padrão): entrada acumulada até `\n`; permite edição (backspace, `Ctrl+U`, `Ctrl+W`).
- **Raw**: sem processamento (usado por editores futuros).

### §11.3 Sequências ANSI mínimas

| Sequência | Ação |
|-----------|------|
| `\e[K` | Apaga até fim da linha |
| `\e[2J` | Limpa tela |
| `\e[H` | Cursor em (0,0) |
| `\e[<r>;<c>H` | Posiciona cursor |
| `\e[3<m>` | Cor de texto |
| `\e[0m` | Reset |
| `\b` | Backspace |
| `\r` | Retorno de carro |
| `\n` | Nova linha |

### §11.4 Interface

```c
typedef struct terminal {
    char     input_buffer[4096];
    size_t   input_head, input_tail;
    char     output_buffer[8192];
    size_t   output_head, output_tail;
    u16      cursor_x, cursor_y;
    u8       attribute;
    int      mode;               // CANONICAL | RAW
    pid_t    foreground_process_group;
    wait_queue_t read_wait;
    wait_queue_t write_wait;
} terminal_t;
```

- `terminal_read` bloqueia se buffer vazio (canônico).
- `terminal_write` enfileira e agenda flush no timer.
- `ioctl(TIOCGWINSZ)` retorna `{rows=25, cols=80}`.

### §11.5 Internacionalização (i18n)

#### §11.5.1 Terminal

- Suporte a UTF-8.
- Fonte bitmap **DEVE** incluir caracteres latinos (á, é, í, ó, ú, ã, õ, ç).

#### §11.5.2 Teclado

- Layouts ABNT2 (brasileiro) e US.
- Alternância via `load_keyboard_map`.

#### §11.5.3 Mensagens

- Sistema de mensagens localizáveis.
- Locale padrão: `pt_BR.UTF-8` ou `C`.

---

## §12 Shell (`msh`)

### §12.1 Funcionalidades obrigatórias

- Prompt configurável: `user@ummenos:cwd$ `.
- Parsing de linha com tokenização por espaço (aspas `"` e `'` suportadas).
- Execução de binários externos via `fork` + `execve`.
- Builtins: `cd`, `pwd`, `exit`, `echo`, `help`, `export`, `unset`, `set`.
- Redirecionamento: `>` (truncar), `>>` (append), `<`.
- Pipes: `command1 | command2 | command3` (múltiplos).
- Sequenciamento: `;`, `&&`, `||`.
- Background: `&`.
- Expansão de variáveis: `$VAR` e `${VAR}`.
- Comentários: `#` até fim de linha.
- Histórico em memória (últimas 128 linhas); setas ↑/↓.

### §12.2 Estrutura de execução

1. Lê linha via `read_line` (usa `read` do TTY).
2. Parseia em `pipeline_t { command_t *commands; int count; }`.
3. Para cada comando, faz `fork`. Aplica redirecionamentos com `dup2`. Conecta pipes.
4. `execve` no filho; pai espera (se foreground).
5. `Ctrl+C` envia SIGINT ao grupo em foreground.

### §12.3 Códigos de retorno

- `0` sucesso, `1` erro genérico, `2` erro de sintaxe, `127` comando não encontrado, `126` não executável.

---

## §13 Usuários, grupos e autenticação

### §13.1 Arquivos de identidade

`/etc/passwd` (texto, uma linha por usuário):

```
nome:uid:gid:gecos:home:shell
root:0:0:root:/root:/bin/msh
user:1000:1000:User:/home/user:/bin/msh
```

`/etc/group`:

```
nome:gid:membros
root:0:
user:1000:
```

### §13.2 Senhas

`/etc/shadow` (modo 0600):

```
nome:algoritmo:salt:hash
```

- Algoritmo `1` = SHA-256(salt || senha), salt de 16 bytes em hex.
- Comparação em tempo constante.

### §13.3 `init` (PID 1)

1. Monta `/proc` (v1 pode ser estático).
2. Abre `/dev/console`.
3. Executa `login` como processo filho.
4. Se `login` sair com código 0, reinicia; se `shutdown` solicitado, faz `sync` e `reboot(RB_POWER_OFF)`.

### §13.4 `login`

1. Solicita usuário e senha.
2. Verifica contra `/etc/shadow`.
3. Em sucesso: `setgid`, `setuid`, `chdir(home)`, `execve(shell)`.
4. Em falha: 3 tentativas, depois atraso de 3 s.

---

## §14 Segurança

### §14.1 Isolamento

- Ring 3 para todo userspace; syscall gate via `syscall`/`sysret`.
- Kernel higher-half inacessível de ring 3 (bit U/S = 0 nas PTEs do kernel).
- Cada processo tem seu próprio PML4; kernel mapeado em todos.

### §14.2 Validação

- Toda string de caminho copiada via `copy_from_user` antes de uso.
- Comprimento máximo validado antes de alocação.
- Nenhuma syscall confia em `argument_vector`/`environment_vector` sem copiar.

### §14.3 Permissões de arquivo

- Checagem em `open`, `execve`, `unlink`, `mkdir`, `chmod`, `chown`.
- `setuid`/`setgid` só permitido para root (v1 não tem binários setuid).

### §14.4 Proteção de memória

- Stack guard page.
- Páginas de código do usuário mapeadas RX (NX para dados).
- COW para isolamento entre pai e filho.

### §14.5 Boas práticas de kernel

- `panic()` imprime registradores, stack trace e halt.
- Asserções em build DEBUG.
- Log estruturado em serial: `[nivel] subsistema: mensagem`.

### §14.6 NÃO implementado em v1 (documentar como risco)

- ASLR, canários de pilha, stack smashing protection no kernel.
- Sandbox, capabilities, seccomp.
- Auditoria de syscalls.

### §14.7 Depuração e logging

#### §14.7.1 Log do kernel

```c
#define KERNEL_LOG_ERROR 0
#define KERNEL_LOG_WARN  1
#define KERNEL_LOG_INFO  2
#define KERNEL_LOG_DEBUG 3
#define KERNEL_LOG_TRACE 4

void print_kernel(int level, const char *format, ...);
```

- Saída em serial e, se disponível, framebuffer.
- Buffer circular de 4 KiB acessível via `/dev/kernel_message` ou syscall `kernel_log`.

#### §14.7.2 Core dumps

Ao receber SIGSEGV ou SIGABRT, o kernel **DEVE**:

1. Escrever `core.<pid>` no diretório atual (se permissão).
2. Formato ELF core dump com registradores e mapa de memória.

#### §14.7.3 GDB stub

- Implementar protocolo GDB remoto na serial (COM1).
- Ativar via parâmetro de boot `debug`.
- Suportar breakpoints, step, inspeção de memória e registradores.

---

## §15 IPC

### §15.1 Pipes

- Buffer circular de 64 KiB.
- `read` bloqueia se vazio e há escritores; retorna 0 se todos fecharam.
- `write` bloqueia se cheio; SIGPIPE se todos leitores fecharam.

### §15.2 Sinais

Ver §7.6.

### §15.3 FDs e `dup`

- Tabela de FDs por processo, 64 entradas.
- Compartilhamento via `dup`; `reference_count` no `file_t`.

---

## §16 Utilitários de userspace

| Binário | Descrição |
|---------|-----------|
| `init` | PID 1, inicia login |
| `login` | Autenticação |
| `msh` | Shell |
| `ls` | Lista diretório (flags `-l`, `-a`) |
| `cat` | Concatena arquivos |
| `echo` | Imprime argumentos |
| `mkdir` | Cria diretório |
| `rmdir` | Remove diretório vazio |
| `rm` | Remove arquivo (`-r` para recursivo) |
| `cp` | Copia arquivo |
| `mv` | Move/renomeia |
| `pwd` | Imprime diretório atual |
| `cd` (builtin) | Muda diretório |
| `ps` | Lista processos |
| `kill` | Envia sinal |
| `whoami` | Mostra UID efetivo |
| `id` | Mostra UID/GID |
| `chmod` | Altera permissões |
| `chown` | Altera dono |
| `sleep` | Pausa em segundos |
| `date` | Data/hora |
| `sync` | Sincroniza cache de disco |
| `shutdown` | Sync + poweroff |
| `clear` | Limpa tela |
| `mount` (futuro) | Monta FS |

Biblioteca `libum`: `print_formatted`, `allocate_memory`/`free_memory` (via `brk`/`mmap`), `string.h` (implementado com nomes descritivos), `unistd.h` (wrappers de syscall), `stdio.h` mínima.

---

## §17 Compilação, boot e testes

### §17.1 Toolchain

- `x86_64-elf-gcc` ou `clang` com `-target x86_64-unknown-none`.
- `nasm` para assembly.
- `ld.lld` ou `ld` com linker script `kernel.ld`.
- `grub-mkrescue` para ISO.
- `qemu-system-x86_64` para teste.

### §17.2 Flags obrigatórias

```
-ffreestanding -fno-stack-protector -fno-pic -mno-red-zone
-mcmodel=kernel -nostdlib -nostdinc -Wall -Wextra -Werror
```

### §17.3 Makefile (alvos)

| Alvo | Ação |
|------|------|
| `all` | Compila kernel, userspace e ISO |
| `kernel` | Compila apenas o kernel |
| `userspace` | Compila `/bin`, `/sbin` (estáticos) |
| `iso` | Gera `ummenos.iso` com GRUB |
| `run` | `qemu-system-x86_64 -cdrom ummenos.iso -serial stdio` |
| `debug` | QEMU com `-s -S` para GDB |
| `test` | Executa suíte de testes de integração |
| `clean` | Remove artefatos |

### §17.4 Testes de integração

Scripts (shell host) enviam comandos via serial e verificam saída esperada. Exemplos:

- `test_boot`: prompt de login em < 3 s.
- `test_login`: credenciais corretas e incorretas.
- `test_filesystem`: criar, escrever, ler, remover arquivo; reiniciar e verificar persistência.
- `test_process`: `fork` + `exec` + `wait`.
- `test_pipe`: `echo hello | cat` → `hello`.
- `test_signal`: `Ctrl+C` interrompe `sleep 100`.
- `test_protection`: processo tentando escrever em `0xFFFFFFFF80000000` recebe SIGSEGV.

### §17.5 Debug

- Log em serial com níveis: `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`.
- Símbolos preservados no ELF (não strip em debug).
- GDB stub via QEMU.

### §17.6 Framework de testes

#### §17.6.1 Testes de integração

- Scripts em Python com `pyserial` e `subprocess`.
- QEMU iniciado com `-serial stdio` ou `-serial file:serial.log`.
- Script envia comandos e verifica respostas na serial.

Exemplo:

```python
import serial, time, subprocess
qemu = subprocess.Popen(
    ["qemu-system-x86_64", "-cdrom", "ummenos.iso", "-serial", "stdio"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)
qemu.stdin.write(b"root\n")
qemu.stdin.write(b"ls\n")
time.sleep(1)
output = qemu.stdout.read(4096)
assert b"bin" in output
```

#### §17.6.2 Testes unitários

- Para PMM, VMM, heap, UmFS: testes em C compilados para o host (não bare metal).
- Usar `assert()` e um runner simples.
- Exemplo: `test_physical_memory.c` que aloca e libera frames, verificando o bitmap.

#### §17.6.3 Cobertura

- Meta: 70% das funções críticas cobertas.
- Usar `gcov` (mas cuidado: em bare metal é complicado; testes no host ajudam).

---

## §18 Documentação e governança

### §18.1 Arquivos de governança

- `LICENSE` — texto MIT.
- `README.md` — visão geral, como compilar, como contribuir.
- `ARCHITECTURE.md` — diagramas e decisões de projeto.
- `CONTRIBUTING.md` — processo de PR, estilo de código.
- `STYLE.md` — guia de estilo resumido, apontando para §4 desta especificação.
- `SPECIFICATION.md` — este documento.

### §18.2 Documentação externa

#### §18.2.1 Documentação do usuário

- Páginas `man` para comandos do shell.
- Guia de instalação em QEMU e hardware real.
- Guia do usuário do `msh`.

#### §18.2.2 Documentação do desenvolvedor

- `README.md`: visão geral, como compilar, como contribuir.
- `ARCHITECTURE.md`: diagramas e decisões de projeto.
- `CONTRIBUTING.md`: estilo de código (§4), processo de PR.
- `STYLE.md`: guia de estilo resumido.

#### §18.2.3 Documentação da API

- Header `ummenos/syscall.h` comentado com Doxygen.
- Lista de syscalls com assinaturas e códigos de erro.

### §18.3 Gerenciamento de pacotes (v2, esboço)

- Formato: `.umpkg` (tar + metadados JSON).
- Repositório: HTTP simples.
- Comandos: `package_install`, `package_remove`, `package_update`.
- Dependências resolvidas topologicamente.

---

## §19 Roadmap de implementação

1. **Fase 1 — Boot**
   - Multiboot2 + GDT + IDT + serial + framebuffer + panic.
2. **Fase 2 — Memória**
   - PMM bitmap + VMM 4 níveis + heap slab.
3. **Fase 3 — Interrupções e timer**
   - PIT + teclado + tratamento de exceções.
4. **Fase 4 — Processos**
   - PCB, escalonador, `fork`/`exec`/`exit`/`wait`, troca de contexto.
5. **Fase 5 — Syscalls e userspace**
   - Gate `syscall`, libc mínima, hello world em ring 3.
6. **Fase 6 — Bloco e UmFS**
   - ATA PIO + cache + superbloco + inodes + VFS.
7. **Fase 7 — TTY e shell**
   - Disciplina de linha, ANSI, `msh` com pipes e redirecionamento.
8. **Fase 8 — Usuários e segurança**
   - `/etc/passwd`, `/etc/shadow`, `login`, permissões.
9. **Fase 9 — Utilitários e polimento**
   - coreutils, testes, documentação.

Cada fase **DEVE** terminar com os testes de integração correspondentes passando antes de avançar.

---

## §20 Limites e constantes

| Constante | Valor |
|-----------|-------|
| `PAGE_SIZE` | 4096 |
| `HZ` | 100 |
| `MAX_PROCESSES` | 256 |
| `FILE_DESCRIPTOR_MAX` | 64 |
| `MAX_PATH` | 4096 |
| `NAME_MAX` | 122 |
| `MAX_ARGUMENTS` | 64 |
| `MAX_OPEN_FILES` | 1024 (global) |
| `PIPE_BUFFER_SIZE` | 65536 |
| `STACK_SIZE` | 8 MiB |
| `KERNEL_HEAP_SIZE` | 16 MiB |
| `BLOCK_SIZE` | 4096 |

---

## §21 Glossário

- **UmMenOS:** nome do projeto. Lê-se como `1 − N`, onde `N` é a versão.
- **UmFS:** sistema de arquivos nativo do UmMenOS.
- **PMM/VMM:** gerenciadores de memória física/virtual.
- **PCB:** Process Control Block.
- **VFS:** Virtual File System.
- **COW:** Copy-on-Write.
- **TTY:** terminal.
- **ABI:** Application Binary Interface (contrato syscall).
- **jiffies:** contador de ticks do timer.
- **BSP:** Bootstrap Processor (CPU 0).
- **AP:** Application Processor (demais CPUs).

---

## §22 Apêndices

### §22.1 Linker script (esqueleto)

```ld
ENTRY(_start)
SECTIONS {
    . = 0x100000;
    .boot : { *(.multiboot2) }
    .text : { *(.text .text.*) }
    .rodata : { *(.rodata .rodata.*) }
    .data : { *(.data .data.*) }
    .bss : { *(COMMON) *(.bss .bss.*) }
    . = ALIGN(4096);
    __kernel_end = .;
}
```

### §22.2 Estrutura de repositório sugerida

```
ummenos/
├── LICENSE
├── README.md
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── STYLE.md
├── SPECIFICATION.md
├── Makefile
├── kernel.ld
├── .clang-format
├── .clang-tidy
├── src/
│   ├── boot/         (multiboot2, gdt, idt, start.asm)
│   ├── memory/       (physical_memory, virtual_memory, heap)
│   ├── process/      (scheduler, fork, execute, signal)
│   ├── syscall/      (dispatch, copy_user)
│   ├── filesystem/
│   │   ├── vfs/
│   │   └── umfs/
│   ├── drivers/      (ata, keyboard, serial, framebuffer, pit)
│   ├── terminal/
│   ├── library/      (kernel_string, kernel_print, list)
│   └── kernel_main.c
├── user/
│   ├── library/      (libum)
│   ├── init/
│   ├── login/
│   ├── shell/
│   └── coreutils/
├── include/
│   ├── ummenos/      (headers públicos, ABI)
│   └── kernel/       (headers internos)
├── tools/
│   └── make_umfs_image
└── tests/
    ├── integration/
    └── unit/
```

---

**Fim da especificação.**
UmMenOS 1 — versão 2.1 do documento. Estável para implementação.
Alterações exigem bump de versão e nota de mudança.