# Plankinspektion - Systembeskrivning och Manual

## Översikt

Detta system är en komplett ML-pipeline för automatisk kvalitetskontroll av plankändar i sågverksproduktion. Systemet detekterar och klassificerar defekter (sprickor och hörnskador) och fattar automatiserade beslut om planka ska godkännas eller kastas.
![alt text](assets/flode.png)

---

## Modulbeskrivning

### 1. Plankänds-detektion  
*(`YOLO_Detect_Cropping/` + `annotation_project_plankends/`)*

**Syfte:** Träna en YOLO-detektionsmodell som hittar plankändar i kamerabilder och beskär dessa för vidare analys.

**Komponenter:**

| Fil | Beskrivning |
|-----|-------------|
| `simple_annotator.py` | GUI-verktyg för att rita bounding boxes runt plankändar |
| `split_dataset.py` | Dela upp annoterad data i train/val/test (80/15/5%) |
| `train_custom_yolo.py` | Träna YOLOv8-detektionsmodell |
| `crop_detected_objects.py` | Använd tränad modell för att beskära plankändar |

**Annotationsverktyg (simple_annotator.py):**  
skapar annoterade bilder för träning. 

- Klicka och dra för att rita box
- `S` / `N` - Spara och nästa bild
- `P` - Föregående bild
- `D` - Ta bort senaste box
- `Q` / `ESC` - Avsluta

**Skapa dataset (split_dataset.py):**
Delar Training och Validation bilder och skriver YOLO "data.yaml"

**Träna YOLO på dataset (train_custom_yolo.py)**
Tränar en modell för detektion av plankändar
Tränad modell sparas i `runs/train/plank_detector/weights/best.pt`

**Croppa (klipp ut) rådata-bilder (crop_detected_objects.py)**
Skapar rådata för STEG 2 - Defekt detektering

---

### 2. Defekt-segmentering (`YOLO_Segment_Skador/` + `annotation_skador/`)

**Syfte:** Träna en YOLO-segmenteringsmodell som identifierar och maskerar defekter på de beskurna plankändarna.

**Defektklasser:**

| Klass ID | Namn | Beskrivning |
|----------|------|-------------|
| 0 | `corner_damage` | hörnskador/kantskada |
| 1 | `crack` | Spricka |

**Komponenter:**

| Fil | Beskrivning |
|-----|-------------|
| `multi_mode_annotator.py` | GUI för annotering av både boxes (hörn) och polylines (sprickor) |
| `prepare_dataset.py` | Konvertera annotationer till YOLO-Seg format |
| `train_yolo_segm.py` | Träna YOLOv8-Seg modell |

**Annotationsverktyg (multi_mode_annotator.py):**

- `M` - Växla läge (BOX / LINE)
- **Box-läge:** Klicka och dra rektangel (för hörnskador)
- **Line-läge:** Klicka för att lägga noder, högerklick avsluta (för sprickor)
- `+` / `-` - Justera linjetjocklek
- `BACKSPACE` - Ta bort senaste nod
- `D` - Ta bort senaste annotering
- `ENTER` - Slutför polyline
- `S` / `N` - Spara och nästa
- `R` - Växla plankdetektering (auto/helbild)

**Visuell feedback:** 
Sprickor färgkodas efter antal kanter de når:

- 🟢 Grön: Når ingen kant (OK)
- 🟡 Gul: Når 1 kant (OK)
- 🔴 Röd: Når 2+ kanter (KASTA)

**Output:** Tränad modell sparas i `runs/segment/defect_detector/weights/best.pt`

---

### 3. Parameterinjustering (`Streamlit_PoC_ Intrimning/`)

**Syfte:** Webbgränssnitt för att finjustera detektionsparametrar och verifiera att systemet fungerar korrekt innan produktionsdrift.

**Kör applikationen:**

```
cd Streamlit_PoC_Intrimning
streamlit run app.py

```

**Konfigurerbara parametrar:**

| Kategori | Parameter | Standard | Beskrivning |
|----------|-----------|----------|-------------|
| **Detektering** | Defektkonfidens | 0.30 | Minsta konfidens för defektdetektering |
| | Beskärningskonfidens | 0.35 | Minsta konfidens för plankdetektering |
| | Min Aspect Ratio | 3.0 | Bredd/höjd-kvot för plankfiltrering |
| **Geometri** | Beskärningsmarginal | 15 px | Marginal runt detekterad plank |
| | Beröringstolerans | 5 px | Avstånd för att räknas som "når kant" |
| | Kantavvisningskvot | 0.65 | Kasta om katet > kvot × tjocklek |
| | Roterade gränser | På | Hantera lutande plankor |
| **Filter** | Rödfiltertröskel | 127 | Filtrera röda transportband |

**Avvisningskriterier:**

1. **Sprickor:** Kastas om sprickan når 2 eller fler kanter
2. **hörnskador:** Kastas om största katet > 65% av planktjockleken

---

### 4. Produktionssystem (`Plank_Inspector_Operator/`)

**Syfte:** Operatörsgränssnitt för realtidsinspektion med kameraström, statistik och batch-hantering.

**Två varianter finns:**
- `flask_operator/` - Webb-baserat (Flask + WebSocket)
- `swedish_version/` - Streamlit-baserat

**Starta Flask-operatören:**
```bash
cd Plank_Inspector_Operator/flask_operator
pip install -r requirements.txt
python app.py
```
Öppna sedan `http://localhost:5000` i webbläsare.

**Funktioner:**

| Funktion | Beskrivning |
|----------|-------------|
| **Realtidsström** | ~10 fps bildström via WebSocket |
| **Batch-inställningar** | Produktspecifika parametrar |
| **Bypass-funktioner** | Tillfälligt inaktivera kontroller |
| **Statistik** | Total, OK, Kasserade, Kassationsfrekvens |
| **Historik** | Senaste 100 inspektioner |
| **Larm** | Varning/Alarm vid hög kassationsfrekvens |

**Batch-inställningar:**
```json
{
  "batch_id": "75x100_furu",
  "corner_check_enabled": true,
  "corner_reject_ratio": 0.65,
  "edge_to_edge_enabled": true,
  "touch_tolerance": 5,
  "defect_confidence": 0.30,
  "reject_rate_warning": 5.0,
  "reject_rate_alarm": 10.0
}
```

---

## Pipeline-arkitektur (`plank_inspector/`)

Kärnbiblioteket som används av både injusteringsverktyget och produktionssystemet.

```
plank_inspector/
├── __init__.py          # Export av PlankInspector, DefectStatus
├── defect_detector.py   # DefectDetector (YOLO-Seg) + PlankCropper (YOLO-Detect)
├── geometry.py          # GeometricVerifier - avvisningslogik
├── pipeline.py          # PlankInspector - komplett pipeline
└── radar.py             # Radarbaserad kantdetektering (experimentell)
```

**Klassdiagram:**
```
PlankInspector
├── DefectDetector      (defekt-segmentering)
├── PlankCropper        (plank-detektering + beskärning)
└── GeometricVerifier   (geometrisk verifiering)
    ├── check_crack()   → CrackResult
    └── check_corner()  → CornerResult
```

**Användning i Python:**
```python
from plank_inspector import PlankInspector, DefectStatus

# Initiera
inspector = PlankInspector(
    defect_model="runs/segment/defect_detector6/weights/best.pt",
    cropper_model="runs/train/plank_detector5/weights/best.pt",
    device='cuda'
)

# Inspektera en bild
result = inspector.inspect_single_plank(image)
if result:
    inspection, crop = result
    if inspection.is_reject:
        print(f"KASTA: {inspection.status.value}")
```

---

## Installation

**Krav:**
- Python 3.8+
- CUDA-kompatibelt GPU (rekommenderas)

**Python environment:**
```bash
python -m venv venv
.\venv\Scripts\activate.bat
pip install -r .\requirements.txt
pip install ultralytics opencv-python numpy streamlit flask flask-socketio
```

**Verifiera GPU:**
```python
import torch
print(f"CUDA tillgänglig: {torch.cuda.is_available()}")
print(f"GPU: {torch.cuda.get_device_name(0)}")
```

---

## Arbetsflöde: Träna nytt system

### Steg 1: Annotera plankändar
```bash
cd Raw_data/annotation_project_plankends
python simple_annotator.py --images images --labels labels
python split_dataset.py --images images --labels labels --output ./
```

### Steg 2: Träna plankdetektor
```bash
cd YOLO_Detect_Cropping
python train_custom_yolo.py
```

### Steg 3: Beskär plankändar
```bash
python crop_detected_objects.py
```

### Steg 4: Annotera defekter
```bash
cd Raw_data/annotation_skador
python multi_mode_annotator.py --images images --labels labels
python prepare_dataset.py --images images --labels labels --output dataset
```

### Steg 5: Träna defektdetektor
```bash
cd YOLO_Segment_Skador
python train_yolo_segm.py
```

### Steg 6: Injustera parametrar
```bash
cd Streamlit_PoC_Intrimning
streamlit run app.py
```

### Steg 7: Produktionsdrift
```bash
cd Plank_Inspector_Operator/flask_operator
python app.py
```

---

## Filstruktur

```
Resultat/
├── Raw_data/
│   ├── annotation_project_plankends/   # Plankändsannotering
│   │   ├── images/                     # Råbilder
│   │   ├── labels/                     # YOLO-format labels
│   │   ├── simple_annotator.py
│   │   ├── split_dataset.py
│   │   └── data.yaml
│   │
│   └── annotation_skador/              # Defektannotering
│       ├── images/                     # Beskurna plankändar
│       ├── labels/                     # YOLO-Seg labels
│       ├── multi_mode_annotator.py
│       └── prepare_dataset.py
│
├── YOLO_Detect_Cropping/               # Plankdetektering
│   ├── train_custom_yolo.py
│   ├── crop_detected_objects.py
│   └── yolov8n.pt
│
├── YOLO_Segment_Skador/                # Defektsegmentering
│   ├── train_yolo_segm.py
│   └── yolov8n-seg.pt
│
├── Streamlit_PoC_Intrimning/           # Parameterinjustering
│   ├── app.py
│   └── plank_inspector/
│
├── Plank_Inspector_Operator/           # Produktionssystem
│   ├── flask_operator/
│   └── swedish_version/
│
└── runs/                               # Tränade modeller
    ├── train/plank_detector*/weights/
    └── segment/defect_detector*/weights/
```

---

## Felsökning

| Problem | Lösning |
|---------|---------|
| "CUDA out of memory" | Minska `batch` i träningsskript eller `imgsz` |
| Ingen plank detekterad | Sänk `cropper_confidence` eller kontrollera `min_aspect_ratio` |
| För många falska positiver | Höj `defect_confidence` |
| Sprickor missar kanter | Öka `touch_tolerance` |
| hörnskador godkänns felaktigt | Sänk `corner_reject_ratio` |

---

## Kontakt och support

Systemet utvecklat av Erik Erlandsson - Videonet AB med hjälp av AI-assisterad programmering (Claude + Cursor).

**Versionhistorik:**

- 2025-12-01: Initial release
- 2025-12-10: Test och dokumentation


