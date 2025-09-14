# Архитектура прошивки

Прошивка Flix это обычный скетч Arduino, реализованный в однопоточном стиле. Код инициализации находится в функции `setup()`, а главный цикл — в функции `loop()`. Скетч состоит из нескольких файлов, каждый из которых отвечает за определенную подсистему.

<img src="img/dataflow.svg" width=600 alt="Firmware dataflow diagram">

Главный цикл `loop()` работает на частоте 1000 Гц. Передача данных между подсистемами происходит через глобальные переменные:

- `t` _(float)_ — текущее время шага, _с_.
- `dt` _(float)_ — дельта времени между текущим и предыдущим шагами, _с_.
- `gyro` _(Vector)_ — данные с гироскопа, _рад/с_.
- `acc` _(Vector)_ — данные с акселерометра, _м/с<sup>2</sup>_.
- `rates` _(Vector)_ — отфильтрованные угловые скорости, _рад/с_.
- `attitude` _(Quaternion)_ — оценка ориентации (положения) дрона.
- `controlRoll`, `controlPitch`, ... _(float[])_ — команды управления от пилота, в диапазоне [-1, 1].
- `motors` _(float[])_ — выходные сигналы на моторы, в диапазоне [0, 1].

## Исходные файлы

Исходные файлы прошивки находятся в директории `flix`. Основные файлы:

- [`flix.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/flix.ino) — основной файл Arduino-скетча. Определяет некоторые глобальные переменные и главный цикл.
- [`imu.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/imu.ino) — чтение данных с датчика IMU (гироскоп и акселерометр), калибровка IMU.
- [`rc.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/rc.ino) — чтение данных с RC-приемника, калибровка RC.
- [`estimate.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/estimate.ino) — оценка ориентации дрона, комплементарный фильтр.
- [`control.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/control.ino) — подсистема управления, трехмерный двухуровневый каскадный ПИД-регулятор.
- [`motors.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/motors.ino) — выход PWM на моторы.
- [`mavlink.ino`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/mavlink.ino) — взаимодействие с QGroundControl или [pyflix](https://github.com/0xSalik/esp-quadcopter/tree/main/tools/pyflix) через протокол MAVLink.

Вспомогательные файлы:

- [`vector.h`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/vector.h), [`quaternion.h`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/quaternion.h) — библиотеки векторов и кватернионов.
- [`pid.h`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/pid.h) — ПИД-регулятор.
- [`lpf.h`](https://github.com/0xSalik/esp-quadcopter/blob/main/flix/lpf.h) — фильтр нижних частот.

### Подсистема управления

Состояние органов управления обрабатывается в функции `interpretControls()` и преобразуется в _команду управления_, которая включает следующее:

- `attitudeTarget` _(Quaternion)_ — целевая ориентация дрона.
- `ratesTarget` _(Vector)_ — целевые угловые скорости, _рад/с_.
- `ratesExtra` _(Vector)_ — дополнительные (feed-forward) угловые скорости, для управления рысканием в режиме STAB, _рад/с_.
- `torqueTarget` _(Vector)_ — целевой крутящий момент, диапазон [-1, 1].
- `thrustTarget` _(float)_ — целевая общая тяга, диапазон [0, 1].

Команда управления обрабатывается в функциях `controlAttitude()`, `controlRates()`, `controlTorque()`. Если значение одной из переменных установлено в `NAN`, то соответствующая функция пропускается.

<img src="img/control.svg" width=300 alt="Control subsystem diagram">

Состояние _armed_ хранится в переменной `armed`, а текущий режим — в переменной `mode`.
