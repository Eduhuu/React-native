# React Native con Expo en WSL2 + Android físico por USB

## Resumen

Configuración para desarrollar con React Native (Expo) en WSL2 y probar la app en un dispositivo Android físico conectado por cable USB.

---

## Problema

WSL2 tiene su propia red NAT aislada. Esto causa dos problemas:

1. **ADB no tiene acceso al USB** conectado al host Windows.
2. **El dispositivo Android no puede alcanzar el Metro bundler** que corre en WSL (IP `172.x.x.x`).

---

## Solución

Se usan dos herramientas en combinación:

- **usbipd-win**: comparte el USB de Windows con WSL para que ADB nativo de Linux vea el dispositivo.
- **adb reverse**: crea un túnel por USB para que el Android acceda a los puertos de Metro en WSL como si fueran `localhost`.

---

## Configuración inicial (solo una vez)

### 1. Instalar usbipd-win en Windows

En PowerShell como administrador:

```powershell
winget install usbipd
```

### 2. Compartir el dispositivo USB con WSL

Conecta el Android por USB y en PowerShell como administrador:

```powershell
# Lista los dispositivos USB y anota el BUSID de tu Android (ej: 1-8)
usbipd list

# Registra el dispositivo (solo la primera vez)
usbipd bind --busid 1-8

# Adjunta el dispositivo a WSL
usbipd attach --wsl --busid 1-8
```

> El `bind` solo se hace una vez. El `attach` hay que repetirlo cada vez que se reconecte el cable o se reinicie Windows.

---

## Uso diario

### Cada vez que conectas el dispositivo

**1. En PowerShell admin (Windows):**

```powershell
usbipd attach --wsl --busid 1-8
```

**2. En WSL, si ADB da problemas:**

```bash
adb kill-server && adb start-server
```

**3. Verifica que el dispositivo aparece:**

```bash
adb devices
```

**4. Arranca la app:**

```bash
npm run start
```

El script `start` ya incluye el reverse tunnel y arranca Expo automáticamente.

---

## Scripts configurados en package.json

```json
"start": "adb reverse tcp:8081 tcp:8081 && adb reverse tcp:19000 tcp:19000 && adb reverse tcp:19001 tcp:19001 && adb reverse tcp:19002 tcp:19002 && expo start --localhost",
"android": "adb reverse tcp:8081 tcp:8081 && adb reverse tcp:19000 tcp:19000 && adb reverse tcp:19001 tcp:19001 && adb reverse tcp:19002 tcp:19002 && expo start --localhost --android",
```

- `npm run start` — Arranca Metro con el tunnel configurado.
- `npm run android` — Igual que start pero abre directamente en Android.

---

## Por qué funciona el adb reverse

`adb reverse tcp:PUERTO tcp:PUERTO` crea un túnel inverso por el cable USB:

```
Android (localhost:8081) ──USB──► WSL (localhost:8081 = Metro bundler)
```

Al usar `expo start --localhost`, Expo anuncia `localhost` como dirección del bundler en lugar de la IP de WSL (que el dispositivo no puede alcanzar). El dispositivo se conecta a su propio `localhost:8081`, que ADB redirige transparentemente a WSL.

---

## Solución de problemas

| Síntoma                                            | Solución                                                                         |
| -------------------------------------------------- | -------------------------------------------------------------------------------- |
| `adb devices` da timeout                           | `adb kill-server && adb start-server`                                            |
| `usbipd attach` falla con "device used by Windows" | `taskkill /F /IM adb.exe` en PowerShell, luego reintentar                        |
| Expo da timeout en Android                         | Verificar que el `adb reverse` se ejecutó correctamente con `adb reverse --list` |
| El dispositivo no aparece tras reconectar          | Repetir `usbipd attach --wsl --busid 1-8` en PowerShell admin                    |
