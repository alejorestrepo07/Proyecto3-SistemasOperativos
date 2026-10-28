# Proyecto de Sistemas Operativos
## Proyecto 3: Tablas de páginas en acción (xv6)

### Integrantes

- Tomás Ramírez Galeano
- Alejandro Restrepo Osorio

### Curso

Sistemas Operativos

### Semestre

2026-2

---

# Descripción

Trabajamos sobre xv6-riscv en el commit `06aad25` y agregamos tres cosas.

Parte A: cada proceso tiene una página `USYSCALL`, justo debajo del trapframe, donde el kernel guarda su pid. El proceso la puede leer pero no escribir. Con eso `ugetpid()` saca el pid leyendo memoria, sin hacer una llamada al sistema.

Parte B: la llamada `vmprint()` imprime la tabla de páginas del proceso. Recorre los tres niveles de Sv39 y muestra solo las entradas válidas.

Parte C: la llamada `pgaccess()` dice qué páginas de un rango se leyeron o escribieron desde la última vez que se preguntó. Para eso revisa el bit `PTE_A`, que el hardware pone en 1 cuando se accede a una página, y luego lo vuelve a poner en 0.

---

# Archivos modificados

- `kernel/memlayout.h`: definimos `USYSCALL` (`TRAPFRAME - PGSIZE`) y `struct usyscall`. La estructura va dentro de `#ifndef __ASSEMBLER__` porque `trampoline.S` también incluye este archivo.
- `kernel/proc.h`: agregamos a `struct proc` el puntero `usyscall` a la página compartida.
- `kernel/proc.c`: `allocproc()` reserva la página y le escribe el pid. `proc_pagetable()` la mapea con `PTE_R | PTE_U`. `proc_freepagetable()` la desmapea y `freeproc()` la libera. `growproc()` ahora usa `USYSCALL` como límite del heap.
- `kernel/sysproc.c`: el límite de `sys_sbrk()` pasó a ser `USYSCALL`. Agregamos `sys_vmprint()` y `sys_pgaccess()`.
- `kernel/vm.c`: `vmprint()` y la función recursiva `vmprint_walk()`, hecha a partir de `freewalk()`.
- `kernel/riscv.h`: definimos `PTE_A` (bit 6).
- `kernel/defs.h`: prototipo de `vmprint()`.
- `kernel/syscall.h` y `kernel/syscall.c`: registramos `SYS_vmprint` (23) y `SYS_pgaccess` (24).
- `user/ulib.c`: `ugetpid()`, que lee el pid de `USYSCALL`.
- `user/user.h` y `user/usys.pl`: declaraciones y stubs de `ugetpid`, `vmprint` y `pgaccess`.

---

# Integración con xv6

1. Clonar xv6-riscv y hacer `git switch -c proyecto3 06aad25`.
2. Copiar los archivos entregados en sus rutas.
3. Agregar `$U/_pgtbltest` a UPROGS en el Makefile.
4. Compilar con `make qemu`.
5. Ejecutar `pgtbltest` y `usertests -q`.

---

# Respuestas a las preguntas

## 1. Interpretación de vmprint()

Esta es la salida de un programa que solo llama a `vmprint()` apenas arranca:

```text
page table 0x0000000087f22000
 ..0x0000000000000000: pte 0x0000000021fc7801 pa 0x0000000087f1e000
 .. ..0x0000000000000000: pte 0x0000000021fc7401 pa 0x0000000087f1d000
 .. .. ..0x0000000000000000: pte 0x0000000021fc7c5b pa 0x0000000087f1f000
 .. .. ..0x0000000000001000: pte 0x0000000021fc7017 pa 0x0000000087f1c000
 .. .. ..0x0000000000002000: pte 0x0000000021fc6c07 pa 0x0000000087f1b000
 .. .. ..0x0000000000003000: pte 0x0000000021fc68d7 pa 0x0000000087f1a000
 ..0x0000003fc0000000: pte 0x0000000021fc8401 pa 0x0000000087f21000
 .. ..0x0000003fffe00000: pte 0x0000000021fc8001 pa 0x0000000087f20000
 .. .. ..0x0000003fffffd000: pte 0x0000000021fd4813 pa 0x0000000087f52000
 .. .. ..0x0000003fffffe000: pte 0x0000000021fd00c7 pa 0x0000000087f40000
 .. .. ..0x0000003ffffff000: pte 0x000000002000184b pa 0x0000000080006000
```

Los permisos están en los últimos bits de la PTE: V = 0x01, R = 0x02, W = 0x04, X = 0x08, U = 0x10, A = 0x40, D = 0x80. Las líneas de los dos primeros niveles terminan en `01`, o sea que solo tienen V y apuntan a la siguiente tabla. Las páginas de último nivel son:

| VA | Bits | Qué es |
|---|---|---|
| `0x0` | V R X U A | Texto (el código). Se lee y se ejecuta, pero no se puede escribir. |
| `0x1000` | V R W U | Datos globales. Se leen y escriben, pero no se ejecutan. |
| `0x2000` | V R W | Página de guarda. No tiene U, así que el usuario no la puede usar. |
| `0x3000` | V R W U A D | Pila. A y D están en 1 porque el programa ya la usó. |
| `0x3fffffd000` | V R U | `USYSCALL`. Solo lectura; de aquí lee `ugetpid()`. |
| `0x3fffffe000` | V R W A D | Trapframe. Sin U: solo lo usa el kernel para guardar los registros del proceso. |
| `0x3ffffff000` | V R X A | Trampolín. Sin U: es código del kernel para entrar y salir de él. Su `pa` es la misma en todos los procesos. |

Como el programa no pidió memoria, no hay heap. Con `sbrk()` aparecerían páginas nuevas desde `0x4000` con V R W U. Las de `sbrklazy()` solo aparecen cuando el programa las toca.

## 2. Página de guarda

`exec()` deja una página debajo de la pila y le quita el bit U. Si el programa desborda la pila, intenta usar esa página y el hardware da un fallo de página porque el usuario no tiene permiso. El kernel ve que la dirección ya está mapeada (no es una página perezosa), imprime `usertrap(): unexpected scause ...` y mata al proceso.

Si la página no existiera, la pila seguiría bajando y dañaría los datos del programa sin que nadie se diera cuenta. Por eso es una PTE válida sin U y no un hueco vacío: un hueco por debajo de `sz` lo tomaría `vmfault()` como memoria perezosa y le asignaría una página.

## 3. Otras llamadas que se pueden acelerar

Se pueden acelerar las llamadas que solo leen un dato que el kernel puede copiar a la página y mantener al día. Por ejemplo:

- `sbrk(0)`, que solo devuelve el tamaño del proceso. El kernel lo actualizaría en la página cada vez que cambia.
- Un `getppid()` (xv6 no lo tiene), porque el padre casi nunca cambia.
- `uptime()`, aunque es más costoso: el contador cambia en cada tick y habría que actualizarlo en la página de todos los procesos, o usar una sola página compartida por todos.

No se pueden acelerar las que tienen que cambiar algo en el kernel o hacer entrada/salida: `fork`, `exec`, `exit`, `wait`, `kill`, `sbrk` con n distinto de 0, `pause`, y las de archivos (`read`, `write`, `open`, `close`, `pipe`, etc.). Esas necesitan que el kernel revise permisos, tome locks o duerma al proceso, y nada de eso se puede hacer leyendo una página.

## 4. Riesgo de USYSCALL con PTE_W

El proceso podría escribir cualquier pid en la página y `ugetpid()` devolvería un valor falso. Un programa que use ese pid para algo, por ejemplo para hacer `kill()`, actuaría sobre el proceso equivocado.

En nuestro código el kernel no lee esa página (usa `p->pid`), así que el kernel no se ve afectado. El problema sería mayor si más adelante el kernel confiara en lo que hay ahí o si se guardaran más datos en la página. En general, lo que el kernel comparte con el usuario debe ser de solo lectura.

## 5. pgaccess() y páginas perezosas

`sbrklazy()` solo aumenta el tamaño del proceso, no crea la PTE ni reserva memoria. Si no hay PTE, el hardware nunca pudo poner `PTE_A` en 1, porque el proceso no ha tocado esa página. Entonces 0 es la respuesta correcta. Cuando el proceso la toca, `vmfault()` le da memoria, se crea la PTE y desde ese momento `pgaccess()` sí la detecta.

No le debe asignar memoria porque eso daña la idea de la asignación perezosa: preguntar por 64 páginas gastaría 64 páginas físicas que el programa tal vez nunca use. Además, una consulta no debería cambiar el estado del proceso, y la página nueva igual saldría en 0. Por eso usamos `walk(pagetable, va, 0)`, que no crea nada, en vez de `walkaddr()` o `vmfault()`.

---

# Resultados de las pruebas

Corrimos `ejecutar_pruebas.py` sobre una copia limpia de xv6 (`06aad25`) con nuestros archivos. La salida completa está en `pruebas/`.

- `pgtbltest`: pasaron las 11 pruebas.
- Revisión del formato de `vmprint()`: OK.
- `usertests -q`: `ALL TESTS PASSED`, sin fugas de memoria.

En `usyscall_solo_lectura` sale `usertrap(): unexpected scause 0xf`. Es normal, porque la prueba escribe en `USYSCALL` a propósito para que el kernel mate el proceso.

El kernel compila sin warnings del compilador. Los avisos `LOAD segment with RWX permissions` son del linker y también salen en xv6 sin modificar.

---

# Decisiones de diseño

- La página `USYSCALL` es del proceso y no de la tabla de páginas. Se reserva en `allocproc()` y se libera en `freeproc()`. Cuando se hace `exec()`, la tabla vieja solo la desmapea y la nueva la vuelve a mapear, así que no hay que copiar nada.
- `proc_freepagetable()` desmapea `USYSCALL` antes de `uvmfree()`. Si no, `freewalk()` encuentra la página todavía mapeada y da `panic: freewalk: leaf`.
- `fork()` no se tocó: el hijo pasa por `allocproc()` y recibe su propia página con su pid.
- `vmprint()` sigue la forma de `freewalk()`. La dirección virtual de cada entrada se calcula con `base | (i << (12 + 9 * nivel))`.
- En `pgaccess()` el rango se valida con `base > p->sz || npages * PGSIZE > p->sz - base` para evitar overflow con una `base` muy grande.
- Después de limpiar `PTE_A` no hacemos `sfence.vma`, porque `userret` (en `trampoline.S`) ya lo hace cada vez que se vuelve a modo usuario.
- La máscara se copia con `copyout()`. Si el puntero no es válido, la llamada devuelve -1.

---

# Limitaciones conocidas

- La página compartida solo tiene el pid.
- `pgaccess()` revisa máximo 64 páginas por llamada, porque la máscara es un `uint64`.
- `pgaccess()` no ve los accesos que hace el kernel a la memoria del proceso (por ejemplo en `read()`), porque el kernel usa su propio mapeo.
- Dependemos de que el hardware ponga `PTE_A` en 1 solo. QEMU lo hace; otro procesador podría dar un fallo de página en vez de eso.
- `vmprint()` imprime en la consola, así que el programa no puede leer el resultado.

---

# Declaración de uso de IA

Usamos Claude para entender el enunciado y las pruebas, y principalmente para la  invetigacion del codigo base de xv6, corregir dos errores de `vmprint()` (uno de compilación y el formato de la indentación),