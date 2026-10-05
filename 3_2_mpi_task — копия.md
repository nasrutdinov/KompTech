# Задания «Компьютерные технологии» (3 семестр)
Ниже — 12 заданий по MPI на C++ с готовыми шаблонами кода и инструкциями для запуска на ноутбуках студентов. Для каждого задания есть: цель, условие, шаблон кода, что проверить, и как оформить на слайде презентации (в духе твоих прошлых запросов).

---

## Задание 1. Проверка числа процессов и корректное завершение
**Цель:** базовая инициализация MPI, проверка `size`, безопасная остановка.  
**Условие:** программа должна работать только при запуске ровно на 4 процессах. Иначе — вывести сообщение и завершить через `MPI_Abort` после барьера.  

**Шаблон (C++):**
```cpp
#include <mpi.h>
#include <iostream>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (size != 4) {
        if (rank == 0) std::cout << "Error: need exactly 4 processes, got " << size << "\n";
        MPI_Barrier(MPI_COMM_WORLD);
        MPI_Abort(MPI_COMM_WORLD, 1);
    }

    std::cout << "Process " << rank << " running successfully.\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** запустить `mpirun -np 2 ./prog` — должна аварийно остановиться; `mpirun -np 4 ./prog` — все процессы печатают «running successfully».  
**Слайд:** цель, условие, ключевые функции (`MPI_Init`, `MPI_Comm_size`, `MPI_Barrier`, `MPI_Abort`), ожидаемый вывод, график времени запуска при 2/4/8 процессах.

---

## Задание 2. Двусторонний обмен массивами (Send/Recv) без deadlock
**Цель:** избежать взаимной блокировки при обмене.  
**Условие:** процессы 0 и 1 обмениваются массивами `int` длины `N`. Сначала 0→1, потом 1→0.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>
#include <vector>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    const int N = 4;
    std::vector<int> sendBuf(N), recvBuf(N);

    for (int i = 0; i < N; ++i) sendBuf[i] = rank * 10 + i;

    if (rank == 0) {
        MPI_Send(sendBuf.data(), N, MPI_INT, 1, 0, MPI_COMM_WORLD);
        MPI_Recv(recvBuf.data(), N, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    } else if (rank == 1) {
        MPI_Recv(recvBuf.data(), N, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        MPI_Send(sendBuf.data(), N, MPI_INT, 0, 0, MPI_COMM_WORLD);
    }

    std::cout << "Rank " << rank << ": received [";
    for (int x : recvBuf) std::cout << x << " ";
    std::cout << "]\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** вывод совпадает с ожидаемым (0 получил данные от 1 и наоборот). Поменять порядок вызовов у одного из процессов — проверить deadlock.  
**Слайд:** цель, схема обмена, код (ключевые строки), ожидаемый вывод, объяснение deadlock и как его избежать.

---

## Задание 3. Обмен через MPI_Sendrecv
**Цель:** безопасный двусторонний обмен.  
**Условие:** те же два процесса, обмен одной функцией `MPI_Sendrecv`.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>
#include <vector>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    const int N = 4;
    std::vector<int> sendBuf(N), recvBuf(N);
    for (int i = 0; i < N; ++i) sendBuf[i] = rank * 100 + i;

    int target = (rank == 0 ? 1 : 0);
    MPI_Sendrecv(sendBuf.data(), N, MPI_INT, target, 0,
                 recvBuf.data(), N, MPI_INT, target, 0,
                 MPI_COMM_WORLD, MPI_STATUS_IGNORE);

    std::cout << "Rank " << rank << ": received [";
    for (int x : recvBuf) std::cout << x << " ";
    std::cout << "]\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** корректный обмен, отсутствие deadlock даже при любом порядке.  
**Слайд:** сравнение с Заданием 2, преимущества `Sendrecv`, код, вывод, сравнение по читаемости и надёжности.

---

## Задание 4. Кольцевой сдвиг рангов (Ring shift)
**Цель:** топология «кольцо», вычисление соседей по модулю.  
**Условие:** каждый процесс отправляет свой `rank` следующему (по модулю `size`). Последний отправляет первому.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int next = (rank + 1) % size;
    int prev = (rank - 1 + size) % size; // не используется, но полезно знать

    int sendVal = rank;
    int recvVal;

    MPI_Send(&sendVal, 1, MPI_INT, next, 0, MPI_COMM_WORLD);
    MPI_Recv(&recvVal, 1, MPI_INT, prev, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

    std::cout << "I am process " << rank << ", received from " << prev << "\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** каждый процесс печатает «получил от» правильного соседа. Запустить на 3, 4, 8 процессах.  
**Слайд:** схема кольца, формулы `next`/`prev`, код, ожидаемый вывод, зависимость времени от числа процессов.

---

## Задание 5. Реверс массива символов через Sendrecv
**Цель:** обмен отдельными элементами, изменение порядка.  
**Условие:** массив символов распределён по одному символу на процесс. Переслать так, чтобы порядок стал обратным.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>
#include <string>
#include <vector>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    char sendChar = 'A' + rank;
    char recvChar;
    int target = size - 1 - rank;

    MPI_Sendrecv(&sendChar, 1, MPI_CHAR, target, 0,
                 &recvChar, 1, MPI_CHAR, target, 0,
                 MPI_COMM_WORLD, MPI_STATUS_IGNORE);

    std::cout << "Rank " << rank << ": sent '" << sendChar
              << "', received '" << recvChar << "'\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** при `size=5` процесс 0 получает `'E'`, процесс 4 получает `'A'` и т.д.  
**Слайд:** идея реверсии, формула `target`, код, таблица соответствия, накладные расходы на мелкие сообщения.

---

## Задание 6. Scatter + Gather для сборки данных
**Цель:** коллективные операции распределения и сборки.  
**Условие:** процесс 0 содержит массив `int`, распределить через `Scatter`, собрать обратно на процессе 1 через `Gather`.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>
#include <vector>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    const int N_PER_PROC = 4;
    const int TOTAL = N_PER_PROC * size;

    std::vector<int> sendbuf, recvbuf(N_PER_PROC);

    if (rank == 0) {
        sendbuf.resize(TOTAL);
        for (int i = 0; i < TOTAL; ++i) sendbuf[i] = i;
    }

    MPI_Scatter(sendbuf.data(), N_PER_PROC, MPI_INT,
                recvbuf.data(), N_PER_PROC, MPI_INT,
                0, MPI_COMM_WORLD);

    // Каждый процесс увеличивает свои элементы на rank
    for (auto &x : recvbuf) x += rank;

    std::vector<int> gatherbuf;
    if (rank == 1) gatherbuf.resize(TOTAL);

    MPI_Gather(recvbuf.data(), N_PER_PROC, MPI_INT,
               gatherbuf.data(), N_PER_PROC, MPI_INT,
               1, MPI_COMM_WORLD);

    if (rank == 1) {
        std::cout << "Gathered on rank 1: [";
        for (int x : gatherbuf) std::cout << x << " ";
        std::cout << "]\n";
    }

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** на процессе 1 собранный массив корректен. Сравнить с ручной рассылкой.  
**Слайд:** диаграмма Scatter/Gather, код, вывод, сравнение объёма кода и производительности.

---

## Задание 7. Параллельное суммирование через Reduce
**Цель:** агрегация результатов.  
**Условие:** локальная сумма на каждом процессе, глобальная сумма на процессе 0 через `Reduce`.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>
#include <vector>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    const int N = 1000;
    long long localSum = 0;
    for (int i = 0; i < N; ++i) localSum += (rank * N + i);

    long long globalSum;
    MPI_Reduce(&localSum, &globalSum, 1, MPI_LONG_LONG, MPI_SUM, 0, MPI_COMM_WORLD);

    if (rank == 0)
        std::cout << "Global sum = " << globalSum << "\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** результат совпадает с последовательным расчётом. Построить график времени от `N` и `size`.  
**Слайд:** логика Reduce, код, формула ожидаемой суммы, графики ускорения.

---

## Задание 8. Broadcast параметров задачи
**Цель:** рассылка одинаковых данных всем процессам.  
**Условие:** процесс 0 задаёт параметры (размер, шаг) и рассылает через `Bcast`. Процессы используют их для вычислений.  

**Шаблон:**
```cpp
#include <mpi.h>
#include <iostream>

struct Params {
    int N;
    double step;
};

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    Params p;
    if (rank == 0) { p.N = 1000; p.step = 0.01; }

    MPI_Bcast(&p, 1, MPI_BYTES, 0, MPI_COMM_WORLD); // или создать тип MPI_Datatype

    // Для простоты используем отдельные Bcast:
    MPI_Bcast(&p.N, 1, MPI_INT, 0, MPI_COMM_WORLD);
    MPI_Bcast(&p.step, 1, MPI_DOUBLE, 0, MPI_COMM_WORLD);

    std::cout << "Rank " << rank << ": N=" << p.N << ", step=" << p.step << "\n";

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** все процессы получают одинаковые значения.  
**Слайд:** когда Broadcast лучше серии Send, код, сравнение времени при малом/большом объёме.

---

## Задание 9. Приём сообщений неизвестного типа и размера (Probe/Get_count)
**Цель:** динамические сообщения, определение типа и размера.  
**Условие:** процесс 0 отправляет сообщения разного типа (int/double/char) в произвольном порядке. Процесс 1 определяет тип и размер через `Probe` и `Get_count`, затем корректно принимает.  

**Шаблон (упрощённый, фиксированные типы):**
```cpp
#include <mpi.h>
#include <iostream>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (rank == 0) {
        int x = 123;
        double y = 3.14;
        char z = 'Z';
        MPI_Send(&x, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
        MPI_Send(&y, 1, MPI_DOUBLE, 1, 1, MPI_COMM_WORLD);
        MPI_Send(&z, 1, MPI_CHAR, 1, 2, MPI_COMM_WORLD);
    } else {
        MPI_Status status;
        MPI_Probe(0, MPI_ANY_TAG, MPI_COMM_WORLD, &status);
        int count;
        MPI_Get_count(&status, MPI_BYTE, &count); // для демонстрации; реальный тип нужно знать
        // В реальном коде тип должен быть известен заранее, Probe помогает узнать count
        int val;
        MPI_Recv(&val, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, &status);
        std::cout << "Received int: " << val << "\n";
    }

    MPI_Finalize();
    return 0;
}
```

**Что проверить:** корректность приёма, накладные расходы Probe.  
**Слайд:** зачем нужен Probe, ограничения, код, анализ накладных расходов.

---

## Задание 10. Параллельная формула трапеции (интегрирование)
**Цель:** реальная вычислительная задача.  
**Условие:** разделить отрезок на части по числу процессов, каждый считает свою часть, глобальный результат через `Reduce`.  
