# Этап 4 — Статический анализ Semgrep

## Выполненные команды

```bash
semgrep ci --code
semgrep ci --code --sarif --sarif-output semgrep-report.sarif
```

## Сводка запуска

| Конфиг | Rules run | Targets scanned | Findings | Статус запуска |
|---|---:|---:|---:|---|
| `semgrep ci --code` | 2884 | 27 | 0 | Успешно |
| `semgrep ci --code --sarif --sarif-output semgrep-report.sarif` | 2884 | 27 | 0 | Успешно, SARIF сохранён |

## Findings

| ruleId | level | file:line | описание | CWE | статус | обоснование |
|---|---|---|---|---|---|---|
| — | — | — | Findings отсутствуют (`results: []`) | — | False Positive | Ложноположительных срабатываний нет, так как Semgrep не вернул ни одного finding. |

## Примечания по SARIF

| Поле | Значение |
|---|---|
| `runs` | 1 |
| `runs[0].results` | 0 элементов |
| `runs[0].invocations[0].toolExecutionNotifications` | 1 warning (`Syntax error at line gradlew:72`) |
