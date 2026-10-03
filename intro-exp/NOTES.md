# Заметки по лабораторной (передача контекста между сессиями)

> Статус: **выполнены подготовка стенда (этап 0) и шаг 4.1 (паспорт системы).**
> Шаг 4.2 (подготовка окружения) ещё **не выполнен**: его делаем в новом диалоге. Ниже его план и черновые решения.

## Задание
- Лабораторная «Introductory Experiment» (курс ОС). Описание: `README.md`, правила отчёта: `report.md`, методички: `experiments/*.md`.
- **Вариант: Read/Write, Cache, 8M, Rand.** Вопрос: насколько запись тяжелее чтения при обходе случайного графа 8 МБ с включённым кэшем?
- Граф один: `graph-rand.bin` (8M, `--seed 427 --topology chain -b 0.5 --min-step-pages 2`). `--no-cache` не используется, `graph-seq.bin` не нужен.
- Серий четыре: `graph_traverse` (read/lseek) × {read, `--write`} и `graph_traverse_mmap` × {read, `--write`}.
- Структура работы: Этап 1 (`graph_traverse`, шаги 4.1–4.9), Этап 2 (`graph_traverse_mmap`, повтор 4.1–4.9), Этап 3 (общее сравнение), отчёт и защита (раздел 7–8 README).

## Стенд
- Хост: MacBook Pro (MacBookPro17,1), Apple M1 (4 P + 4 E ядра), 16 ГБ RAM, SSD APPLE AP1024Q 1 ТБ, APFS (`results/passport_host.txt`).
- Гипервизор: VirtualBox 7.2.20 (r175154). VM `debian13`: Debian 13 (trixie) arm64, ядро 6.12.111, 4 vCPU, 4 ГБ RAM, диск `debian13.vdi` 31,29 ГБ (`results/vm_config.txt`, скриншот `results/vm_details.png`).
- Вход в VM с Mac: `ssh -p 2222 istnoteast@127.0.0.1`. Проброс порта делался командой `VBoxManage controlvm debian13 natpf1 "ssh,tcp,,2222,,22"` (выполняется на Mac, не в VM; после выключения VM может понадобиться заново или через Настройки → Сеть → Проброс портов).
- Пользователь VM: `istnoteast`. `sudo` работает с паролем **этого** пользователя (не root). Пароль здесь не хранится.
- Проект в VM: `~/intro-exp`. Программы собраны в `out/` (`graph_traverse`, `graph_traverse_mmap`), граф `graph-rand.bin` лежит рядом.
- Правило работы: команды лабораторной выполняются только при приглашении `istnoteast@istnoteast:~/intro-exp$` (в VM). На Mac приглашение `ilyastolbcenko@MacBook-Pro-22 ~ %`.
- В консоли окна VirtualBox вставка из буфера не работает, поэтому работаем через SSH-терминал на Mac.
- Установлено в VM: build-essential, clang 19.1.7, python3 3.13.5, strace 6.13, sysstat, stress-ng 0.19.02, linux-perf 6.12.111, linux-cpupower, util-linux, smartmontools, numactl, lshw, git.
  - **Не установлен** пакет `time` (нужен для `/usr/bin/time -v`): `sudo apt install -y time`.
  - `sysctl` не в `PATH` обычного пользователя (использовать `/proc/sys/...` или `/usr/sbin/sysctl`).

## Что сделано

### Этап 0: стенд
- VirtualBox + Debian 13 arm64 установлены (текстовая установка, без рабочего стола, SSH-сервер и стандартные утилиты).
- Проект скопирован в VM по `scp`. `graphgen.py` лежит в `src/` (в README он указан как `lab/util/graphgen.py`, в нашей копии путь `src/graphgen.py`).
- Сборка: `clang -o out/graph_traverse src/graph_traverse.c`, `clang -o out/graph_traverse_mmap src/graph_traverse_mmap.c`.
- Граф сгенерирован командой: `python3 src/graphgen.py -s 8M --seed 427 --topology chain -b 0.5 --min-step-pages 2 --verify -o graph-rand.bin`.
  - Результат: 8 388 592 байта, 349 523 вершины по 24 байта, `fan_out = 1`, root_index 277 622 (offset 6 662 968), доля переходов вперёд/назад 0,499/0,501, доля коротких переходов 0,008.
  - `--verify`: достижимо 100 % вершин, максимальная глубина BFS 349 522, граф ацикличен.

### Шаг 4.1: паспорт системы (закрыт)
Файлы в `results/`: `passport_vm.txt` (138 строк), `passport_host.txt`, `vm_config.txt`, `vm_details.png`, а также `env_before.txt` (снимок VM, см. ниже).

**VM (гость), источник: `passport_vm.txt`**
| Параметр | Значение |
|---|---|
| Ядро / архитектура | Linux 6.12.111+deb13-arm64, aarch64 |
| CPU | 4 vCPU, 1 поток на ядро, 1 кластер, 1 NUMA-узел; `Vendor ID: Apple`, модель `-` (гость не знает) |
| RAM / swap | 3,8 ГиБ (≈3911 МБ) / 1,6 ГиБ; на момент снятия `buff/cache` 1,6 ГиБ, swap не используется |
| Диск | `VBOX HARDDISK` 31,3 ГБ; `/` на ext4 (`/dev/sda3`, 28,7 ГБ, опции `rw,relatime,errors=remount-ro`), ESP 977 МБ, swap 1,6 ГБ |
| Кэши CPU | sysfs показывает уровни L1d, L1i, L2, **размеров нет** |
| Параметры записи страниц | `dirty_ratio=20`, `dirty_background_ratio=10`, `dirty_expire_centisecs=3000` (30 с), `dirty_writeback_centisecs=500` (5 с) |

**Хост, источник: `passport_host.txt` (`system_profiler`, `sysctl`)**
| Параметр | Значение |
|---|---|
| Модель | MacBook Pro (MacBookPro17,1), чип Apple M1 |
| Ядра | 8: 4 производительных (P) + 4 энергоэффективных (E) |
| RAM | 16 ГБ |
| Накопитель | Apple SSD AP1024Q, 1 ТБ, APFS, SMART Verified |
| Кэш P-ядра | L1d 128 КБ, L2 12 МБ |
| VirtualBox | 7.2.20 r175154 |

**Недоступно в гостевой ОС (и почему), для раздела «ограничения» отчёта**
- `smartctl`: SMART недоступен, диск виртуальный (`device lacks SMART capability`).
- `cpupower`, governor, Turbo: драйвера cpufreq в госте нет, частотой управляет macOS. Зафиксировать частоту нельзя.
- Размеры кэшей CPU и точная модель процессора: в госте не видны; взяты с хоста (оговорка: на каком физическом ядре P или E работает vCPU, гость не знает, `taskset` закрепляет процесс только на виртуальном ядре).
- `systemd-detect-virt` вернул `none` (VirtualBox на Apple Silicon использует гипервизор macOS и не распознаётся). Виртуальность подтверждается настройками VirtualBox (`vm_details.png`, `vm_config.txt`), а не этим выводом.
- `ROTA=1` у виртуального диска ложный признак: физический накопитель хоста SSD (по `system_profiler`).
- `lshw` показывает только обобщённые устройства (`4GiB System memory`, `33GB HARDDISK`).

**Выводы из паспорта, важные для эксперимента**
- Граф 8 МБ ≪ RAM гостя, целиком помещается в page cache; также меньше L2 P-кластера хоста (12 МБ), но привязку vCPU к P/E-ядрам утверждать нельзя.
- Диск гостя лежит на SSD хоста (двойное кэширование: page cache гостя и кэш macOS для `.vdi`). Гипотеза про «позиционирование головки» из README к нам не относится.
- Параметры `dirty_*`: страницы в режиме `--write` не будут сброшены на диск во время короткого замера (порог 10 % ≈ сотни МБ, а файл 8 МБ); их сбросит таймер примерно через 30 с, поэтому нужен `sync` между запусками.

**Снимок VM перед 4.2 (`results/env_before.txt`)**: `load average` 0,00 / 0,00 / 0,00, idle по ядрам ≈ 99–100 %, `steal` 0. Все прерывания устройств (virtio-диск, сеть, графика, USB/звук) обрабатывает **CPU0**; CPU2 и CPU3 полностью простаивают.

**Состояние хоста (для отчёта)**
- До разгрузки: `Load Avg` 2,48 / 2,55 / 2,58, `PhysMem` занято 15 из 16 ГБ, свободно 67 МБ, 4 ГБ в сжатии. Крупные потребители: WindowServer, v2RayTun, Telegram, Chrome (≈1,4 ГБ), VirtualBoxVM.
- После закрытия Chrome, Telegram (и сворачивания окна VM): `Load Avg` 2,21 / 2,76 / 2,76, CPU idle 79 %, свободно 3428 МБ, `System-wide memory free percentage` 68 %, `swapouts` не росли.
- Перед финальной серией (шаг 4.6) снимок хоста и VM повторить.

## Что показал код (`src/graph_traverse.c`, `src/graph_io.h`)
- Read: на каждую вершину `lseek` + `read` (24 Б), ≈ 700 тыс. вызовов за один обход (349 523 вершины).
- Write: добавляется `lseek` + `write` (8 Б, поле `value`), ≈ 1,4 млн вызовов. `fsync`/`msync` нет, `close` страницы не сбрасывает, диск в замере не участвует (только копирование и пометка страниц «грязными»).
- Каждая итерация заново открывает файл и проходит всю цепочку; прогресс печатается в `stderr` (в серии перенаправлять в `/dev/null`).
- Аргументы: `[--write] [--no-cache] <iterations> <file1> [file2 ...]`. Для нашего варианта `--no-cache` не используем.
- `graph_traverse_mmap.c` ещё не читали (нужно перед Этапом 2).

## Шаг 4.2: подготовить окружение (НЕ выполнен, делаем в новом диалоге)
Источники: `README.md` (шаг 4.2), `experiments/environment.md`. Черновые решения, которые уже согласованы:
1. **Фон на Mac:** закрыть браузер, мессенджеры, VPN/прокси (v2RayTun), облачные синхронизации; подключить зарядку; окно VM свернуть (не закрывать), работать через SSH. Критерий: `top -l 1 | head -12` и `memory_pressure | tail -5` показывают запас памяти и низкую загрузку. Результат сохранить: `{ top -l 1 | head -12; memory_pressure | tail -5; } > results/host_load_before.txt`.
2. **Не давать Mac уснуть:** в отдельном окне терминала Mac `caffeinate -dims` (остановка Ctrl+C после серий).
3. **Фон в VM:** `uptime`, `top -bn1 | head -20` (уже снято в `env_before.txt`: простой).
4. **Ядро для `taskset`:** CPU 2 (прерывания на CPU0, CPU2 свободен). Проверка: `taskset -c 2 true && echo ok`, `taskset -c 2 bash -c 'taskset -cp $$'` (ожидаем `current affinity list: 2`).
5. **Governor и Turbo:** не контролируются (ограничение VM, записать в отчёт). **`nice`:** не используем (машина простаивает).
6. **Кэш:** тёплый (один прогревочный запуск, результат отбрасывается; `drop_caches` не используем), одинаково для read и write. **`sync`** перед каждым запуском, вне замеряемого времени.
7. **Размер данных:** 8 МБ ≪ RAM 3,8 ГиБ, обосновать для защиты.
8. Записать решения в `results/env_decisions.md` и убедиться, что чек-лист из `environment.md` (раздел 9) закрыт.

## Дальше
1. Закончить 4.2 (см. выше).
2. Шаг 4.3 (baseline, в VM): прогрев и по одному запуску read/write (`sync; time taskset -c 2 ./out/graph_traverse 1 graph-rand.bin` и с `--write`); `strace -c -o results/strace_*.txt` (запуск вида `taskset -c 2 strace -c -o ... ./out/graph_traverse ...`); `/usr/bin/time -v` (нужен пакет `time`); `perf stat -e task-clock,context-switches,cpu-migrations,page-faults` (аппаратные счётчики в VM, вероятно, недоступны); один запуск под `stress-ng --cpu 2 --io 1 --vm 1 --vm-bytes 256M --timeout 40s`. По времени одного обхода выбрать `<iterations>` (цель ≈ 1–3 с на запуск).
3. 4.4 гипотеза. Черновик: write примерно вдвое «тяжелее» read по числу системных вызовов, рост времени в основном за счёт `sys`; реальный сброс на диск в замере почти не виден (тёплый кэш и отложенная запись).
4. 4.5 план (метрика, N по пилотной серии N=10 и формуле из `statistics.md`, число отбрасываемых прогревочных запусков, **чередование** R/W/R/W…, раздел «измерение отдельно от мониторинга»). 4.6 снимок «до». 4.7 скрипт сбора (CSV сырых данных, метаданные: граф, режим, итерации, taskset). 4.8 анализ (среднее, σ, ДИ по t-распределению, график с «усами», наложенные плотности). 4.9 вывод (сравнение, погрешность, достаточность N).
5. Этап 2 (`mmap`): повторить 4.1–4.9, сравнить context switches и page faults. Этап 3: общий график четырёх серий, вывод про интрузивность `perf stat`/`strace`.
6. Отчёт: PDF, лучше на Typst (ГОСТ-шаблон по ссылке в `report.md`); в основной текст только существенное, скрипты и CSV в приложение.

## Что не включать в отчёт
- Серийный номер, UUID и UDID хоста (в `passport_host.txt` они отфильтрованы).
- UUID и MAC из `results/vm_config.txt`; для приложения сделать копию: `grep -vEi "uuid|macaddress" results/vm_config.txt > results/vm_config_clean.txt`.
