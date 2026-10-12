# SSP AutoTests — Python + pytest + ADB

Стартовый каркас автотестов для SomersPOS / SmartSkyPOS API 1.9.18 SR2.

## Архитектура

Windows + pytest -> ADB -> Android SSP Test Bridge -> SSP Intent API -> тестовый хост.
Карту прикладывают вручную, когда терминал показывает экран ожидания карты.

**Важно:** Python-часть готова к запуску, но Android bridge требует AAR SSP API из вашей поставки. В PDF указано, что для вызова SSP нужно использовать `TransactionParams` и `PaymentActivity`; точные package/imports и доступность полей зависят от предоставленной AAR. Поэтому Android-файл — интеграционный шаблон, его нужно сверить с AAR и собрать на Android Studio. Это не готовый подписанный APK.

## Требования на ПК

- Windows 10/11
- Python 3.11+ (рекомендуется 3.12 или 3.13; если установлен Python 3.14, можно попробовать, но для стабильной совместимости плагинов лучше отдельное окружение 3.12/3.13)
- Android SDK Platform Tools (`adb`) в PATH
- терминал с разрешённым ADB debugging
- тестовый хост и тестовые карты

## Установка

```powershell
cd ssp_autotests
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
adb devices -l
```

Если `adb devices -l` показывает `unauthorized`, разблокируйте терминал и подтвердите RSA-диалог. Если подключено больше одного устройства, задайте serial в `config.json`.

## Подключение Android bridge

1. Откройте `android-bridge` в Android Studio.
2. Добавьте в модуль `app` AAR-библиотеку SSP 1.9.18 SR2, выданную командой SSP.
3. Сверьте импорты `TransactionParams`, `PaymentActivity`, имена ключей и констант с AAR.
4. Доделайте отмеченные `TODO` в `MainActivity.kt`, соберите APK и установите его на терминал.
5. Убедитесь, что package/activity совпадают с `config.json`.

Документация SSP рекомендует Intent для финансовых операций:
- action: `com.skytech.smartskypos.PAYMENT`
- package SSP: `com.skytech.smartskypos`
- type: `payment`, `cancel`, `refund`, `reconciliation`
- `TransactionParams.amount`; для `cancel` и `refund` также `rrn`
- результат: `transactionResult` или `reconciliationResult`

Для закрытия смены API AIDL описывает `reconciliation()` без параметров. Проверьте, поддерживает ли установленная версия SSP Intent-тип `reconciliation` в вашем окружении. Если нет — реализуйте закрытие смены через AIDL в bridge.

## Проверить связь

```powershell
python -m ssp_tests.cli devices
python -m ssp_tests.cli bridge-check
```

`bridge-check` проверяет запуск Activity/доставку команды, но не выполняет оплату.

## Запуск тестов

Оплата на 100.00:
```powershell
pytest -m payment -s
```

Оплата на заданную сумму:
```powershell
$env:SSP_AMOUNT = "100.00"
pytest -m payment -s
```

Отмена/возврат требуют RRN успешно проведённой тестовой операции:
```powershell
$env:SSP_RRN = "123456789012"
$env:SSP_AMOUNT = "100.00"
pytest -m cancel -s
pytest -m refund -s
```

Сверка итогов:
```powershell
pytest -m reconciliation -s
```

Все интеграционные тесты по умолчанию имеют маркер `manual_card`: pytest запустит операцию, а тест ожидает, пока вы приложите карту. Не запускайте несколько финансовых операций параллельно.

## JSON-результат bridge

Bridge должен записать в Android logcat одну строку с префиксом `SSP_BRIDGE_RESULT:` и JSON, например:
```json
{"operation":"payment","activityResult":"OK","code":0,"isApproved":true,"rc":"00","rrn":"123456789012","message":""}
```

Для reconciliation:
```json
{"operation":"reconciliation","activityResult":"OK","code":0,"rc":"00","message":"","transactionLogCount":3}
```

Негативный результат должен тоже возвращаться JSON, а не молча завершать Activity. В JSON не включайте PAN, PIN, CVV, ключи и другие чувствительные платёжные данные.

## Ограничения безопасности и тестирования

- Только тестовый терминал/хост и разрешённые тестовые карты.
- Отмена и возврат выполняются только по RRN тестовой транзакции.
- Не запускайте тест возврата автоматически после неуспешной/неопределённой оплаты.
- При timeout сначала проверьте состояние SSP и журнал/результат операции: не повторяйте финансовую команду вслепую.
- Закрытие смены — изменяющая состояние операция; запускайте отдельным тестом, когда готовы закрыть тестовую смену.
