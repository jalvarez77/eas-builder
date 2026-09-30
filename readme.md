# EAS Build Local con Docker (jaccessapp)

Build container separado del devcontainer, para compilar APKs sin gastar cuota del plan Free de EAS.

## 1. Imagen base (`eas-android-builder`)

Dockerfile guardado aparte. Contiene: JDK 17, Node 20, `eas-cli`, Android SDK (API 36 / build-tools 36.0.0, según Expo SDK 54), y `git config --global --add safe.directory /workspace`.

> Si sube el Expo SDK y cambia el API level, actualizar `platforms;android-XX` y `build-tools;XX.0.0` en el Dockerfile y rebuildear.

Build de la imagen (solo cuando cambie el Dockerfile):

```bash
docker build -t ghcr.io/jalvarez77/eas-android-builder:1.0.0 .
```

## 2. Access Token de Expo

Generar en: https://expo.dev/accounts/jalvarez77/settings/access-tokens

Guardarlo como variable de entorno persistente en WSL2:

```bash
echo 'export EXPO_TOKEN="tu-token-real-aqui"' >> ~/.bashrc
source ~/.bashrc
```

Verificar que quedó seteada:

```bash
echo $EXPO_TOKEN
```

## 3. Wrapper script

Archivo: `~/.local/bin/eas-build-local-jaccessapp`

```bash
#!/bin/bash
docker run --rm -t \
  --dns 1.1.1.1 --dns 8.8.8.8 \
  -e EXPO_TOKEN \
  -v ~/projects/jaccess/jaccessfe:/workspace \
  -v eas-gradle-cache:/root/.gradle \
  ghcr.io/jalvarez77/eas-android-builder:1.0.0 \
  eas "$@"
```

Crear y dar permisos:

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/eas-build-local-jaccessapp
chmod +x ~/.local/bin/eas-build-local-jaccessapp
```

Asegurar que `~/.local/bin` esté en el PATH (si `which eas-build-local-jaccessapp` no lo encuentra):

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Config properties de gradle

```bash
docker run --rm -v eas-gradle-cache:/root/.gradle --entrypoint bash \
  ghcr.io/jalvarez77/eas-android-builder:1.0.0 -c 'cat > /root/.gradle/gradle.properties <<EOF
systemProp.org.gradle.internal.http.connectionTimeout=60000
systemProp.org.gradle.internal.http.socketTimeout=60000
systemProp.org.gradle.internal.repository.max.retries=5
systemProp.org.gradle.internal.repository.initial.backoff=1000
reactNativeArchitectures=armeabi-v7a,arm64-v8a
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g
org.gradle.parallel=true
org.gradle.caching=true
EOF'
```

Qué hace cada grupo:

- Las 4 líneas systemProp...: hacen que Gradle reintente las descargas si la red falla.
- reactNativeArchitectures=armeabi-v7a,arm64-v8a: compila solo para teléfonos reales (quita x86 y x86_64, que son para emuladores). Sirve también para la tienda.
- Las 3 últimas: más memoria y builds más rápidos.

## 4. Correr el build

```bash
eas-build-local-jaccessapp build --platform android --profile production --local
```

El APK queda en `~/projects/jaccess/jaccessfe/` (mismo directorio del proyecto, montado como volumen).

## 5. Troubleshooting

**Error "dubious ownership" de Git** → ya resuelto en el Dockerfile con `git config --global --add safe.directory /workspace`.

**`config --json exited with non-zero code: 1` sin más detalle** → correr manualmente dentro del container para ver el error real:

```bash
docker run --rm -it \
  -e EXPO_TOKEN \
  -v ~/projects/jaccess/jaccessfe:/workspace \
  -v eas-gradle-cache:/root/.gradle \
  eas-android-builder \
  bash

cd /workspace
node ./node_modules/expo/bin/cli config
```

**`PluginError: Failed to resolve plugin for module "X"`** → `node_modules` incompleto o inconsistente con el entorno del container. Reinstalar dentro del mismo container de build:

```bash
rm -rf node_modules
npm install
```

## Notas

- Un solo Dockerfile/imagen sirve para jaccessapp y SalesHub — solo cambia el volumen `-v` del proyecto en el wrapper de cada uno.
- El SDK de Android vive horneado en la imagen (no en volumen) porque no cambia entre proyectos.
- El cache de Gradle (`eas-gradle-cache`) sí es volumen persistente — acelera builds siguientes.
- Si sube el Expo SDK y cambia el API level, actualizar `platforms;android-XX` y `build-tools;XX.0.0` en el Dockerfile y rebuildear la imagen.