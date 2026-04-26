```markdown
Введение
========

Проект **PPCI (Pure Python Compiler Infrastructure)** — это компилятор,
написанный полностью на языке программирования `Python <https://www.python.org/>`_.
Он содержит фронтенды для различных языков программирования, а также
функциональность для генерации машинного кода. С помощью этой библиотеки
вы можете генерировать (работающий!) машинный код, используя Python
(что очень удобно для изучения, расширения и т.д.).

Проект включает:

- Фронтенды для языков: C, Python, Pascal, Basic и Brainfuck
- Генерацию кода для нескольких архитектур: 6500, arm, avr, m68k, microblaze, msp430, openrisc, risc-v, stm8, x86_64, xtensa
- Утилиты командной строки: ppci-cc, ppci-ld, ppci-opt и другие
- Поддержку WebAssembly, JVM, OCaml
- Поддержку форматов ELF, EXE, S-record и hex-файлов
- Промежуточное представление (IR), которое может быть сериализовано в JSON
- Проект может использоваться как библиотека для скриптовой сборки

Установка
---------

### Установка из PyPI (официальная версия)

Поскольку компилятор — это Python-пакет, его можно установить с помощью pip:

```bash
pip install ppci
```

### Установка из GitHub (форк с поддержкой русских строк)

Если вам нужна поддержка кириллицы в строковых литералах, используйте форк с исправленной кодировкой:

```bash
pip install git+https://github.com/artradeskz/ppci.git
```

Пример компиляции
-----------------

### Простейшая программа на C с русским текстом

Создайте файл `hello.c`:

```c
void puts(char *s);
void putc(char c);
void exit(int status);
void syscall(long nr, long a, long b, long c);

int main()
{
    puts("Привет, мир!");
    exit(0);
}

void puts(char *s)
{
    while (*s) {
        putc(*s++);
    }
    putc('\n');
}

void putc(char c)
{
    syscall(1, 1, (long int)&c, 1);
}

void exit(int status)
{
    syscall(60, status, 0, 0);
}

void syscall(long nr, long a, long b, long c)
{
    asm(
        "mov rax, %0 \n"
        "mov rdi, %1 \n"
        "mov rsi, %2 \n"
        "mov rdx, %3 \n"
        "syscall \n"
        :
        : "r" (nr), "r" (a), "r" (b), "r" (c)
        : "rax", "rdi", "rsi", "rdx"
    );
}
```

### Скрипт линковки `linux64.ld`

Сохраните следующий файл для линковки под Linux x86_64:

```ld
MEMORY code LOCATION=0x40000 SIZE=0x10000 {
    SECTION(code)
}

MEMORY ram LOCATION=0x20000000 SIZE=0xA000 {
    SECTION(data)
}
```

### Компиляция и запуск

```bash
# Компиляция
ppci-cc -c hello.c -o hello.o

# Линковка
ppci-ld --entry main --layout linux64.ld hello.o -o hello

# Запуск
./hello
```

**Ожидаемый вывод:**

```
Привет, мир!
```

### Пример с математическими вычислениями

Более сложный пример: факториал, числа Фибоначчи, возведение в степень.

```c
void syscall(long nr, long a, long b, long c);
void exit(int status);

void print_number(long num) {
    char buffer[20];
    int i = 0;
    
    if (num < 0) {
        syscall(1, 1, (long)"-", 1);
        num = -num;
    }
    
    if (num == 0) {
        syscall(1, 1, (long)"0", 1);
        return;
    }
    
    while (num > 0) {
        buffer[i++] = '0' + (num % 10);
        num = num / 10;
    }
    
    while (i > 0) {
        syscall(1, 1, (long)&buffer[--i], 1);
    }
}

void print_char(char c) {
    syscall(1, 1, (long)&c, 1);
}

void print_string(char *s) {
    while (*s) {
        print_char(*s++);
    }
}

long factorial(long n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

long factorial_iter(long n) {
    long result = 1;
    long i;
    for (i = 2; i <= n; i++) {
        result = result * i;
    }
    return result;
}

long fibonacci(long n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

long power(long base, long exp) {
    long result = 1;
    long i;
    for (i = 0; i < exp; i++) {
        result = result * base;
    }
    return result;
}

void exit(int status) {
    syscall(60, status, 0, 0);
}

int main() {
    long n = 5;
    long m = 10;
    long i;
    long result;
    
    print_string("=== Математические вычисления ===\n\n");
    
    print_string("Факториал ");
    print_number(n);
    print_string(" = ");
    print_number(factorial(n));
    print_string(" (рекурсивно)\n");
    
    print_string("Факториал ");
    print_number(m);
    print_string(" = ");
    print_number(factorial_iter(m));
    print_string(" (итеративно)\n\n");
    
    print_string("Числа Фибоначчи:\n");
    for (i = 0; i <= 10; i++) {
        print_string("F(");
        print_number(i);
        print_string(") = ");
        print_number(fibonacci(i));
        print_char('\n');
    }
    print_char('\n');
    
    print_string("2^10 = ");
    print_number(power(2, 10));
    print_char('\n');
    
    print_string("3^5 = ");
    print_number(power(3, 5));
    print_char('\n');
    
    result = (5 + 3) * 4 - 12 / 3;
    print_string("\n(5 + 3) * 4 - 12 / 3 = ");
    print_number(result);
    print_char('\n');
    
    exit(0);
}

void syscall(long nr, long a, long b, long c) {
    asm(
        "mov rax, %0 \n"
        "mov rdi, %1 \n"
        "mov rsi, %2 \n"
        "mov rdx, %3 \n"
        "syscall \n"
        :
        : "r" (nr), "r" (a), "r" (b), "r" (c)
        : "rax", "rdi", "rsi", "rdx"
    );
}
```

Результат работы:

```
=== Математические вычисления ===

Факториал 5 = 120 (рекурсивно)
Факториал 10 = 3628800 (итеративно)

Числа Фибоначчи:
F(0) = 0
F(1) = 1
F(2) = 1
F(3) = 2
F(4) = 3
F(5) = 5
F(6) = 8
F(7) = 13
F(8) = 21
F(9) = 34
F(10) = 55

2^10 = 1024
3^5 = 243

(5 + 3) * 4 - 12 / 3 = 28
```

Пример использования API
------------------------

Компиляция C-кода через Python API:

```python
import io
from ppci.api import cc, link

source_file = io.StringIO("""
int printf(char* fmt) { }

void main() {
    printf("Hello world!\\n");
}
""")
obj = cc(source_file, 'x86_64')
obj = link([obj])
```

Ассемблирование кода через API:

```python
import io
from ppci.api import asm

source_file = io.StringIO("""section code
pop rbx
push r10
mov rdi, 42""")
obj = asm(source_file, 'x86_64')
print(obj.get_section('code').data)
```

Низкоуровневый API:

```python
from ppci.arch.x86_64 import instructions, registers
i = instructions.Pop(registers.rbx)
print(i.encode())  # b'['
```

Функциональность
----------------

- Утилиты командной строки:
    - `ppci-cc`
    - `ppci-ld`
    - и другие
- Поддержка языков: C, Pascal, Python, Basic, Brainfuck, C3
- Поддержка процессоров: 6500, arm, avr, m68k, microblaze, msp430, openrisc, risc-v, stm8, x86_64, xtensa
- Поддержка форматов: ELF, COFF PE (EXE), hex, S-record
- Использует человекочитаемые форматы (JSON, XML)

Документация
------------

Официальная документация: https://ppci.readthedocs.io/

⚠️ **Внимание:** Проект находится в альфа-стадии и не готов к промышленному использованию!

---

## 🔧 Патч для поддержки кириллицы

Если вы используете официальную версию PPCI и хотите добавить поддержку русских строк, измените в файле:

```
~/.local/lib/python3.10/site-packages/ppci/lang/c/nodes/expressions.py
```

Строку:
```python
encoding = "latin1"
```

Замените на:
```python
encoding = "utf-8"
```

Или просто используйте форк:
```bash
pip install git+https://github.com/artradeskz/ppci.git
```

