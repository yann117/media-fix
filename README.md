# media-fix

`media-fix` is a Perl-based utility (v1.0.4) designed to repair, synchronize, and standardize metadata (EXIF/XMP) and filenames for photo and video collections. It handles complex timezone offsets and restores missing metadata using filename patterns.

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
Some cleanup is applied to the `remaineder`, like removing trailing `~1, ..., _01, ..., _HDR`

If you want to rename only with the `YYYYMMDD_HHMMSS` pattern, use the `-strict` parameter.

**Note for WhatsApp:** The name/date is completely wrong, not corresponding to the actual date of the asset, but when the file was shared with you.

As we have no other choice we still take it as reference.

The hour/minuste/seconds will be computed from the trailing digits after `WA` as a number of seconds added to `12:00:00`, the goal being to have distinct filenames.
```
WA00001 -> 12:00:01
WA03670 -> 13:01:10
```


### Forcing Incorrect Metadata
By combining filename recognition with the `-skiptag` parameter, you can force or reset existing but incorrect EXIF tags. Simply rename your image/video file with the desired date pattern and run the command with both `-tag` and `-skiptag` to overwrite the internal metadata.

### XMP Sidecar
This was initially designed for **Immich** to generate an external XMP sidecar file having the correct `DateTimeOriginal`.

With this method you can fix your library without touching your assets. Note however that Immich is currently (v2.6.1) not handling XMP files efficiently.
```bash
media-fix -xmp
```

You will need to generate all your XMP, then in Immich navigate to _Administration > Job Queues > Sidecar Metadata > Discover_.

This will make your XMP files visible for Immich and will trigger the _Extract Metadata_ (unfortunately on *ALL* you assets.

**Note:** If an existing XMP is found, it will *not* be rewritten/updated.

---

## Examples

**Dry-Run (Preview only):**
```bash
media-fix -tag -rename -touch
```

**Full Execution with Logging:**
```bash
media-fix -tag -rename -touch -run -verbose
```

**Force Update from Filenames:**
```bash
media-fix -tag -skiptag -run
```

---

## Requirements
* Perl
* `Image::ExifTool`
* `File::Find::Rule`
