# media-fix

`media-fix` is a Perl-based utility designed to repair, synchronize, and standardize metadata (EXIF/XMP) and filenames for photo and video collections. It handles complex timezone offsets and restores missing metadata using filename patterns.

## Usage

```bash
media-fix -tag|-touch|-rename|-xmp [-run] [-strict] [-skiptag] [-tz='str'] [-verbose] [FILE]
```

### Options

* **-tag** : Rewrite/fix incomplete EXIF date tags, usually from the filename.
* **-skiptag** : Ignore existing EXIF dates to force new dates from the filename.
* **-touch** : Change file modification date according to the target date.
* **-rename** : Change filename according to the target date.
* **-strict** : Enforce strict renaming format `YYYYMMDD_HHMMSS`.
* **-xmp** : Write an XMP sidecar with the correct `FileModifyDate` and timezone.
* **-run** : **Execution mode.** By default, the script runs in **Dry-Run** mode (no changes applied).
* **-tz='str'** : Override default timezone (e.g., `-tz='Europe/Lisbon'`).
* **-verbose** : Write detailed logs to the console and `process.log`.
* **FILE** : Optional. If not specified, processes the current directory tree recursively.

---

## Timezone Management

In the EXIF standard, date tags are paired with offsets for technical accuracy:

| Date Tag | SubSec Tag | Offset Tag |
| :--- | :--- | :--- |
| DateTimeOriginal | SubSecTimeOriginal | OffsetTimeOriginal |
| CreateDate | SubSecTimeDigitized | OffsetTimeDigitized |
| ModifyDate | SubSecTime | OffsetTime |

If offsets are missing, `media-fix` defaults to the system local time. You can override this behavior:

1. **Command Line**: Specify a `-tz` parameter with a valid TimeZone (highest priority).
2. **Folder Configuration**: Create an empty file starting with `.tz` (e.g., `.tz_europe_lisbon`) inside a directory.

**Note on Videos:** The tool automatically converts UTC-stored video metadata (QuickTime/MP4 standard) to your target or overridden timezone.

---

## Advanced Controls

### Ignoring Directories
To skip a directory during recursive processing, place an empty file named `.noprocess` in that folder.

### Filename Parsing
If EXIF data is missing, the tool extracts dates from usual filename:
* Standard: `YYYYMMDD_HHMMSS`
* WhatsApp: `YYYYMMDD-WA...`
* Folders: Can infer dates from folder names (e.g., `20230512_Trip`).

Anything found after a matching date will be kept as `remainder` and used for the final name.
Some cleanup is applied to the `remainder`, like removing trailing `~1, ..., _01, ..., _HDR`

If you want to rename only with the `YYYYMMDD_HHMMSS` pattern, use the `-strict` parameter.

**Note for WhatsApp:** The name/date is completely wrong, not corresponding to the actual date of the asset, but when the file was shared with you:
As we have no other choice we still take it as reference.

The hour/minuste/seconds will be computed from the trailing digits after `WA` as a number of seconds added to `12:00:00`, the goal being to have distinct filenames.
```
WA00001 -> 12:00:01
WA03670 -> 13:01:10
```
```bash
> media-fix -tag -skiptag IMG-20250714-WA1234.jpg
'./IMG-20250714-WA1234.jpg' tagged to '2025:07:14 12:20:34.000+02:00'
```

### Forcing Incorrect Metadata
By combining filename recognition with the `-skiptag` parameter, you can force or reset existing but incorrect EXIF tags.

Simply rename your image/video file first with the desired date pattern you want and then run the command with both `-tag` and `-skiptag` to overwrite the internal metadata.

### XMP Sidecar
This was initially designed for **Immich** to generate an external XMP sidecar file having the correct `DateTimeOriginal`.

With this method you can fix your library without touching your assets. Note however that Immich is currently (v2.6.1) not handling XMP files efficiently.
```bash
> media-fix -xmp -run 20251021_214337.jpg
'./20251021_214337.jpg' XMP file written

> cat 20251021_214337.jpg.xmp 
<?xpacket begin='﻿' id='...'?>
<x:xmpmeta xmlns:x='adobe:ns:meta/' x:xmptk='Image::ExifTool 13.50'>
<rdf:RDF xmlns:rdf='http://www.w3.org/1999/02/22-rdf-syntax-ns#'>

 <rdf:Description rdf:about=''
  xmlns:exif='http://ns.adobe.com/exif/1.0/'>
  <exif:DateTimeOriginal>2025-10-21T21:43:37.000+02:00</exif:DateTimeOriginal>
 </rdf:Description>
</rdf:RDF>
</x:xmpmeta>
<?xpacket end='w'?>
```

You will need to generate all your XMP, then in Immich navigate to _Administration > Job Queues > Sidecar Metadata > Discover_.

This will make your XMP files visible for Immich and will trigger the _Extract Metadata_ (unfortunately on *ALL* you assets.

**Note:** If an existing XMP is found, it will *not* be rewritten/updated. Also if the file has correct Exif tags, the XMP is not written as not required.


### Restoring broken assets library
This tool can also be used to restore the correct filenames froma damaged filesystem.

After recovering the files with tolls like `testdisk/photorec`, you can rename images/videos from their EXIF tags using the `-rename` tag:
```
> media-fix -tag -touch -rename has-good-exif 
'./has-good-exif' touched to '2023:09:14 17:20:02+02:00' renamed to '20230914_172002 has-good-exif.jpg'
```

---

## Examples

**Dry-Run (preview only):**
```bash
media-fix -tag -touch -rename
```

**Full Execution:**
```bash
media-fix -tag -touch -rename -run
```

**Full Execution with logging:**
```bash
> media-fix -tag -verbose 20251021_214337.jpg
   TimeZone settings: system 'default'='Europe/Paris'
Processing image './20251021_214337.jpg' 
   EXIF Tag read: extratag is missing, using default 'SubSecTimeOriginal'='0'
   EXIF Tag read: extratag is missing, using default 'OffsetTimeOriginal'='Europe/Paris'
   EXIF Tag read: TZ Date 'DateTimeOriginal'='2025:10:21 21:43:37'
   EXIF Tag write: TZ Date 'DateTimeOriginal, SubSecTimeOriginal, OffsetTimeOriginal'='2025:10:21 21:43:37, 0, +02:00'
   EXIF Tag write: TZ Date 'CreateDate, SubSecTimeDigitized, OffsetTimeDigitized'='2025:10:21 21:43:37, 0, +02:00'
   EXIF Tag write: TZ Date 'ModifyDate, SubSecTime, OffsetTime'='2025:10:21 21:43:37, 0, +02:00'
   Finalize apply: tagged to '2025:10:21 21:43:37.000+02:00'
```

---

## Requirements
* Perl
* `Image::ExifTool`
* `File::Find::Rule`
