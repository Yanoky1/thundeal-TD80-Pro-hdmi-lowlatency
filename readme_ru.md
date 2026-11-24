ThundeaL TD80Pro и клоны на плате be30a3: задержка HDMI-входа со ~150 мс до ~20–40 мс и честные 60fps. Без прошивок, только ADB

Я нашел способ, как уменьшить задержку при выводе изображения через HDMI (например с ПК или PlayStation или Xbox на проектор для игр). Ниже я прикладываю инструкцию и фотографию своих тестов. У меня получилось сократить задержку со 150 миллисекунд до 50 и увеличить фреймрейт с 30 до 60 кадров в секунду.

Я не уверен, что получится провернуть такое с другими моделями проекторов, но в теории способ ограничения вывода частоты кадров может быть подобный, ИИ в помощь.

!!!!

Я не несу ответственность за поломанное устройство, я лишь рассказываю, как получилось у меня и какой результат.

!!!!

Проверено на сборке 20251112154138 — тесты проводились на ней.

Платформа: Amlogic txhd2, плата be30a3, Android 13 (SDK 33). Прошивка userdebug/test-keys — рут через adb root, никаких разблокировок не нужно. Подходит всем клонам этого ODM (Stellamore STM930/T950S: Tandil, ThundeaL, WZATCO и т.п.) — проверьте себя командами adb shell getprop ro.board.platform (txhd2) и ro.product.board (be30a3).

⚠️ Сначала главное

Цифровое масштабирование и трапеция отменяют весь результат. Сбросьте масштаб в 100% (максимальная картинка) и трапецию в 0, автотрапецию/автофокус по камере — выключить. Причина: без геометрии кадр HDMI-входа идёт напрямую аппаратным слоем (sideband-план, GPU не участвует). Трапеция/зум — произвольная геометрия, которую железный видео-план не умеет, и вендор переводит картинку на GPU-композицию: на Mali-450 и 1 ГБ RAM это +1–2 кадра задержки, и вендор заранее урезывает эту ветку до 30fps. Проекцию выравнивайте физически — ножками/площадкой.

Как я мерил

Камера телефона в режиме 240 fps, на ПК секундомер с миллисекундами при дублировании экрана на проектор, оба экрана в одном кадре. Серия из 10–15 кадров, медиана. Сток у меня: ~150 мс (обзорщики намерили на этой модели ~110).

Почему сток такой медленный — что я нашёл внутри

1. ~73 мс — намеренная задержка видео под аудио (AV-sync). Узлы hdmin_delay_duration/last_required_total_delay — это одна переменная. После перезагрузки они нулевые, «73» вписывает audio HAL при каждом открытии HDMI-входа.
2. Вендорский флаг /sys/class/dummy_venc/encp_projector_fps = 1 урезает вывод до 30fps — при физически 60-герцовой панели (LVDS 2040×1130 @ 138.312 МГц, драйвер панели отчитывает честные 60.00 Гц). Половина кадров входа просто не показывалась.
3. Тяжёлой обработки (AI-PQ) в стоке и так нет, вход 1080p60 RGB без HDCP — конвертаций нет. Остаётся захват vdin и скан-аут.

Готового рецепта под эту плату я не нашёл нигде — ни на 4PDA, ни на XDA, ни на GitHub. Пришлось копать самому: счётчики ядра в sysfs и GPL-исходники Amlogic.

Что понадобится перед стартом

На проекторе включите режим разработчика, отладку по сети и разрешите ADB. На компьютере, с которого будут идти команды (у меня Windows), распакуйте platform-tools, запустите консоль из этой папки, введите adb connect IP-проектора:5555, затем adb devices — проектор должен появиться в списке. Если что-то пойдёт не по плану — Google в помощь.

Быстрая диагностика стока
```
adb shell "cat /sys/class/video/vframe_states | grep delayed"   → 68–72 мс
adb shell "cat /sys/class/dummy_venc/encp_projector_fps"        → 1
adb shell "cd /sys/module/aml_media/parameters; a=$(cat vsync_count); sleep 10; c=$(cat vsync_count); echo vsync_per_s=$(( (c-a)/10 ))"  → 30
```
Разовый запуск (для проверки; перезагрузка откатывает всё)
```
@echo off
adb root
adb shell "echo 0 > /sys/class/dummy_venc/encp_projector_fps"
adb shell "echo 0 > /sys/module/aml_media/parameters/use_low_latency"
adb shell "echo 1 > /sys/module/aml_media/parameters/pc_mode_en"
adb shell "echo 1 > /sys/module/aml_media/parameters/vdin_pc_mode"
adb shell "echo 1 > /sys/class/videoqueue/force_game_mode"
adb shell "echo 0 > /sys/class/video/hdmin_delay_duration"
adb shell "echo 0 > /sys/class/video/last_required_total_delay"
adb shell "echo 0 > /sys/class/video/enable_hdmi_delay_normal_check"
adb shell "echo 1 > /sys/module/aml_media/parameters/freerun_en"
adb shell "echo 1 > /sys/class/video/freerun_mode"
adb shell "echo 0 > /sys/class/dummy_venc/encp_projector_fps"
adb shell "cat /sys/class/video/vframe_states | grep delayed"
```
Контроль после: delayed ≈ 44, флаг 0, vsync_per_s=60.

Профили

Профиль	encp_projector_fps	use_low_latency	Задержка	Примечание
Рекомендуемый	0	0	~40 мс (медиана серии: впиши)	60fps, картинка целая
Спорт	0	1	~20 мс	разрыв на середине экрана при 60fps
Сток	1	0	~150 мс	30fps, как из коробки

Почему у «спорта» разрыв: use_low_latency выводит кадр посреди развёртки, не дожидаясь VSync. При вендорских 30fps линия перекидывания сидела у верхнего края и не мешала, при 60fps уезжает на середину.

Про звук. AV-sync-задержку мы снимаем, и если звук у тебя идёт с проектора — он будет обгонять картинку: для кино верните echo 73 в оба delay-узла или перезагрузитесь. У меня звук идёт напрямую с ПК на колонки, поэтому меня это не касается.

Автозапуск (переживает перезагрузки)

Всё выше — sysfs: после ребута сбрасывается, а tvserver при открытии HDMI заново вписывает свои 73 мс. Поэтому я поставил себе сервис: тяжёлые ручки применяются один раз, остальное держит цикл.

lowlat.sh:
```
#!/system/bin/sh
echo 1 > /sys/module/aml_media/parameters/pc_mode_en
echo 1 > /sys/module/aml_media/parameters/vdin_pc_mode
echo 1 > /sys/class/videoqueue/force_game_mode
echo 1 > /sys/module/aml_media/parameters/freerun_en
echo 1 > /sys/class/video/freerun_mode
echo 0 > /sys/class/dummy_venc/encp_projector_fps
while true; do
  echo 0 > /sys/class/dummy_venc/encp_projector_fps
  echo 0 > /sys/module/aml_media/parameters/use_low_latency
  echo 0 > /sys/class/video/hdmin_delay_duration
  echo 0 > /sys/class/video/last_required_total_delay
  echo 0 > /sys/class/video/enable_hdmi_delay_normal_check
  sleep 1
done
```
lowlat.rc:

```
service lowlat /system/bin/sh /data/local/tmp/lowlat.sh
    user root
    group root
    disabled

on property:sys.boot_completed=1
    start lowlat
```

Установка:

```
adb root
adb shell "mount -o remount,rw /vendor"
adb push lowlat.sh /data/local/tmp/lowlat.sh
adb push lowlat.rc /vendor/etc/init/lowlat.rc
adb shell "sed -i 's/\r$//' /data/local/tmp/lowlat.sh /vendor/etc/init/lowlat.rc"
adb shell "chmod 755 /data/local/tmp/lowlat.sh; chmod 644 /vendor/etc/init/lowlat.rc"
adb reboot
```

Проверка через минуту после загрузки (при открытом HDMI): adb shell "pgrep -f lowlat" даёт PID; флаг 0; delayed ≈ 44; vsync_per_s=60.

Грабли, на которые я наступил (чтобы вы не наступали):

1. Файлы из Windows имеют переводы строк CRLF — init такой .rc молча не парсит. Лечится sed -i 's/\r$//' (в команде установки уже есть).
2. adb remount пишет в overlayfs, который не переживает перезагрузку — файл пропадёт. Используйте прямой mount -o remount,rw /vendor (verity на этой прошивке отключена).
3. Проверяйте сервис через pgrep -f lowlat, а не ps -A | grep lowlat — последний показывает имя sh, а не путь скрипта.
4. После OTA (переключение A/B-слота) сервиса в новом слоте не будет — переустановите.

Откат автозапуска: удалить /vendor/etc/init/lowlat.rc, перезагрузиться.

Не трогать (проверено — бесполезно или опасно)

• /sys/class/display/mode принимает только dummy_p/dummy_panel, посторонние имена молча игнорирует.
• hdmin_delay_min_ms/max_ms — статистика, перезаписывается сама.
• pts_latency (18000) — латентность DVB-потока, к HDMI-входу отношения не имеет.
• /dev/mem, /dev/kmem, /proc/kcore — в ядре вырезаны, «мягкой» правки памяти нет.
• Прошивки от других плат be3xx — кирпич; кастомов под be30a3/txhd2 не существует.

Риски

Ничего не прошивается и ничего не удаляется: все настройки — записи в sysfs, перезагрузка возвращает сток; автозапуск — один добавленный файл в /vendor, который удаляется откатом. Единственный необратимый риск остаётся там же, где и был: попытка прошить чужие образы.
