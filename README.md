# Android bridge integration notes

Это исходный шаблон для Android Studio, а не готовый APK. Добавьте предоставленный SSP AAR и сверяйте API с его классами.

Ожидаемый контракт от ПК:
- `operation=status` — только диагностика bridge, не финансовая операция;
- `operation=payment`, `amount=100.00`;
- `operation=cancel`, `amount=100.00`, `rrn=...`;
- `operation=refund`, `amount=100.00`, `rrn=...`;
- `operation=reconciliation`.

По завершении bridge пишет одну строку `SSP_BRIDGE_RESULT:{JSON}` в logcat. Python-клиент очищает logcat перед запуском, чтобы не принять старый результат.
