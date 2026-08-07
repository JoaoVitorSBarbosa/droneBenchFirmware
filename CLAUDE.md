# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Conventions

Commit messages, PR titles, PR descriptions, and branch names must be written in **English**, following the Conventional Commits standard:

```
<type>: <short imperative summary>

Optional body explaining the why, not the what.
```

Common types: `feat`, `fix`, `refactor`, `docs`, `chore`.

- Subject line: 50 chars max, no period at the end
- Body: wrap at 72 chars, separated from subject by a blank line
- PR title mirrors the main commit message
- PR body: `## Summary` (bullet list of changes) + `## Test plan` (checklist)

## Build & Flash Commands

All commands use the PlatformIO CLI (`pio`). The single build target is `esp32-s3-n16r8v`.

```bash
pio run                          # compilar
pio run --target upload          # compilar e gravar no ESP32-S3
pio run --target uploadfs        # gravar filesystem LittleFS (páginas web em data/)
pio device monitor               # abrir monitor serial (115200 baud)
pio run --target upload && pio device monitor   # gravar e monitorar em sequência
```

> `uploadfs` é necessário sempre que arquivos em `data/` forem modificados — o filesystem é uma partição separada do firmware.

## Credenciais Wi-Fi

O arquivo `include/env.h` **não está no repositório** (gitignored). Copie o molde e preencha:

```bash
cp include/env_example.h include/env.h
# edite AP_SSID e AP_PASSWORD em env.h
```

## Arquitetura

### Visão geral

Firmware embarcado para um drone de bancada (1-DOF / 2-DOF), rodando em **ESP32-S3 DevKitC-1** com FreeRTOS em dual-core. O objetivo é uma malha de controle determinística a 100 Hz com telemetria paralela.

### Tasks FreeRTOS (`src/main.cpp`)

| Task | Core | Frequência | Responsabilidade |
|------|------|------------|-----------------|
| `taskSensor` | 0 | 50 Hz | Lê `IMU::read()` e envia dados para `sensorQueue` |
| `taskTelemetry` | 0 | 50 Hz | Consome `telemQueue`, imprime CSV via Serial e envia JSON via SSE ao browser |
| `taskControl` | 1 | 100 Hz | Consome `sensorQueue`, calcula erro, aciona motores e envia `TelemData` para `telemQueue` |
| `taskCalibration` | 0 | event-driven | Consome `calibCmdQueue`, executa rotinas de `Calibration` (IMU, motores, encoders, reset) |

A comunicação entre tasks é exclusivamente via **FreeRTOS queues** (`sensorQueue` depth 5, `telemQueue` depth 10, `motorCmdQueue`/`calibCmdQueue`/`ctrlParamsQueue` depth 1, overwrite). O `loop()` deleta a própria task — padrão correto para FreeRTOS no Arduino.

### Fluxo de dados

```
IMU::read() ──► sensorQueue ──► taskControl ──► Motor::setVelocidade()
                                      └──► telemQueue ──► taskTelemetry ──► Serial CSV
                                                                       └──► WebManager::sendTelemetry() ──► SSE /api/events
```

### Classes principais

- **`IMU`** (`include/IMU.h`): encapsula MPU-6050 via Adafruit MPU6050 (I2C) + leitura opcional de dois encoders. Produz `SensorData` com pitch (acelerômetro), yaw (integração giroscópio), e ângulos dos encoders.
- **`Motor`** (`include/Motor.h`): abstração LEDC PWM para driver bidirecional (RPWM/LPWM). `setVelocidade(float)` aceita range `[-MOTOR_PWM_MAX_DUTY, +MOTOR_PWM_MAX_DUTY]`.
- **`Encoder`** (`include/Encoder.h`): encoder óptico em quadratura via interrupção (`IRAM_ATTR`). 600 PPR × 4 = 2400 pulsos/volta.
- **`WebManager`** (`include/WebManager.h`): servidor HTTP assíncrono (ESPAsyncWebServer). Expõe `attachMotorQueue()`/`attachCalibQueue()`/`attachCtrlParamsQueue()` para ligar as filas do controlador e `sendTelemetry(json)`/`sendCalibStatus(json)` para SSE.

### Interface Web

O ESP32 sobe como **Access Point** (credenciais em `env.h`). Páginas servidas via LittleFS:

| Rota | Arquivo | Função |
|------|---------|--------|
| `/` | `data/index.html` | Telemetria, configuração dos ganhos, calibração, teste de motor e rotina de malha aberta |
| `POST /api/test/motors` | — | Aciona motores em modo teste: params `dutyPitch`, `dutyYaw` |
| `POST /api/test/stop` | — | Para ambos os motores |
| `GET /api/events` | — | SSE stream com JSON de telemetria (`{pitch, yaw, encPitch, encYaw, uPitch, uYaw, ax, ay, az, mode}`) |

### PWM dos Motores

- Frequência: 20 kHz (inaudível)
- Resolução atual: 8 bits (duty 0–255). Máximo possível a 20 kHz: **11 bits** (duty 0–2047)
- Para aumentar: alterar `MOTOR_PWM_RESOLUTION` e `MOTOR_PWM_MAX_DUTY` em `constants.h`

### Lei de controle

Implementada em `taskControl` (`src/main.cpp`), realimentação de estados com ação integral (LQI) e anti-windup por canal — não é mais um placeholder proporcional:

- `DOF1`: `uPitch = Ki1·∫(rp−θ)dt − (Kx1·θ + Kx2·θ̇)`, `uYaw = −uPitch`
- `DOF2`: acoplado pitch+yaw — `uPitch = Ki1·∫p + Ki2·∫y − (Kx1·θ + Kx2·Ω + Kx3·θ̇ + Kx4·Ω̇)`, `uYaw = Ki3·∫p + Ki4·∫y − (Kx5·θ + Kx6·Ω + Kx7·θ̇ + Kx8·Ω̇)`

Os ganhos (`Ki1-4`, `Kx1-8`) são ajustáveis em runtime via `/api/params` (aba Controle da web) e persistidos com defaults em `data/parameters.json`.

### Warning conhecido

Nenhum warning de biblioteca conhecido para a Adafruit MPU6050.
