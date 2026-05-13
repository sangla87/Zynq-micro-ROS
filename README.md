# Zynq-micro-ROS
micro‑ROS en Zynq (Zybo Z7‑10, Cortex‑A9, FreeRTOS) flujo completo (Docker + ROS2 + micro‑ROS + Cortex‑A9 + FreeRTOS).

````markdown
# micro-ROS en Zynq (Zybo Z7-10, Cortex-A9, FreeRTOS)

Guía paso a paso para generar la librería estática de micro-ROS en un entorno reproducible usando Docker (Ubuntu 22.04), con soporte de comunicación por Ethernet (UDP).

---

# 🚀 1. Crear contenedor Docker

Ejecutar en el host:

```bash
docker run -it --name microros \
  -v ~/Descargas:/host_files \
  -v /opt/Xilinx:/opt/Xilinx \
  ubuntu:22.04 /bin/bash
````

## 📌 Explicación

*   `/host_files` → acceso a archivos del host (configuración)
*   `/opt/Xilinx` → acceso a toolchain de Vitis
*   entorno completamente aislado

***

# 🧱 2. Instalar dependencias base

Dentro del contenedor:

```bash
apt update && apt install -y \
  curl gnupg lsb-release git python3-pip \
  build-essential cmake \
  python3-colcon-common-extensions \
  python3-rosdep
```

***

# ⚙️ 3. Instalar ROS 2 (Humble)

```bash
apt install -y software-properties-common
add-apt-repository universe

curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | apt-key add -

echo "deb http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" \
  > /etc/apt/sources.list.d/ros2.list

apt update
apt install -y ros-humble-desktop
```

***

# 🔧 4. Inicializar rosdep

```bash
rosdep init
rosdep update
```

⚠️ Importante:

*   `rosdep init` → requiere permisos root (en Docker ya lo eres)
*   `rosdep update` → ejecutar sin sudo

***

# 📁 5. Crear workspace micro-ROS

```bash
source /opt/ros/humble/setup.bash

mkdir ~/microros_ws
cd ~/microros_ws

git clone https://github.com/micro-ROS/micro_ros_setup src/micro_ros_setup

rosdep update
rosdep install --from-paths src --ignore-src -y

colcon build
source install/setup.bash
```

***

# 🧪 6. Generar firmware

```bash
ros2 run micro_ros_setup create_firmware_ws.sh generate_lib
```

***

# 📁 7. Preparar configuración

```bash
mkdir ~/microros_ws/config
```

Copiar archivos desde el host:

```bash
cp /host_files/colcon.meta ~/microros_ws/config/
cp /host_files/toolchain.cmake ~/microros_ws/config/
```

***

# ⚙️ 8. Configuración `colcon.meta`

## ✅ Versión final (Ethernet UDP + FreeRTOS)

```json
{
  "names": {
    "tracetools": {
      "cmake-args": [
        "-DTRACETOOLS_DISABLED=ON",
        "-DTRACETOOLS_STATUS_CHECKING_TOOL=OFF"
      ]
    },
    "rosidl_typesupport": {
      "cmake-args": [
        "-DROSIDL_TYPESUPPORT_SINGLE_TYPESUPPORT=ON"
      ]
    },
    "rcl": {
      "cmake-args": [
        "-DBUILD_TESTING=OFF",
        "-DRCL_COMMAND_LINE_ENABLED=OFF",
        "-DRCL_LOGGING_ENABLED=OFF"
      ]
    },
    "rcutils": {
      "cmake-args": [
        "-DENABLE_TESTING=OFF",
        "-DRCUTILS_NO_FILESYSTEM=ON",
        "-DRCUTILS_NO_THREAD_SUPPORT=ON",
        "-DRCUTILS_NO_64_ATOMIC=ON",
        "-DRCUTILS_AVOID_DYNAMIC_ALLOCATION=ON"
      ]
    },
    "microxrcedds_client": {
      "cmake-args": [
        "-DUCLIENT_PIC=OFF",
        "-DUCLIENT_PROFILE_UDP=ON",
        "-DUCLIENT_PROFILE_TCP=OFF",
        "-DUCLIENT_PROFILE_SERIAL=OFF",
        "-DUCLIENT_PROFILE_DISCOVERY=OFF",
        "-DUCLIENT_PROFILE_STREAM_FRAMING=ON",
        "-DUCLIENT_PROFILE_CUSTOM_TRANSPORT=OFF",
        "-DUCLIENT_PROFILE_MULTITHREAD=OFF",
        "-DUCLIENT_PROFILE_SHARED_MEMORY=OFF",
        "-DUCLIENT_PLATFORM_FREERTOS_PLUS_TCP=ON"
      ]
    },
    "rmw_microxrcedds": {
      "cmake-args": [
        "-DRMW_UXRCE_MAX_NODES=16",
        "-DRMW_UXRCE_MAX_PUBLISHERS=32",
        "-DRMW_UXRCE_MAX_SUBSCRIPTIONS=32",
        "-DRMW_UXRCE_MAX_HISTORY=8"
      ]
    }
  }
}
```

***

# ⚙️ 9. Toolchain (`toolchain.cmake`)

## ✅ Cortex-A9 + Vitis

```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_CROSSCOMPILING 1)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER /opt/Xilinx/Vitis/2022.2/gnu/aarch32/lin/gcc-arm-none-eabi/bin/arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER /opt/Xilinx/Vitis/2022.2/gnu/aarch32/lin/gcc-arm-none-eabi/bin/arm-none-eabi-g++)

set(CMAKE_C_COMPILER_WORKS 1 CACHE INTERNAL "")
set(CMAKE_CXX_COMPILER_WORKS 1 CACHE INTERNAL "")

set(FLAGS "-Wall -Wextra -Os -ffunction-sections -fdata-sections -nostdlib -flto -mthumb -mcpu=cortex-a9 -mfpu=neon-vfpv3 -mfloat-abi=hard -mtune=cortex-a9 -DCLOCK_MONOTONIC=0" CACHE STRING "" FORCE)

set(CMAKE_C_FLAGS_INIT "-std=c11 ${FLAGS}" CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS_INIT "-std=c++11 ${FLAGS}" CACHE STRING "" FORCE)
```

***

# 🔨 10. Compilar micro-ROS

```bash
cd ~/microros_ws

ros2 run micro_ros_setup build_firmware.sh \
  $(pwd)/config/toolchain.cmake \
  $(pwd)/config/colcon.meta
```

***

# 📦 11. Resultado

```bash
firmware/build/libmicroros.a
```

***

# 🌐 12. Ejecutar agente ROS2

```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

***

# ⚠️ Problemas comunes

| Problema                | Causa                    |
| ----------------------- | ------------------------ |
| toolchain no encontrado | `/opt/Xilinx` no montado |
| no conecta ROS2         | UDP deshabilitado        |
| fallo en runtime red    | falta stack TCP/IP       |
| build falla             | flags incorrectos        |

***

# 🧠 Notas

*   Cortex-A9 → ARM 32-bit
*   FreeRTOS → entorno bare-metal (sin Linux)
*   micro-ROS genera librería independiente del sistema

***

# ✅ Comandos útiles

Entrar al contenedor:

```bash
docker start microros
docker exec -it microros /bin/bash
```

Alias opcional:

```bash
alias microros="docker exec -it microros /bin/bash"
```

***

# 🚀 Próximos pasos

*   Integrar `libmicroros.a` en Vitis
*   Configurar red (FreeRTOS+TCP)
*   Implementar nodo micro-ROS

***

```

---

## ✅ Tip rápido para GitHub

Cuando lo pegues:
- usa **editor Markdown**
- guarda como `README.md` en la raíz del repo

---

Si quieres, en el siguiente paso puedo ayudarte a añadir:
- diagrama del flujo (Docker → build → Vitis)
- o sección específica de integración en Vitis (muy útil para tu caso)
```
