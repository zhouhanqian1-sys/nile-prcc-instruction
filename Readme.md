# NILE PRCC Instruction

NOTE: The paths below are examples from one working PRCC run. Replace them with
your own PRCC paths before running this workflow.

## 1. Configure Paths

```bash
# NILE software root.
export NILE_ROOT=/data/pro00118610/NILE

# Writable working directory for scripts, prepared input, compiled classes,
# logs, and output.
export WORK_DIR=/data/pro00110219/zq63/ICU_SOFA/nile_smoke_test

# Input CSV file prepared by the user.
export INPUT_CSV=$WORK_DIR/input/nile_input.csv
```

Run this workflow inside PRCC OnDemand, such as Code Server Terminal, RE Cluster
Shell Access, or an OnDemand terminal. Do not run it on DCC login.

## 2. Prepare Input CSV

The input must be a CSV file with a header row and these four columns:

```text
patient_num,encounter_num,start_date,notes
```

Column meanings:

```text
patient_num   Stable patient identifier. Durable Patient Key can be used here.
encounter_num Stable encounter, document, or note identifier.
start_date    Note date or report date.
notes         Clinical note text.
```

Example:

```csv
patient_num,encounter_num,start_date,notes
123456,987654,2024-04-23,"Patient reports shortness of breath."
123456,987655,2024-04-24,"No evidence of pneumonia."
```

If your column names are different, set column-name variables before running:

```bash
export PATIENT_COL="Durable Patient Key"
export ENCOUNTER_COL="Encounter Key"
export START_DATE_COL="Report Date"
export NOTES_COL="Report Text"
```

## 3. Create `setup_conda_java.sh`

Create this file in `$WORK_DIR`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Creates a Conda environment with a full JDK, including javac.

ENV_NAME="${ENV_NAME:-nile-java}"
OPENJDK_VERSION="${OPENJDK_VERSION:-21}"

if ! command -v conda >/dev/null 2>&1; then
  echo "[ERROR] conda was not found on PATH."
  exit 1
fi

CONDA_BASE="$(conda info --base)"
source "$CONDA_BASE/etc/profile.d/conda.sh"

if conda env list | awk '{print $1}' | grep -qx "$ENV_NAME"; then
  echo "[INFO] Conda env already exists: $ENV_NAME"
  conda install -y -n "$ENV_NAME" -c conda-forge "openjdk=$OPENJDK_VERSION"
else
  echo "[INFO] Creating Conda env: $ENV_NAME"
  conda create -y -n "$ENV_NAME" -c conda-forge "openjdk=$OPENJDK_VERSION"
fi

conda activate "$ENV_NAME"
hash -r

echo "[INFO] Java runtime:"
which java
java -version

echo "[INFO] Java compiler:"
which javac
javac -version

echo "[INFO] Ready. Run:"
echo "conda activate $ENV_NAME"
echo "bash run_nile_smoke_test.sh"
```

Make it executable:

```bash
chmod 700 setup_conda_java.sh  # Restrict script access to the current user.
```

## 4. Create `run_nile_smoke_test.sh`

Create this file in `$WORK_DIR`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Generic CSV-based NILE pipeline for PRCC.
# It validates a user-provided input CSV, compiles NILECSVProcessor into a
# writable work directory, and runs NILE CSV mode.

umask 077

NILE_ROOT="${NILE_ROOT:-/data/pro00118610/NILE}"
WORK_DIR="${WORK_DIR:-/data/pro00110219/zq63/ICU_SOFA/nile_smoke_test}"
INPUT_CSV="${INPUT_CSV:-$WORK_DIR/input/nile_input.csv}"

NROWS="${NROWS:-200}"
NOTE_CHAR_LIMIT="${NOTE_CHAR_LIMIT:-8000}"
RUN_NILE="${RUN_NILE:-1}"
SKIP_COMPILE="${SKIP_COMPILE:-0}"

PATIENT_COL="${PATIENT_COL:-patient_num}"
ENCOUNTER_COL="${ENCOUNTER_COL:-encounter_num}"
START_DATE_COL="${START_DATE_COL:-start_date}"
NOTES_COL="${NOTES_COL:-notes}"

INPUT_DIR="$WORK_DIR/input"
OUTPUT_DIR="$WORK_DIR/output"
LOG_DIR="$WORK_DIR/logs"
CLASSES_DIR="$WORK_DIR/classes"

RUN_STAMP="$(date +%Y%m%d_%H%M%S)"
PREPARED_CSV="$INPUT_DIR/nile_input_prepared_${NROWS}_${RUN_STAMP}.csv"
OUTPUT_CSV="$OUTPUT_DIR/nile_output_${NROWS}_${RUN_STAMP}.csv"
MANIFEST="$LOG_DIR/manifest_${RUN_STAMP}.txt"
LOG_FILE="$LOG_DIR/run_${RUN_STAMP}.log"

mkdir -p "$INPUT_DIR" "$OUTPUT_DIR" "$LOG_DIR" "$CLASSES_DIR"

main() {
echo "[INFO] NILE CSV pipeline started at $(date)"
echo "[INFO] NILE_ROOT=$NILE_ROOT"
echo "[INFO] WORK_DIR=$WORK_DIR"
echo "[INFO] INPUT_CSV=$INPUT_CSV"
echo "[INFO] PREPARED_CSV=$PREPARED_CSV"
echo "[INFO] OUTPUT_CSV=$OUTPUT_CSV"
echo "[INFO] NROWS=$NROWS"
echo "[INFO] Input columns: $PATIENT_COL, $ENCOUNTER_COL, $START_DATE_COL, $NOTES_COL"

if [ ! -f "$INPUT_CSV" ]; then
  echo "[ERROR] INPUT_CSV not found: $INPUT_CSV"
  exit 10
fi

if ! command -v python3 >/dev/null 2>&1; then
  echo "[ERROR] python3 was not found on PATH."
  exit 11
fi

echo "[INFO] Validating and preparing input CSV..."
python3 - "$INPUT_CSV" "$PREPARED_CSV" "$NROWS" "$NOTE_CHAR_LIMIT" "$MANIFEST" \
  "$PATIENT_COL" "$ENCOUNTER_COL" "$START_DATE_COL" "$NOTES_COL" <<'PY'
import csv
import re
import sys
from pathlib import Path

input_csv = Path(sys.argv[1])
prepared_csv = Path(sys.argv[2])
nrows = int(sys.argv[3])
note_char_limit = int(sys.argv[4])
manifest_path = Path(sys.argv[5])
patient_col_arg = sys.argv[6]
encounter_col_arg = sys.argv[7]
start_date_col_arg = sys.argv[8]
notes_col_arg = sys.argv[9]

try:
    csv.field_size_limit(sys.maxsize)
except OverflowError:
    csv.field_size_limit(2_147_483_647)

def norm_name(name):
    return re.sub(r"[^a-z0-9]+", "", str(name).lower())

def clean_key(value):
    if value is None:
        return ""
    value = str(value).replace("\x00", "").strip()
    value = re.sub(r"\.0$", "", value)
    return value

def clean_text(value):
    if value is None:
        return ""
    text = str(value).replace("\x00", " ")
    text = re.sub(r"[\r\n\t]+", " ", text)
    text = re.sub(r"\s+", " ", text).strip()
    if note_char_limit > 0 and len(text) > note_char_limit:
        text = text[:note_char_limit].rstrip()
    return text

def find_col(fieldnames, requested):
    by_norm = {norm_name(name): name for name in fieldnames}
    key = norm_name(requested)
    if key in by_norm:
        return by_norm[key]
    raise SystemExit(
        "[ERROR] Required input column not found: "
        + requested
        + "\n[INFO] Available columns: "
        + ", ".join(fieldnames)
    )

prepared_csv.parent.mkdir(parents=True, exist_ok=True)

with input_csv.open("r", encoding="utf-8-sig", newline="") as in_handle:
    reader = csv.DictReader(in_handle)
    if not reader.fieldnames:
        raise SystemExit("[ERROR] Input CSV has no header.")

    patient_col = find_col(reader.fieldnames, patient_col_arg)
    encounter_col = find_col(reader.fieldnames, encounter_col_arg)
    start_date_col = find_col(reader.fieldnames, start_date_col_arg)
    notes_col = find_col(reader.fieldnames, notes_col_arg)

    scanned = written = skipped_missing_id = skipped_missing_text = 0

    with prepared_csv.open("w", encoding="utf-8", newline="") as out_handle:
        writer = csv.DictWriter(
            out_handle,
            fieldnames=["patient_num", "encounter_num", "start_date", "notes"],
        )
        writer.writeheader()

        for row in reader:
            scanned += 1
            patient_num = clean_key(row.get(patient_col))
            encounter_num = clean_key(row.get(encounter_col))
            start_date = clean_key(row.get(start_date_col))
            notes = clean_text(row.get(notes_col))

            if not patient_num or not encounter_num or not start_date:
                skipped_missing_id += 1
                continue
            if not notes:
                skipped_missing_text += 1
                continue

            writer.writerow(
                {
                    "patient_num": patient_num,
                    "encounter_num": encounter_num,
                    "start_date": start_date,
                    "notes": notes,
                }
            )
            written += 1

            if nrows > 0 and written >= nrows:
                break

print(f"[INFO] Source columns used: {patient_col}, {encounter_col}, {start_date_col}, {notes_col}")
print(f"[INFO] Rows scanned: {scanned}")
print(f"[INFO] Rows written: {written}")
print(f"[INFO] Rows skipped for missing id/date: {skipped_missing_id}")
print(f"[INFO] Rows skipped for missing text: {skipped_missing_text}")
print(f"[INFO] Prepared CSV: {prepared_csv}")

if written == 0:
    raise SystemExit("[ERROR] No usable rows were written.")

with manifest_path.open("w", encoding="utf-8") as manifest:
    manifest.write("NILE CSV pipeline manifest\n")
    manifest.write(f"input_csv={input_csv}\n")
    manifest.write(f"prepared_csv={prepared_csv}\n")
    manifest.write(f"source_columns={patient_col},{encounter_col},{start_date_col},{notes_col}\n")
    manifest.write("prepared_columns=patient_num,encounter_num,start_date,notes\n")
    manifest.write(f"nrows_requested={nrows}\n")
    manifest.write(f"rows_scanned={scanned}\n")
    manifest.write(f"rows_written={written}\n")
    manifest.write(f"note_char_limit={note_char_limit}\n")
PY

chmod 600 "$PREPARED_CSV" "$MANIFEST"

if [ "$RUN_NILE" != "1" ]; then
  echo "[INFO] RUN_NILE=$RUN_NILE, stopping after input preparation."
  exit 0
fi

if [ ! -f "$NILE_ROOT/NILE/NILECSVProcessor.java" ]; then
  echo "[ERROR] NILECSVProcessor.java not found under $NILE_ROOT/NILE"
  exit 20
fi

if [ ! -f "$NILE_ROOT/NILE/NER_dictionary.txt" ]; then
  echo "[ERROR] NILE dictionary not found: $NILE_ROOT/NILE/NER_dictionary.txt"
  exit 21
fi

JAVA_BIN="$(command -v java || true)"
JAVAC_BIN="$(command -v javac || true)"

if [ -z "$JAVA_BIN" ] || [ -z "$JAVAC_BIN" ]; then
  echo "[ERROR] java and javac must both be available. Activate the Conda JDK env."
  exit 22
fi

JAVA_MAJOR="$("$JAVA_BIN" -version 2>&1 | awk -F '[\".]' '/version/ { if ($2 == "1") print $3; else print $2; exit }')"
JAVA_MAJOR="${JAVA_MAJOR:-0}"

if [ "$JAVA_MAJOR" -ge 20 ] && [ -f "$NILE_ROOT/NILE/lib/nile_20.jar" ]; then
  NILE_JAR="nile_20.jar"
elif [ "$JAVA_MAJOR" -ge 11 ] && [ -f "$NILE_ROOT/NILE/lib/nile_11.jar" ]; then
  NILE_JAR="nile_11.jar"
elif [ "$JAVA_MAJOR" -ge 8 ] && [ -f "$NILE_ROOT/NILE/lib/nile_8.jar" ]; then
  NILE_JAR="nile_8.jar"
else
  echo "[ERROR] No compatible nile_*.jar found for Java major version: $JAVA_MAJOR"
  exit 23
fi

echo "[INFO] Java runtime: $JAVA_BIN"
"$JAVA_BIN" -version 2>&1 | sed 's/^/[INFO] /'
echo "[INFO] Selected NILE jar: $NILE_JAR"

CSV_CP="$CLASSES_DIR:.:NILE/lib/$NILE_JAR:NILE/lib/opencsv-5.12.0.jar:NILE/lib/commons-lang3-3.12.0.jar:NILE/lib/commons-collections4-4.4.jar"

cd "$NILE_ROOT"

if [ "$SKIP_COMPILE" != "1" ]; then
  echo "[INFO] Compiling NILECSVProcessor into $CLASSES_DIR"
  "$JAVAC_BIN" -d "$CLASSES_DIR" -cp "$CSV_CP" NILE/NILECSVProcessor.java
else
  echo "[INFO] SKIP_COMPILE=1, using existing compiled classes."
fi

echo "[INFO] Running NILE CSV processor..."
"$JAVA_BIN" -cp "$CSV_CP" NILE.NILECSVProcessor \
  --dictionary-path NILE/NER_dictionary.txt \
  --input-csv "$PREPARED_CSV" \
  --output-csv "$OUTPUT_CSV" \
  --patient-num-col patient_num \
  --encounter-num-col encounter_num \
  --start-date-col start_date \
  --notes-col notes

chmod 600 "$OUTPUT_CSV"

python3 - "$OUTPUT_CSV" "$MANIFEST" <<'PY'
from pathlib import Path
import sys

output_csv = Path(sys.argv[1])
manifest_path = Path(sys.argv[2])

rows = 0
nonempty_codes = 0
with output_csv.open("r", encoding="utf-8", errors="replace") as handle:
    header = handle.readline().strip()
    for line in handle:
        rows += 1
        parts = line.rstrip("\n").split("|")
        if len(parts) >= 4 and parts[3].strip():
            nonempty_codes += 1

print(f"[INFO] NILE output CSV: {output_csv}")
print(f"[INFO] NILE output header: {header}")
print(f"[INFO] NILE output rows: {rows}")
print(f"[INFO] Rows with non-empty code_certainty: {nonempty_codes}")

with manifest_path.open("a", encoding="utf-8") as manifest:
    manifest.write(f"output_csv={output_csv}\n")
    manifest.write(f"nile_output_rows={rows}\n")
    manifest.write(f"nile_rows_with_nonempty_codes={nonempty_codes}\n")
PY

echo "[INFO] Manifest: $MANIFEST"
echo "[INFO] NILE CSV pipeline completed at $(date)"
}

set +e
(
  set -euo pipefail
  main "$@"
) 2>&1 | tee -a "$LOG_FILE"
status=${PIPESTATUS[0]}
exit "$status"
```

Make it executable:

```bash
chmod 700 run_nile_smoke_test.sh  # Restrict script access to the current user.
```

## 5. Configure Java Environment

Create and activate the Conda JDK environment:

```bash
cd /data/pro00110219/zq63/ICU_SOFA/nile_smoke_test  # Use your own WORK_DIR.
bash setup_conda_java.sh                            # Install OpenJDK and javac.
conda activate nile-java                            # Activate before running NILE.
```

Verify Java:

```bash
which java       # Should point to the Conda environment.
java -version    # OpenJDK 21 is expected by default.
which javac      # Must exist.
javac -version   # Confirms the compiler is available.
```

If OpenJDK 21 is unavailable:

```bash
ENV_NAME=nile-java17 OPENJDK_VERSION=17 bash setup_conda_java.sh
conda activate nile-java17
```

## 6. Run the Full Pipeline

Place your input CSV at `$INPUT_CSV`, then run:

```bash
cd /data/pro00110219/zq63/ICU_SOFA/nile_smoke_test  # Use your own WORK_DIR.
conda activate nile-java                            # Required for java and javac.
INPUT_CSV=$WORK_DIR/input/nile_input.csv bash run_nile_smoke_test.sh
```

Run a smaller test:

```bash
NROWS=50 INPUT_CSV=$WORK_DIR/input/nile_input.csv bash run_nile_smoke_test.sh
```

Run all rows:

```bash
NROWS=0 INPUT_CSV=$WORK_DIR/input/nile_input.csv bash run_nile_smoke_test.sh
```

Use different source column names:

```bash
PATIENT_COL="Durable Patient Key" \
ENCOUNTER_COL="Encounter Key" \
START_DATE_COL="Report Date" \
NOTES_COL="Report Text" \
INPUT_CSV=$WORK_DIR/input/raw_notes.csv \
bash run_nile_smoke_test.sh
```

## 7. Expected Outputs

```text
$WORK_DIR/input/nile_input_prepared_<N>_<timestamp>.csv
$WORK_DIR/classes/NILE/NILECSVProcessor.class
$WORK_DIR/output/nile_output_<N>_<timestamp>.csv
$WORK_DIR/logs/run_<timestamp>.log
$WORK_DIR/logs/manifest_<timestamp>.txt
```

The NILE output is pipe-delimited:

```text
patient_num|encounter_num|start_date|code_certainty
```

`code_certainty` uses suffixes:

```text
Y = present
N = negated
P = possible or uncertain
```

## 8. Basic QC

```bash
wc -l output/nile_output_*.csv
tail -50 logs/run_*.log
cat logs/manifest_*.txt
grep -c "Skipping malformed dictionary entry" logs/run_*.log
```

Dictionary warnings can occur during NILE initialization. They do not
necessarily mean the run failed. Confirm success by checking output rows and
non-empty `code_certainty` values.

## 9. Common Errors

Missing compiler:

```text
java and javac must both be available
```

Fix:

```bash
conda activate nile-java
```

Missing dictionary:

```text
NILE dictionary not found
```

Fix:

```bash
ls -lh $NILE_ROOT/NILE/NER_dictionary.txt
```

Input column mismatch:

```text
Required input column not found
```

Fix: rename the CSV columns or set `PATIENT_COL`, `ENCOUNTER_COL`,
`START_DATE_COL`, and `NOTES_COL`.
