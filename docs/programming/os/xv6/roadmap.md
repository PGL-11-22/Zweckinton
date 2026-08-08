# План работы

Цель — не просто разобрать четыре лабораторные, а построить цельный конспект по операционным системам. xv6 будет главным практическим примером, но мы также разберём общую терминологию, классические алгоритмы и то, чем учебная ОС отличается от Linux и других промышленных систем.

## Как будет устроена каждая глава

Для каждой подсистемы мы будем собирать не набор разрозненных фактов, а одинаковую структуру:

1. Какую проблему решает подсистема.
2. Термины и точные определения.
3. Аппаратные механизмы, на которые опирается ядро.
4. Абстракции и инварианты ядра.
5. Структуры данных и ключевые функции xv6.
6. Полный control flow одной операции по исходному коду.
7. Схемы памяти, состояний или взаимодействий.
8. Гонки, граничные случаи и типичные ошибки.
9. Небольшой эксперимент в xv6 и способ его проверки.
10. Сравнение с Linux и другими подходами.
11. Вопросы для самопроверки.

## Сквозная терминология

Параллельно с главами будет расти отдельный глоссарий. В нём мы разделим:

- аппаратные термины: hart, core, ISA, CSR, MMU, TLB, PLIC, CLINT, MMIO, DMA;
- термины процессов: program, process, thread, context, state, scheduling, preemption;
- термины памяти: page, frame, virtual address, physical address, address space, PTE, page fault;
- термины параллелизма: race condition, critical section, atomicity, visibility, ordering, deadlock;
- термины ввода-вывода и хранения: device, driver, interrupt, polling, block, sector, buffer cache, inode;
- термины интерфейсов: ABI, API, syscall, calling convention, file descriptor, socket;
- пары терминов, которые часто путают: concurrency/parallelism, trap/interrupt, page/frame, process/thread, mode/privilege level.

---

# Часть I. Git-интенсив и подготовка среды

Git активно изучаем в начале, пока исходники xv6 ещё не меняются или меняются минимально. После этой части отдельных Git-заданий не будет: мы просто будем применять уже освоенный workflow.

## G0. Модель Git и первый коммит

- working tree, index/staging area, repository;
- blob, tree, commit, parent, branch, `HEAD`;
- `status`, `diff`, `diff --cached`, `add`, `commit`, `show`, `log`;
- явное добавление файлов вместо привычки `git add .`;
- первый атомарный коммит каркаса MkDocs.

## G1. История, ссылки и удалённые репозитории

- граф коммитов и достижимость;
- branch как подвижная ссылка, tag как метка;
- detached `HEAD`;
- `origin` и `upstream`, remote-tracking branches, fetch/pull/push;
- shallow clone и догрузка истории;
- тег каноничной базы xv6 и собственная учебная ветка.

## G2. Ветки и слияния

- `switch`, `branch`, fast-forward, three-way merge, merge base;
- merge commit и линейная история;
- конфликт как невозможность автоматически совместить два состояния;
- создание и разрешение контролируемого конфликта;
- проверка diff до слияния.

## G3. Исправление ошибок и восстановление

- разница между `restore`, `reset` и `revert`;
- `commit --amend` для ещё не опубликованного коммита;
- `reflog` как карта перемещений ссылок;
- безопасное восстановление потерянного коммита;
- когда нельзя переписывать опубликованную историю.

## G4. Перенос и упорядочивание изменений

- `cherry-pick` одного смыслового коммита;
- rebase как перепривязка цепочки коммитов;
- interactive rebase: `reword`, `edit`, `squash`, `fixup`, `drop`;
- отделение смысового diff от шума форматирования;
- подготовка чистой цепочки коммитов для первого учебного изменения xv6.

## G5. Ежедневные и диагностические инструменты

- `stash` для краткоживущих изменений;
- `worktree` для параллельной работы с двумя ветками;
- `bisect` и `bisect run` для поиска регрессии;
- теги контрольных точек;
- простой workflow code review: ветка → diff → тесты → review → merge.

---

# Часть II. Общая модель операционной системы

## 1. Зачем нужна ОС

- ОС как менеджер ресурсов и как поставщик абстракций;
- мультиплексирование CPU, памяти и устройств;
- изоляция, защита, совместное использование и постоянство данных;
- kernel space и user space;
- монолитное ядро, микроядро, hybrid kernel, exokernel;
- место xv6 среди Unix-подобных систем и её учебные ограничения.

## 2. Аппаратура глазами ядра

- CPU, core, hart, register, program counter, stack pointer;
- память, шина, кэш, контроллер устройства;
- ISA, ABI, calling convention и binary interface;
- RISC-V privilege levels M/S/U;
- CSR, exception, interrupt, timer;
- MMU, physical/virtual address, MMIO, DMA;
- что QEMU эмулирует для xv6.

## 3. Загрузка xv6

- firmware, bootloader и kernel image как общие понятия;
- адресное пространство QEMU `virt`;
- linker script, sections `.text/.rodata/.data/.bss` и символы линкера;
- `entry.S` → `start.c` → `main.c`;
- инициализация стека, режима S, таймера и вторичных hart;
- порядок инициализации подсистем в `main`;
- запуск первого user process.

## 4. Граница user/kernel

- привилегии и защищённые операции;
- system call, exception, trap, interrupt и fault;
- `ecall`, syscall number, аргументы и return value;
- trampoline, trapframe, `stvec`, `sepc`, `scause`, `sstatus`;
- `uservec` → `usertrap` → `syscall` → `usertrapret` → `userret`;
- kernel trap и вложенные события;
- безопасное копирование данных между user/kernel;
- разбор одного syscall от user-обёртки до ядра и обратно.

---

# Часть III. Вычисление, параллелизм и память

## 5. Программа, процесс и thread

- разница между program, process и thread;
- PID, process state, parent/child, zombie, orphan;
- состав `struct proc`: kernel stack, trapframe, context, pagetable, open files, cwd;
- жизненный цикл `allocproc/fork/exec/exit/wait/freeproc`;
- что именно копирует `fork`;
- что заменяет `exec`;
- отличие модели xv6 от многопоточного процесса Linux.

## 6. Context switch и планирование

- context, context switch и mode switch;
- scheduler thread каждого CPU;
- `scheduler`, `sched`, `yield`, `swtch`;
- cooperative и preemptive scheduling;
- timer interrupt и time slice;
- round-robin xv6;
- общая теория: FIFO, SJF, STCF, Round Robin, priority scheduling, MLFQ;
- latency, response time, turnaround time, throughput, fairness, starvation;
- многоядерное планирование и CPU affinity как темы за пределами xv6.

## 7. Параллелизм и синхронизация

- concurrency и parallelism;
- interleaving, race condition, data race, critical section;
- atomicity, visibility, ordering и memory model;
- interrupt masking и atomic instructions RISC-V;
- spinlock xv6, `acquire/release`, ownership и отключение прерываний;
- sleeplock, mutex, semaphore, condition variable, monitor;
- `sleep/wakeup` и lost wakeup;
- deadlock, livelock, starvation, priority inversion;
- условия Coffman и порядок захвата блокировок;
- поиск инвариантов в коде xv6.

## 8. Физическая память

- RAM как массив физических байтов;
- page frame, page size, alignment и fragmentation;
- карта физической памяти xv6;
- `kinit`, `freerange`, `kalloc`, `kfree` и free list;
- отравление выделенной/освобождённой памяти;
- first fit, best fit, segregated lists, slab и buddy allocator;
- internal/external fragmentation;
- buddy allocator и XOR-оптимизация из задания `alloc`;
- взаимодействие аллокатора с остальным ядром.

## 9. Sv39 и виртуальная память

- зачем нужны virtual memory и address space;
- virtual page и physical frame;
- разбиение VA на VPN[2], VPN[1], VPN[0] и offset;
- три уровня page table;
- PTE: V/R/W/X/U/G/A/D и software-defined bits;
- `satp`, TLB и `sfence.vma`;
- `walk`, `mappages`, `walkaddr`, `uvmunmap`, `freewalk`;
- kernel direct mapping, trampoline и kernel stacks;
- user address space: text, data, heap, guard page, stack, trapframe;
- page fault, protection fault и lazy allocation;
- huge pages, ASID, swapping и page replacement как общая теория, которая почти не реализована в xv6.

## 10. Жизненный цикл address space

- загрузка ELF в `exec`;
- ELF header, program header, segment и section;
- `sbrk`, рост heap и `uvmalloc/uvmdealloc`;
- копирование address space при `fork`;
- copy-on-write: read-only PTE, reference count, write fault, private copy;
- взаимодействие COW с `copyout`, `exec`, `exit` и ошибками выделения;
- гонки reference counting;
- demand paging, memory-mapped files, shared memory и overcommit как расширения модели.

---

# Часть IV. Ввод-вывод, хранение и IPC

## 11. Ввод-вывод и драйверы

- device, controller, bus, driver;
- programmed I/O, polling, interrupt-driven I/O и DMA;
- port I/O и MMIO;
- interrupt controller PLIC;
- UART и console: путь символа на вводе и выводе;
- virtio disk и descriptor rings;
- blocking/non-blocking, synchronous/asynchronous I/O;
- buffering, caching, batching и backpressure;
- жизненный цикл I/O request в xv6.

## 12. Файлы и файловые дескрипторы

- файл как абстракция последовательности байтов;
- file descriptor, open file description и inode;
- per-process descriptor table и global file table;
- `open/read/write/close/dup/fstat`;
- offset, access mode и reference count;
- device files и единый Unix-интерфейс;
- pipe как файл и как IPC;
- blocking semantics, EOF и broken pipe;
- полная трасса `open → read/write → close`.

## 13. Файловая система xv6

- disk layout: boot block, superblock, log, inode blocks, bitmap, data blocks;
- block, sector и filesystem block;
- on-disk inode и in-memory inode;
- direct и indirect block addresses;
- pathname resolution, directory entry, root, cwd, `.` и `..`;
- hard link, link count, unlink и момент удаления inode;
- buffer cache и sleeplock;
- слои syscall → file → inode → log → buffer cache → driver;
- ограничения формата xv6.

## 14. Сбои и журналирование

- crash consistency и частично выполненная операция;
- transaction, commit point, write-ahead log;
- `begin_op/end_op`, absorption и group commit;
- recovery после сбоя;
- инварианты журнала xv6;
- разница между atomicity, durability и persistence;
- journaling, copy-on-write filesystems и checksums как промышленные подходы.

## 15. IPC и взаимодействие процессов

- shared memory и message passing;
- pipes xv6;
- producer/consumer и bounded buffer;
- Unix signals, message queues, shared memory, futex и sockets как общая теория;
- local IPC и network IPC;
- serialization, copying и zero-copy;
- влияние IPC-модели на архитектуру микроядра.

---

# Часть V. Сеть, защита и темы за пределами xv6

## 16. Сетевое устройство и стек протоколов

- PCI, BAR, MMIO-регистры и E1000;
- RX/TX descriptor rings и DMA;
- packet, frame, datagram, segment;
- MAC-адрес, Ethernet frame и EtherType;
- ARP cache и resolution;
- IPv4 header, addressing, checksum и fragmentation;
- ICMP echo;
- UDP, port, socket и demultiplexing;
- byte order и parsing untrusted packet data;
- путь пакета от NIC до user process и обратно;
- чего нет в учебном стеке: TCP, routing, firewall, congestion control.

## 17. Защита и безопасность

- protection и security;
- privilege, isolation, least privilege, trusted computing base;
- user/kernel boundary и page permissions;
- validation of pointers, lengths, indexes и packet fields;
- identity, authentication, authorization и access control;
- discretionary/mandatory access control как общая теория;
- capabilities и file descriptors как ограниченные дескрипторы ресурсов;
- attack surface xv6 и почему xv6 не является production-системой.

## 18. Виртуализация и контейнеры

- virtual machine и process virtual machine;
- trap-and-emulate и hardware virtualization;
- hypervisor type 1/type 2;
- guest/host, virtual CPU и virtual device;
- container, namespace, cgroup и image;
- разница изоляции VM, container и process;
- как механизмы, изученные на xv6, лежат в основе этих систем.

## 19. Производительность и наблюдаемость

- latency, throughput, utilization, scalability;
- CPU-bound, memory-bound и I/O-bound workload;
- cache locality, contention, false sharing и batching;
- tracing, logging, counters, sampling и profiling;
- benchmark, workload, warm-up и reproducibility;
- как измерять изменения xv6 и не делать ложных выводов;
- GDB, QEMU monitor, disassembly, symbol table и печать состояния ядра.

## 20. Сравнение xv6 с production-системой

- какие идеи xv6 прямо переносятся на Unix/Linux;
- чего в xv6 нет: signals, threads, demand paging, swap, mmap, permissions, VFS, TCP, SMP scalability;
- как усложняются scheduler, memory manager, VFS и driver model в Linux;
- где проходит граница между учебной моделью и промышленной реализацией;
- какие темы изучать дальше после xv6.

---

# Часть VI. Лабораторные как case studies

Лабораторные не заменяют основные главы. Мы возвращаемся к ним после разбора соответствующей теории:

1. `intro` — процессы, syscall и путь user ↔ kernel.
2. `alloc` — динамическое выделение объектов ядра, buddy allocator и его метаданные.
3. `cow` — Sv39, page fault, reference counting и copy-on-write `fork`.
4. `network` — PCI/E1000, DMA и учебный network stack.
5. `pe` — отдельная case study по форматам executable files; сравнение PE и ELF.
6. `networkfs` — отдельная case study по FUSE, VFS, RPC и сетевой файловой системе.

Для каждой case study нужно показать:

- состояние starter-версии;
- поставленную проблему;
- смысловой diff без шума форматирования;
- новые инварианты и точки синхронизации;
- ошибочные подходы и пройденные тесты;
- совместимость с каноничной веткой xv6.

## Порядок фактической работы

1. Пройти Git-интенсив G0–G3 на MkDocs и безопасных учебных ветках.
2. Зафиксировать каноничную базу xv6 и настроить upstream/origin.
3. Написать вводные главы 1–4 и сквозной глоссарий.
4. Довести Git-интенсив до G5 на первой малой модификации xv6.
5. После этого перейти к систематическому разбору ОС без искусственных Git-заданий на каждую главу.
6. Разбирать лабораторные только после теории соответствующей подсистемы.
7. После каждой части делать обзорную схему и прогон самопроверки.
8. В финале собрать цельную карту xv6 и таблицу «абстракция → структура данных → функции → аппаратура».

## Критерии готового конспекта

- [ ] Все термины определены до первого сложного использования и связаны с глоссарием.
- [ ] Каждая подсистема объяснена на трёх уровнях: общая идея, аппаратный механизм, код xv6.
- [ ] Для каждого важного пути есть control-flow или state diagram.
- [ ] Указаны инварианты, блокировки, граничные случаи последствия ошибок.
- [ ] Все ключевые пути проверены по актуальному исходному коду xv6.
- [ ] Ясно отделено то, что реализовано в xv6, от общей теории и поведения Linux.
- [ ] Каждая лабораторная привязана к теории, а не заменяет её.
- [ ] Конспект читается без необходимости сначала изучать README заданий.
