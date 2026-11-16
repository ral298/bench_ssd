# Benchmark Completo de SSD en Linux (`bench_ssd.sh`)

Script en Bash basado en `fio` diseñado para evaluar el rendimiento real y sostenido de unidades SSD en Linux bajo perfiles equivalentes a **CrystalDiskMark** y cargas de trabajo mixtas reales (como simulaciones EDA/post-layout, compilación paralela y transferencias masivas).

## Características principales

* **Detección automática de unidades:** Si el equipo cuenta con un solo SSD físico (`ROTA=0`), inicia la prueba automáticamente. Si detecta múltiples SSDs, despliega un menú interactivo con modelo, tamaño y puntos de montaje.
* **Compatibilidad con sistemas de archivos modernos:** Resuelve automáticamente volúmenes **Btrfs** (subvolúmenes), **LVM**, **LUKS** y particiones estándar para localizar una ruta con permisos de escritura.
* **Carga real sostenida:** Utiliza un archivo temporal de **2 GB** pre-asignado en memoria flash con **4 pasadas** por escenario (`16 segundos` por prueba) utilizando I/O directo (`--direct=1`) para evitar falsos positivos por caché RAM del sistema operativo.
* **Registro automático por modelo:** Crea la carpeta `./datos_ssds/` y guarda una tabla detallada con fecha, modelo, revisión de firmware, velocidad (`MB/s`), `IOPS` y latencia (`us`) en un archivo nombrado según el SSD evaluado (por ejemplo, `./datos_ssds/sn520.txt` o `./datos_ssds/sn810.txt`).
* **Limpieza automática:** Elimina el archivo temporal de 2 GB al finalizar o si el script se interrumpe (`trap cleanup EXIT`).

## Escenarios evaluados (14 pruebas)

1. **`SEQ1M Q8T1` (Lectura / Escritura):** Velocidad secuencial máxima bajo perfil NVMe.
2. **`SEQ1M Q1T1` (Lectura / Escritura):** Rendimiento secuencial en un solo hilo (transferencia real de archivos grandes).
3. **`SEQ128K Q32T1` (Lectura / Escritura):** Streaming secuencial con cola profunda.
4. **`RND4K Q32T16` (Lectura / Escritura):** Techo máximo de IOPS aleatorios multi-hilo (`16 hilos x 32 colas`).
5. **`RND4K Q32T1` (Lectura / Escritura):** Acceso aleatorio con cola profunda en un hilo.
6. **`RND4K Q1T1` (Lectura / Escritura):** Agilidad del sistema operativo y lectura/escritura de miles de archivos pequeños.
7. **`MIX 70/30` (`SEQ1M Q8T1` y `RND4K Q32T1`):** Lectura (70%) y escritura (30%) simultáneas para medir comportamiento bajo multitarea pesada y efectividad de caché DRAM.

## Instalación y uso

### 1. Crear el script `bench_ssd.sh`

Copia y pega el siguiente bloque en tu terminal para generar el archivo `bench_ssd.sh`, darle permisos de ejecución y correrlo (el script instalará automáticamente `fio` y `jq` mediante `apt` si no están presentes):

```bash
cat << 'EOF' > ~/bench_ssd.sh
#!/usr/bin/env bash
export LC_ALL=C
set -euo pipefail

# 1. Verificar dependencias
for cmd in fio jq awk lsblk df tee; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "Instalando dependencias necesarias (fio, jq)..."
        sudo apt update && sudo apt install -y fio jq gawk util-linux coreutils
        break
    fi
done

# Función auxiliar para obtener el disco físico padre (soporta Btrfs, LVM, LUKS y particiones)
get_parent_disk() {
    local dev=$1
    local cur=${dev#/dev/}
    cur=${cur%%\[*}
    while [[ -n "$cur" ]]; do
        local dtype pk
        dtype=$(lsblk -d -n -o TYPE "/dev/$cur" 2>/dev/null | xargs || true)
        if [[ "$dtype" == "disk" ]]; then
            echo "$cur"
            return 0
        fi
        pk=$(lsblk -d -n -o PKNAME "/dev/$cur" 2>/dev/null | xargs || true)
        [[ -z "$pk" ]] && break
        cur="$pk"
    done
    echo ""
}

# 2. Detectar SSDs físicos (ROTA=0, excluyendo zram/loop/ram)
mapfile -t SSDS < <(lsblk -d -n -o NAME,ROTA,TYPE | awk '$2 == "0" && $3 == "disk" && $1 !~ /^(zram|ram|fd|sr)/ {print $1}')

if [[ ${#SSDS[@]} -eq 0 ]]; then
    echo "Error: No se detectó ningún SSD en el sistema."
    exit 1
elif [[ ${#SSDS[@]} -eq 1 ]]; then
    SELECTED_DEV="${SSDS[0]}"
    echo "SSD único detectado automáticamente: /dev/$SELECTED_DEV"
else
    echo "================================================================================"
    echo "  Se detectaron múltiples SSDs en tu equipo:"
    echo "================================================================================"
    for i in "${!SSDS[@]}"; do
        dev="${SSDS[$i]}"
        mdl=$(lsblk -d -n -o MODEL "/dev/$dev" 2>/dev/null | xargs || echo "Desconocido")
        sz=$(lsblk -d -n -o SIZE "/dev/$dev" 2>/dev/null | xargs || echo "?")
        mnts=$(lsblk -nr -o MOUNTPOINT "/dev/$dev" 2>/dev/null | awk 'NF > 0 && $0 != "[SWAP]"' | paste -sd ", " -)
        [[ -z "$mnts" ]] && mnts="(Sin montar)"
        printf "  [%d] /dev/%-10s | %-28s | %-7s | Montado en: %s\n" "$((i+1))" "$dev" "$mdl" "$sz" "$mnts"
    done
    echo "--------------------------------------------------------------------------------"
    while true; do
        read -rp "Elige el número del SSD que quieres evaluar [1-${#SSDS[@]}]: " choice
        if [[ "$choice" =~ ^[0-9]+$ ]] && (( choice >= 1 && choice <= ${#SSDS[@]} )); then
            SELECTED_DEV="${SSDS[$((choice-1))]}"
            break
        fi
        echo "Opción no válida. Ingresa un número entre 1 y ${#SSDS[@]}."
    done
fi

# 3. Localizar ruta con permisos de escritura dentro del SSD seleccionado
HOME_SRC=$(df --output=source "$HOME" | tail -n1)
HOME_DISK=$(get_parent_disk "$HOME_SRC")

TARGET_DIR=""
if [[ "$HOME_DISK" == "$SELECTED_DEV" && -w "$HOME" ]]; then
    TARGET_DIR="$HOME"
else
    while IFS= read -r mnt; do
        [[ -z "$mnt" || "$mnt" == "[SWAP]" ]] && continue
        if [[ -w "$mnt" ]]; then
            TARGET_DIR="$mnt"
            break
        fi
    done < <(lsblk -nr -o MOUNTPOINT "/dev/$SELECTED_DEV" 2>/dev/null)
fi

if [[ -z "$TARGET_DIR" ]]; then
    echo "Error: No se encontró ninguna partición montada con permisos de escritura en /dev/$SELECTED_DEV."
    exit 1
fi

AVAIL_KB=$(df --output=avail "$TARGET_DIR" | tail -n1 | xargs)
if (( AVAIL_KB < 3145728 )); then
    echo "Error: Espacio insuficiente en $TARGET_DIR. Se requieren al menos 3 GB libres."
    exit 1
fi

# 4. Preparar directorio y archivo de salida por modelo de SSD
OUT_DIR="./datos_ssds"
mkdir -p "$OUT_DIR"

MODEL_NAME=$(lsblk -d -n -o MODEL "/dev/$SELECTED_DEV" 2>/dev/null | xargs || echo "$SELECTED_DEV")
DISK_SIZE=$(lsblk -d -n -o SIZE "/dev/$SELECTED_DEV" 2>/dev/null | xargs || echo "")
FW_REV=$(lsblk -d -n -o REV "/dev/$SELECTED_DEV" 2>/dev/null | xargs || echo "N/A")

SHORT_MODEL=$(echo "$MODEL_NAME" | grep -oiE 'sn[0-9]{3,4}[a-z0-9]*' | head -n1 | tr '[:upper:]' '[:lower:]' || true)
if [[ -n "$SHORT_MODEL" ]]; then
    FILE_SLUG="$SHORT_MODEL"
else
    FILE_SLUG=$(echo "$MODEL_NAME" | tr '[:upper:]' '[:lower:]' | sed -E 's/[^a-z0-9]+/_/g; s/^_|_$//g')
fi
[[ -z "$FILE_SLUG" ]] && FILE_SLUG="$SELECTED_DEV"

OUT_FILE="${OUT_DIR}/${FILE_SLUG}.txt"
: > "$OUT_FILE"

TEST_FILE="$TARGET_DIR/.ssd_bench_2g_tmp.bin"
SIZE="2G"
LOOPS=4
RUNTIME=4 # 4 pasadas x 4s = 16s por prueba sobre el archivo de 2 GB

cleanup() {
    rm -f "$TEST_FILE"
}
trap cleanup EXIT

echo ""
echo "Preparando archivo de prueba de $SIZE en $TARGET_DIR..."
fio --name=prep --filename="$TEST_FILE" --size="$SIZE" --bs=1M --iodepth=8 \
    --ioengine=libaio --direct=1 --rw=write --output-format=json >/dev/null

echo "Ejecutando las 14 pruebas (aprox. 3.5 min en total, guardando en $OUT_FILE)..."
echo ""

{
    echo "================================================================================"
    echo "  BENCHMARK COMPLETO SSD - Perfil CrystalDiskMark + Cargas Reales/Mixtas"
    echo "  Fecha    : $(date '+%Y-%m-%d %H:%M:%S')"
    echo "  Unidad   : /dev/$SELECTED_DEV ($MODEL_NAME - $DISK_SIZE | FW: $FW_REV)"
    echo "  Config   : Archivo de $SIZE x $LOOPS pasadas por prueba | Ruta: $TARGET_DIR"
    echo "================================================================================"
    printf "%-27s | %-15s | %-11s | %-12s\n" "Prueba" "Velocidad" "IOPS" "Latencia"
    echo "--------------------------------------------------------------------------------"
} | tee -a "$OUT_FILE"

run_single() {
    local label=$1
    local rw=$2
    local bs=$3
    local iodepth=$4
    local numjobs=$5

    local json_key="read"
    [[ "$rw" == *write* ]] && json_key="write"

    local json
    json=$(fio --output-format=json --group_reporting=1 \
        --ioengine=libaio --direct=1 --filename="$TEST_FILE" --size="$SIZE" \
        --loops="$LOOPS" --time_based --runtime="$RUNTIME" \
        --name=job --rw="$rw" --bs="$bs" --iodepth="$iodepth" --numjobs="$numjobs")

    local bw iops lat_ns mb_s iops_int lat_us
    bw=$(echo "$json" | jq -r ".jobs[0].${json_key}.bw_bytes")
    iops=$(echo "$json" | jq -r ".jobs[0].${json_key}.iops")
    lat_ns=$(echo "$json" | jq -r ".jobs[0].${json_key}.lat_ns.mean")

    mb_s=$(awk "BEGIN {printf \"%.2f\", $bw/1000000}")
    iops_int=$(awk "BEGIN {printf \"%.0f\", $iops}")
    lat_us=$(awk "BEGIN {printf \"%.1f\", $lat_ns/1000}")

    printf "%-27s | %10s MB/s | %11s | %9s us\n" "$label" "$mb_s" "$iops_int" "$lat_us" | tee -a "$OUT_FILE"
}

run_mix() {
    local label=$1
    local rw=$2
    local bs=$3
    local iodepth=$4

    local json
    json=$(fio --output-format=json --group_reporting=1 \
        --ioengine=libaio --direct=1 --filename="$TEST_FILE" --size="$SIZE" \
        --loops="$LOOPS" --time_based --runtime="$RUNTIME" \
        --name=job --rw="$rw" --rwmixread=70 --bs="$bs" --iodepth="$iodepth" --numjobs=1)

    local r_bw w_bw r_iops w_iops r_lat w_lat mb_s iops_int lat_us
    r_bw=$(echo "$json" | jq -r ".jobs[0].read.bw_bytes")
    w_bw=$(echo "$json" | jq -r ".jobs[0].write.bw_bytes")
    r_iops=$(echo "$json" | jq -r ".jobs[0].read.iops")
    w_iops=$(echo "$json" | jq -r ".jobs[0].write.iops")
    r_lat=$(echo "$json" | jq -r ".jobs[0].read.lat_ns.mean")
    w_lat=$(echo "$json" | jq -r ".jobs[0].write.lat_ns.mean")

    mb_s=$(awk "BEGIN {printf \"%.2f\", ($r_bw + $w_bw)/1000000}")
    iops_int=$(awk "BEGIN {printf \"%.0f\", $r_iops + $w_iops}")
    lat_us=$(awk "BEGIN {printf \"%.1f\", (($r_lat * 0.7) + ($w_lat * 0.3))/1000}")

    printf "%-27s | %10s MB/s | %11s | %9s us\n" "$label" "$mb_s" "$iops_int" "$lat_us" | tee -a "$OUT_FILE"
}

sep_line() {
    echo "--------------------------------------------------------------------------------" | tee -a "$OUT_FILE"
}

# --- 1. Secuencial Máximo (Perfil NVMe) ---
run_single "SEQ1M Q8T1    (Lectura)"    "read"      "1M"   8  1
run_single "SEQ1M Q8T1    (Escritura)"  "write"     "1M"   8  1
sep_line

# --- 2. Secuencial Hilo Único (Copia real de archivos grandes .vcd/.gds) ---
run_single "SEQ1M Q1T1    (Lectura)"    "read"      "1M"   1  1
run_single "SEQ1M Q1T1    (Escritura)"  "write"     "1M"   1  1
sep_line

# --- 3. Secuencial Streaming Multi-cola ---
run_single "SEQ128K Q32T1 (Lectura)"    "read"      "128k" 32 1
run_single "SEQ128K Q32T1 (Escritura)"  "write"     "128k" 32 1
sep_line

# --- 4. Aleatorio Extremo NVMe (16 hilos x 32 colas = techo máximo de IOPS) ---
run_single "RND4K Q32T16  (Lectura)"    "randread"  "4k"   32 16
run_single "RND4K Q32T16  (Escritura)"  "randwrite" "4k"   32 16
sep_line

# --- 5. Aleatorio Cola Profunda 1 Hilo ---
run_single "RND4K Q32T1   (Lectura)"    "randread"  "4k"   32 1
run_single "RND4K Q32T1   (Escritura)"  "randwrite" "4k"   32 1
sep_line

# --- 6. Aleatorio Hilo Único (Agilidad del SO, scripts Python/cocotb) ---
run_single "RND4K Q1T1    (Lectura)"    "randread"  "4k"   1  1
run_single "RND4K Q1T1    (Escritura)"  "randwrite" "4k"   1  1
sep_line

# --- 7. Carga Mixta Simultánea 70% Lectura / 30% Escritura (Multitarea pesada) ---
run_mix    "MIX 70/30 SEQ1M Q8T1"      "rw"        "1M"   8
run_mix    "MIX 70/30 RND4K Q32T1"     "randrw"    "4k"   32

echo "================================================================================" | tee -a "$OUT_FILE"
echo "Archivo temporal de $SIZE eliminado."
echo "Resultados guardados exitosamente en: $OUT_FILE"
EOF

chmod +x ~/bench_ssd.sh
~/bench_ssd.sh
```

### 2. Comparar resultados entre dos SSDs

Una vez ejecutado el benchmark en dos unidades diferentes, puedes comparar ambos reportes lado a lado desde la terminal con:

```bash
diff -y -W 170 ./datos_ssds/sn520.txt ./datos_ssds/sn810.txt
```