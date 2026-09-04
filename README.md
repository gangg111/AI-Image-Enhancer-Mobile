### AI Image Enhancer Mobile

Aplikacja na Androida do lokalnej obróbki zdjęć, w 100% offline: upscaling, wyostrzanie,
odszumianie, usuwanie znaku wodnego, poprawa twarzy i usuwanie tła. Wnioskowanie leci
na ONNX Runtime w wariancie QNN, czyli przez Qualcomm AI Engine Direct, więc modele
liczą się na NPU Hexagon albo na GPU Adreno, a nie na CPU. Nic nie wychodzi do sieci.

### Pobranie

Gotowy plik `.apk` znajdziesz w zakładce **Releases** tego repozytorium.
Wymagania: Android 8.0 (API 26) lub nowszy, arm64-v8a, układ Qualcomm Snapdragon.

### Modele w paczce

| funkcja | model | rozmiar | wejście |
|---|---|---|---|
| usuwanie tła | IS-Net general-use | 176 MB | 1024×1024 |
| upscaling, poprawa twarzy | Real-ESRGAN SRVGGNetCompact x4 | 4,9 MB | 512×512 (kafelki) |

Znak wodny i filtry (wyostrzanie, odszumianie) liczy OpenCV, bez sieci neuronowej.

### Jak wybierany jest akcelerator

Sesja ONNX tworzona jest kaskadą: najpierw Hexagon NPU (`backend_type=htp`), potem
Adreno (`backend_type=gpu`), a CPU dopiero gdy oba odmówią przyjęcia grafu. Dwa pierwsze
kroki mają ustawione `session.disable_cpu_ep_fallback=1`, więc jeśli QNN nie weźmie
całego grafu, tworzenie sesji rzuca wyjątkiem zamiast po cichu policzyć na CPU. Udane
utworzenie sesji jest więc dowodem, że graf poszedł na akcelerator. Model fp32 wchodzi
na NPU bez kwantyzacji, bo ORT ustawia grafowi precyzję FP16.

Backend, który faktycznie przyjął graf, apka pokazuje w nagłówku po pierwszej inferencji
i wypisuje do logu:

```
adb logcat -s AIENHANCER
```

### Uwaga techniczna: dlaczego nie BiRefNet i nie NNAPI

Wcześniejsza wersja używała BiRefNet massive (972 MB) z akceleracją NNAPI i wywalała się
przy tworzeniu sesji komunikatem:

```
OrtException: Error code - ORT_FAIL - message: model_builder.cc:523
AddOperation ResultCode: ANEURALNETWORKS_BAD_DATA, op = 8
```

Kod `op = 8` to `ANEURALNETWORKS_FLOOR` (NDK, `NeuralNetworksTypes.h`), a NNAPI przyjmuje
dla niego tensory o randze najwyżej 4. BiRefNet ma 20 węzłów `Floor` o randze 6, w module
konwolucji deformowalnej dekodera (`dec_att/aspp*/atrous_conv`), więc NNAPI odrzuca cały
model, zanim policzy pierwszy piksel. Podmiana wag na inny checkpoint BiRefNet niczego nie
zmienia, bo topologia grafu jest identyczna.

IS-Net ma maksymalną rangę 4 i po optymalizacji ORT redukuje się do 355 węzłów w siedmiu
typach operacji, które QNN obsługuje w komplecie. Jakość względem BiRefNetu na zdjęciach
portretowych: MAE od 0,0056 do 0,0118 i IoU od 0,987 do 0,990.

### Licencje

IS-Net general-use: eksport ONNX na licencji MIT, wagi na Apache-2.0.
Real-ESRGAN: BSD-3-Clause.
Biblioteki Qualcomm AI Engine Direct są dystrybuowane wyłącznie jako część APK,
zgodnie z warunkami AI Stack License.
