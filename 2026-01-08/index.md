# Learning Log: 2026-01-08

- **[TOCTOU Race Condition in Go](toctou-race-condition-golang.md)**: Fixed Time-of-Check-Time-of-Use race condition in trading bot cooldown mechanism by using atomic check-and-set under single write lock
