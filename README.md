### AI Image Enhancer Mobile

Aplikacja na Androida do lokalnej obróbki zdjęć, w 100% offline: upscaling, wyostrzanie,
odszumianie, usuwanie znaku wodnego, poprawa twarzy i usuwanie tła. Wnioskowanie leci
na ONNX Runtime, modele są w APK, nic nie wychodzi do sieci.

### Pobranie

Gotowy plik `.apk` znajdziesz w zakładce **Releases** tego repozytorium.
Wymagania: Android 8.0 (API 26) lub nowszy, arm64-v8a.

### Modele w paczce

| funkcja | model | rozmiar | wejście |
|---|---|---|---|
| usuwanie tła | IS-Net general-use | 176 MB | 1024×1024 |
| upscaling, poprawa twarzy | Real-ESRGAN SRVGGNetCompact x4 | 4,9 MB | dowolne |

Znak wodny i filtry (wyostrzanie, odszumianie) liczy OpenCV, bez sieci neuronowej.

### Uwaga techniczna: dlaczego usuwanie tła nie jedzie po NNAPI

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

Obecna wersja liczy usuwanie tła modelem IS-Net na CPU. Pomiar na maszynie x86, 6 wątków,
wejście 1024×1024: 0,48 s wobec 7,8 s dla BiRefNetu, około 270 MB pamięci wobec około 6 GB.
Jakość względem BiRefNetu na zdjęciach portretowych: MAE 0,0056 do 0,0118 i IoU 0,987 do 0,990.

### Licencje modeli

IS-Net general-use: eksport ONNX na licencji MIT, wagi na Apache-2.0.
Real-ESRGAN: BSD-3-Clause.
