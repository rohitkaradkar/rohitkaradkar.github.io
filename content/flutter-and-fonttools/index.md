---
draft: false
date: 2023-12-24
title: Flutter, Fonttools, and Helvetica Neue
slug: flutter-and-fonttools
summary: How integrating a simple font in Flutter, led to the discovery of interesting things
tags: [scripts, flutter, fonts, typography]
categories: [flutter]
---

## Finding the correct font asset file for Helvetica Neue
I was working on a Flutter project where I had to use the Helvetica Neue font, but as it's a proprietary font, an asset for it wasn't available freely on the internet. 

Then I remembered that MacOS has a "Font Book" app that lets you browse, manage, and import/export Fonts on your Mac.

{{< img src="./images/font-book-export.png"
        align="center"
        alt="Exporting font from Font Book app"
        caption="Exporting font from Font Book app"
>}}

## Splitting the TrueType Collection (TTC) font file by its style
The `.ttc` font file contained all 14 styles of Helvetica Neue, which I didn't need and would contribute to the app size. 
I decided to split the font file by its style.

```python
#!/usr/bin/env python3

from fontTools.ttLib.ttCollection import TTCollection
from fontTools.ttLib import TTFont
import os
import sys

filename = sys.argv[1]
ttc = TTCollection(filename)
print(f"Splitting {filename} ...")

# filename without extension
basename = os.path.basename(filename).split('.')[0]

font: TTFont
for i, font in enumerate(ttc):
    stylename = font['name'].getDebugName(2).replace(' ', '')
    font_file_name = f"{basename}-{stylename}.ttf"
    font.save(font_file_name)
    print(f"Saved -> {font_file_name}, Weight: {font['OS/2'].usWeightClass}")
```
```bash
# Install fontTools python library
pip3 install fontTools

# Pass the font file as an argument
python3 splitfonts.py HelveticaNeue.ttc

# Output. sorted by weight for easy reference
Splitting HelveticaNeue.ttc ...
Saved -> HelveticaNeue-UltraLight.ttf, Weight: 100
Saved -> HelveticaNeue-UltraLightItalic.ttf, Weight: 100
Saved -> HelveticaNeue-Thin.ttf, Weight: 200
Saved -> HelveticaNeue-ThinItalic.ttf, Weight: 200
Saved -> HelveticaNeue-Light.ttf, Weight: 300
Saved -> HelveticaNeue-LightItalic.ttf, Weight: 300
Saved -> HelveticaNeue-Regular.ttf, Weight: 400
Saved -> HelveticaNeue-Italic.ttf, Weight: 400
Saved -> HelveticaNeue-Medium.ttf, Weight: 500
Saved -> HelveticaNeue-MediumItalic.ttf, Weight: 500
Saved -> HelveticaNeue-Bold.ttf, Weight: 700
Saved -> HelveticaNeue-BoldItalic.ttf, Weight: 700
Saved -> HelveticaNeue-CondensedBold.ttf, Weight: 700
Saved -> HelveticaNeue-CondensedBlack.ttf, Weight: 900
```

Now, I could use the `Thin`, `Regular`, `Bold`, and `CondensedBold` files.
# Using custom font in Flutter
```dart
// main.dart
@override
Widget build(BuildContext context) {
  const sampleText = 'The quick brown fox jumps over the lazy dog';
  const helveticaNeue = TextStyle(
    fontFamily: 'HelveticaNeue',
    fontSize: 16,
  );
  return Scaffold(
    body: Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(
          sampleText,
          style: helveticaNeue.copyWith(fontWeight: FontWeight.w200),
          textAlign: TextAlign.center,
        ),
        const SizedBox(height: 8),
        Text(
          sampleText,
          style: helveticaNeue.copyWith(fontWeight: FontWeight.w400),
          textAlign: TextAlign.center,
        ),
        const SizedBox(height: 8),
        Text(
          sampleText,
          style: helveticaNeue.copyWith(fontWeight: FontWeight.w700),
          textAlign: TextAlign.center,
        ),
        const SizedBox(height: 8),
        Text(
          sampleText,
          style: helveticaNeue.copyWith(fontWeight: FontWeight.w800),
          textAlign: TextAlign.center,
        ),
      ],
    ),
  );
}
```

```yaml
# pubspec.yaml
flutter:
  fonts:
    - family: HelveticaNeue
      fonts:
        - asset: fonts/HelveticaNeue-Thin.ttf
          weight: 200
        - asset: fonts/HelveticaNeue-Regular.ttf
          weight: 400
        - asset: fonts/HelveticaNeue-Bold.ttf
          weight: 700
        - asset: fonts/HelveticaNeue-CondensedBold.ttf
          weight: 800
```
{{<img src="./images/false-weights.png"
      align="center"
      alt="Bold and CondensedBold font looks the same"
      caption="Bold and CondensedBold font looks the same"
      width="360"
  >}}

## Realising font weight doesn't work in Flutter
Even though I have applied and used separate weights for the `Bold` and `CondensedBold` fonts, they look the same. After pulling some hairs and digging through Flutter GitHub issues, I found out that, for given `fontFamily` and `fontWeight`, the Flutter engine selects the font asset based on its font metadata, ignoring the `weight` and `style` properties declared in `pubspec.yaml`. Refer Issue [#3591](https://github.com/flutter/website/issues/3591#issuecomment-521806077).

A good approach here is to always check the font metadata for overlapping weights and only declare multiple assets under the same `fontFamily` if their weights align with [FontWeight](https://api.flutter.dev/flutter/dart-ui/FontWeight-class.html) class guidelines. For my use case, we can clearly see that the `Bold` and `CondensedBold` fonts have the same weight.

Using fonttools library, I changed the weight of `CondensedBold` font to `800` but it still wasn't enough. Flutter engine still couldn't differentiate between `Bold` and `CondensedBold` fonts and always used the former. So, declaring the different font family for the `CondensedBold` font seems to be the best choice.

```python
# fontmetadata.py
#!/usr/bin/env python3

import os
import sys
from fontTools import ttLib

font_file_path = sys.argv[1]
font = ttLib.TTFont(font_file_path)
print(f"{font_file_path} -> Name: {font['name'].getDebugName(1)}, {font['name'].getDebugName(2)}, {font['OS/2'].usWeightClass}")
```
```bash
# Install fontTools python library
pip3 install fontTools

# Check font metadata for all the ttf fonts in current directory
ls | grep '\.ttf' | xargs -I '{}' python3 fontmetadata.py '{}'

# Output
HelveticaNeue-Bold.ttf -> Name: Helvetica Neue, Bold, 700
HelveticaNeue-CondensedBold.ttf -> Name: Helvetica Neue, Condensed Bold, 700
HelveticaNeue-Regular.ttf -> Name: Helvetica Neue, Regular, 400
HelveticaNeue-Thin.ttf -> Name: Helvetica Neue, Thin, 200
```
```yaml
# pubspec.yaml
flutter:
  uses-material-design: true

  fonts:
    - family: HelveticaNeue
      fonts:
        - asset: fonts/HelveticaNeue-Thin.ttf
        - asset: fonts/HelveticaNeue-Regular.ttf
        - asset: fonts/HelveticaNeue-Bold.ttf

    - family: HelveticaNeue-CondensedBold
      fonts:
        - asset: fonts/HelveticaNeue-CondensedBold.ttf
```
{{<img src="./images/final.png"
      align="center"
      alt="When CondensedBold is used as a separate font family"
      caption="When CondensedBold is used as a separate font family"
      width="360">}}

# TL;DR
- Default available MacOS fonts can be exported from the Font Book app.
- You can split the TrueType Collection (TTC) font file by its style using the `fontTools` python library.
- While using custom fonts in Flutter, always check the font metadata for overlapping weights
- Flutter engine relies on font metadata, ignoring the `weight` and `style` properties declared in `pubspec.yaml`.
- Declaring separate font family for overlapping font weights is a safe workaround as of Flutter `3.16.8`.