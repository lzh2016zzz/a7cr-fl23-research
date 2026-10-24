# a7cr-fl23-research

**English** · [简体中文入口](README.zh-CN.md)

Research notes on porting the FL2/FL3 still-photo Creative Looks to Sony ILCE-7CR Ver.2.01, using parameters from the Alpha 1 II (ILCE-1M2) Ver.4.00. Updated: 2026-10-16. This document covers the implementation approach, key offsets and values, operation sequence, and validation and troubleshooting.

## Disclaimer

**Firmware modification can brick your camera. Proceed at your own risk.** These notes describe limited testing on the author's own device. Successful tests do not establish safety for other bodies, firmware versions, combinations of settings, or future use. Outstanding validation is identified below. This is unofficial research, provided as is, without guarantees of compatibility, stability, or recoverability after a failure.

Writing modified firmware may prevent the camera from starting, cause loss of data or settings, introduce malfunctions, affect warranty coverage, and incur repair costs. The official memory-card recovery procedure is not guaranteed to repair every kind of damage. Users must verify the model and firmware version, make backups, and decide for themselves whether to proceed. **To the extent permitted by applicable law, the author and contributors accept no liability for equipment damage, data loss, repair costs, or other losses resulting from use of these notes or implementation of the modifications.**

This repository documents key writes, offsets, values, and steps. It does not distribute source code, firmware, write payloads, or private device logs. Unless explicitly stated otherwise, addresses below are **offsets within the corresponding file**, not runtime addresses. Short values are shown as raw hexadecimal bytes; multibyte integers are little-endian.

Current status: FL2/FL3 can be selected and used to take photos, and existing looks have passed basic shooting checks. Original photos taken with fixed white balance and user feedback show visible image differences. The name mappings, Fade defaults, and additional `0xb029 ColorMode` fix have verified on-disk writes. **The latest changes still require a restart, a reset of each look, and validation with newly captured photos.** Compatibility across all shooting combinations and color matching against a physical Alpha 1 II have not been established.

## Implementation approach

The A7CR AP already contains FL2/FL3 names, a twelve-look model, and eighteen parameter slots. However, the product flags, selection capabilities, saved bindings, and CP still-photo processing are not fully connected. The approach is to reuse the existing slots and native settings mechanisms, then complete the missing conversions, image processing, and metadata mappings. Finding the names alone only establishes an entry point.

The processing chain is:

> Show FL2/FL3 in the menu -> allow selection -> bind the selected slot to the new look -> convert the AP look to a CP mode -> select the CP curves and color parameters -> write the photo and its look identifiers.

Factory defaults determine which values a reset restores. Boot snapshots affect which version runs after a restart. Both require separate checks.

| Layer | Why it needs attention | Possible symptom if the chain is incomplete |
| --- | --- | --- |
| Menu and selection capabilities | Menu visibility and permission to select are controlled separately | FL2/FL3 appear but the camera reports that the function is unavailable |
| Slot bindings and identifier conversion | The reserved slots were bound to ST; each layer uses its own identifiers | The new name is selectable, but the actual request still selects ST |
| CP processing and color parameters | The new modes need parameter dispatch, gamma, chroma, and matrix support | The request changes, but processing falls back or only part of the effect is applied |
| Saved settings | Startup restores saved settings, overriding firmware initial bindings | The file is correct, but old values return after a restart |
| Photo metadata | Numeric modes, name strings, and playback ColorMode have separate writers | The image changes, but photo information or camera playback still shows ST |
| Factory defaults and reset | Current values differ from reset sources; base color processing is separate from the eight adjustment values | Manual adjustments work, but reset restores the old defaults |
| Loading the new components | An old WBI can restore the old process state | The on-disk file verifies correctly, but the new component is not running |

This implementation primarily uses persistent files and native Backup settings. Memory reads only help verify which settings or components are in use. The AP interface, CP image processing, and file identifiers each require validation; success in one layer does not prove the whole chain works.

### Identifier mapping

Each row below belongs to a different table or field. Identify the target layer before writing a value: AP look10 is not CP mode10.

| Location / meaning | FL2 | FL3 |
| --- | ---: | ---: |
| AP ordinary parameter slot index | 16 | 17 |
| AP look stored in the slot | 10 | 11 |
| AP ordinary UI internal enum | 17 | 18 |
| CP mode / Tag9416 CreativeStyle | 19 | 20 |
| CP still-photo color parameter index | 55 | 56 |
| CP still-photo gamma row | 48 | 49 |
| MakerNote `0xb029` ColorMode | 22 | 23 |

Identifiers in this table are decimal. Offsets and raw bytes later in this document are hexadecimal. Tag9416 `PictureProfile` also records 48/49, but it is a separate field from `CreativeStyle` and `0xb029 ColorMode`. Check the metadata group and tag together when inspecting a photo.

## 1. Firmware sources and extracted components

Target: **Sony ILCE-7CR Ver.2.01**. Donor: **Sony Alpha 1 II (ILCE-1M2) Ver.4.00**, referred to here as **A12**.

The following FL2/FL3 data and references were extracted from A12 firmware:

| Component | A12 source | Use in the A7CR port |
| --- | --- | --- |
| FL2/FL3 gamma curve records, including the ISO-column variants | `cp/cpapp-b.bin` | Supply still-photo gamma rows48/49; record layouts and target-specific fields are adapted to the A7CR |
| FL2/FL3 chroma records for three illuminants | CP default parameter unit01 | Merged into the A7CR color-parameter unit and selected through new parameter indices55/56 |
| FL2/FL3 color matrices for three illuminants | CP default parameter unit0d | Native `3 x 16 x 6` coefficient records per look, merged into the A7CR matrix unit |
| Eight factory adjustment defaults | `usr/lib/appFw.so` factory tables and reset sources | Establish `0,0,0,1,0,4,3,1` as the reference; the A7CR only needs the two Fade defaults changed from 0 to +1 |
| Look and metadata identifier mappings | Corresponding A12 CP tables | Reference for FL2/FL3 mode identities and ColorMode22/23 |

The A7CR already supplies the FL2/FL3 menu names, display resources, parameter slots, native saved-settings mechanisms, and WBI interfaces. Menu flags, selection capabilities, converters, CP dispatch and table limits, metadata writers, and saved bindings are adapted on the A7CR side. Donor functions and structure offsets are not transplanted wholesale.

This is a selective integration of A12 look data into A7CR files and parameters. The camera's existing calibration and unrelated records are retained. Extracted or modified Sony firmware is not distributed in this repository.

Offsets below apply to the stated firmware versions and stages. Each stage requires checking the expected original bytes. The CP sequence is original -> image-processing patch -> name fix -> additional ColorMode fix. The last two are incremental and must not overwrite one another or be applied against the wrong stage. Longer instruction sequences and curve blocks are not reproduced here, so **this document is not a complete patch that can be applied directly**. The short-value tables alone cannot reconstruct the complete candidate.

## 2. AP: menu, selection, mode conversion, and defaults

Target: `/usr/lib/appFw.so`, on the `/usr` ext2 filesystem backed by `/dev/nflasha15`, normally mounted read-only. This experiment preserved the original inode and wrote only fixed windows. Complete-file readback, fsync, and restoration of the read-only mount were verified.

| File offset | Length | Before -> after | Purpose |
| --- | ---: | --- | --- |
| `0x59892c8` | 2 | `0000` -> `0100` | FL2 menu product flag, column 8 |
| `0x5989380` | 2 | `0000` -> `0100` | FL3 menu product flag, column 8 |
| `0x4b4ab34` | 4 | `00000000` -> `0a000000` | Initial look for ordinary slot16: ST -> FL2 |
| `0x4b4ab3c` | 4 | `00000000` -> `0b000000` | Initial look for ordinary slot17: ST -> FL3 |
| `0x3528cec` | 112 | Extend the bounded conversion to 12 entries; output table below | AP look -> CP mode |
| `0x4b2f209` | 2 | `0000` -> `0101` | Two selection capabilities; enabling the menu alone leaves selection unavailable |
| `0x4b3ba94` | 4 | `00000000` -> `01000000` | FL2 factory Fade default, PID `0x1565` |
| `0x4b3ba9c` | 4 | `00000000` -> `01000000` | FL3 factory Fade default, PID `0x1566` |

The conversion window preserves special-value and invalid-value handling. For ordinary look values `0..11`, its outputs are: `00 03 02 01 10 0f 11 12 05 0d 13 14`.

Mapping: FL2 uses ordinary slot16, look10, UI enum17, and CP mode19. FL3 uses ordinary slot17, look11, UI enum18, and CP mode20.

Existing saved values override initial bindings, so the two ordinary slots also require a Backup migration:

| Item | Before -> after | Save operation |
| --- | --- | --- |
| AP BKID `0x02cf1287`, 1B | `00` -> `0a` | Read back both items, then issue one `SAVE3(02cf)` |
| AP BKID `0x02cf1289`, 1B | `00` -> `0b` | Same save operation |

These changes use native `WRITE2`. Both the saved values and the active model values 10/11 were read back after a restart. `0x02cf1286` is the playback rating filter; `1288/128a` are MR look values. They must not be written as adjustment parameters.

### Factory defaults versus current values

The Alpha 1 II factory table and reset sources give the same eight defaults for FL2 and FL3: `0,0,0,1,0,4,3,1`, in this order: Contrast, Highlights, Shadows, Fade, Saturation, Sharpness, Sharpness Range, and Clarity. ST differs only in Fade, which is 0. This is evidence from static firmware defaults, not readings from a physical Alpha 1 II menu. CP supplies each look's underlying image processing.

Only the two factory Fade defaults change from 0 to 1; the eight current adjustment values are not overwritten in bulk. After loading the new AP, use the camera's native reset separately for FL2 and FL3, then check that Fade and its default marker are +1. The reset core is at AP file offset `0x2616388`; ordinary UI17/18 map through `0x2d50d18` to default slots16/17.

## 3. CP: image-processing dispatch, gamma, and color parameters

Target: `/cp/cpapp-b.bin`. In this experiment, `/cp` is the vfat filesystem on `/dev/nflasha8`. Only this file is modified; NOR, boot0, and the boot-source selector are not written.

Still-photo modes19/20 map to parameters55/56 and gamma48/49 respectively. The existing BW parameter54 is preserved. The initial image-processing stage covers 29 windows totaling 4032B of write ranges, without changing the file length. Longer blocks are described without reproducing their complete contents.

| File offset | Length | Modification | Before -> after / description |
| --- | ---: | --- | --- |
| `0x138834` | 4 | `stl_mode19` | `06200870` -> `3720e7e7` |
| `0x13884e` | 4 | `stl_mode20` | `06200870` -> `3820dae7` |
| `0x1384c4` | 2 | `mov_mode19_fallback` | `a900` -> `4100` |
| `0x1384c6` | 2 | `mov_mode20_fallback` | `b200` -> `4100` |
| `0x138852` | 26 | `movie_gamma_normalization_in_reclaimed_case` | Updated in the image-processing stage; complete block omitted |
| `0x64950e` | 16 | `chroma_strict_parameter_whitelist` | Updated in the image-processing stage; complete block omitted |
| `0x64954a` | 2 | `chroma_new_parameters_use_map` | `04d1` -> `efd1` |
| `0x64986a` | 26 | `secondary_chroma_strict_parameter_whitelist` | Updated in the image-processing stage; complete block omitted |
| `0x6498cc` | 2 | `secondary_chroma_new_parameters_use_map` | `04d1` -> `ead1` |
| `0x6492b6` | 2 | `matrix_parameter_upper_bound` | `342b` -> `382b` |
| `0x323672` | 4 | `gamma_table_ref0_lo` | `47f6840b` -> `47f6a05b` |
| `0x32367c` | 4 | `gamma_table_ref0_hi` | `c0f2101b` -> `c0f2201b` |
| `0x323f9a` | 4 | `gamma_table_ref1_lo` | `47f68401` -> `47f6a051` |
| `0x323fa2` | 4 | `gamma_table_ref1_hi` | `c0f21011` -> `c0f22011` |
| `0x323fe8` | 4 | `gamma_table_ref2_lo` | `47f68401` -> `47f6a051` |
| `0x323ff0` | 4 | `gamma_table_ref2_hi` | `c0f21011` -> `c0f22011` |
| `0x3240ce` | 4 | `gamma_table_ref3_lo` | `47f68400` -> `47f6a050` |
| `0x3240d2` | 4 | `gamma_table_ref3_hi` | `c0f21010` -> `c0f22010` |
| `0x9cf294` | 4 | `gamma_table_ref4_lo` | `47f68401` -> `47f6a051` |
| `0x9cf29c` | 4 | `gamma_table_ref4_hi` | `c0f21011` -> `c0f22011` |
| `0x32338a` | 8 | `movie_gamma_guard_call_32338a` | `44f67471c0f2e001` -> `15f662fa00bf00bf` |
| `0x3234b0` | 8 | `movie_gamma_guard_call_3234b0` | `44f67471c0f2e001` -> `15f6cff900bf00bf` |
| `0x3233f2` | 4 | `mode_table_ref0_lo` | `44f67470` -> `48f21830` |
| `0x3233f6` | 4 | `mode_table_ref0_hi` | `c0f2e000` -> `c0f22010` |
| `0x323474` | 2 | `gamma_limit_323474` | `2f28` -> `3228` |
| `0x323490` | 2 | `gamma_limit_323490` | `2f28` -> `3228` |
| `0x9cf262` | 2 | `gamma_limit_9cf262` | `2f28` -> `3228` |
| `0x20140` | 4 | `initialized_data_length` | `a0141400` -> `c0231400` |
| `0x10ff4a0` | 3872 | `3872_byte_provisional_data_component` | Updated in the image-processing stage; complete block omitted |

Color parameters are updated separately through native Backup operations. Modifying CP code alone is insufficient:

| CP chip1 unit | Length | Extracted A12 content used |
| --- | ---: | --- |
| `0x02a40001`, chroma and mappings | 8192 | FL2/FL3 chroma records; mappings adapted for the A7CR |
| `0x02a4000d`, matrices for three illuminants | 28672 | FL2/FL3 native matrices for three illuminants |

Merge the added Alpha 1 II records into the target camera's existing contents, preserving unit00/02 and the other looks. Use `WRITE41` for the two units, read both back completely with `READ40`, then issue one `SAVE3(02a4)`. Retention across a restart has been verified. Do not replace the camera's entire backup with the donor's backup.

### Separate photo-name fixes

Even when numeric modes19/20 and gamma48/49 are correct, an old string writer can still write `Standard`, leaving camera playback showing ST. Two additional windows are changed on top of the image-processing patch:

| CP file range (exclusive end) | Length | New behavior |
| --- | ---: | --- |
| `[0x148e7e,0x148f0c)` | 142 | MakerNote `0xb020`: mode19 -> FL2, mode20 -> FL3 |
| `[0x149e34,0x149ea4)` | 112 | SR2 root-level tag `0x7770` ColorMode: same mapping |

Each name occupies 16B: ASCII `FL2`/`FL3` followed by 13 NUL bytes. The second window stores these strings at file offsets `0x149e80/0x149e90`. External entry points `0x148ea0/0x148ea2` in the first window and the existing BW/Off branches are preserved. The two windows change 247 actual bytes and produce the CP name-fix stage described in section 1. Existing photos are not renamed automatically.

### Additional playback ColorMode fix

After the name fix, new photos still showed another inconsistency: `0xb020` contained FL2/FL3 and Tag9416 contained19/20 and48/49, but `0xb029 ColorMode` remained 0. The corresponding Alpha 1 II table uses22/23, and the A7CR AP already contains the matching icon branches. This field was therefore fixed separately, without changing the curves or color parameters that already produced visible image differences.

The expected input for the following table is **only the CP file after the name-fix stage**:

| CP file offset | Length | Before -> after / purpose |
| --- | ---: | --- |
| `0x148ef8` | 20 | Initially ten `00bf` pairs; replace with `152901d88a1c18e0002216e000bf00bf00bf00bf` to add mode19 ->22 and mode20 ->23 handling |
| `0x148f1e` | 2 | `07d8` -> `ebd8`, redirect the original out-of-range branch to the added handling |

Write the 20B helper before changing the 2B entry branch. The write ranges total 22B, with 12 bytes actually changed. The helper occupies a padding tail whose ownership was established in the previous stage. The original twenty-entry ColorMode table, other modes, gamma, color parameters, and file length are preserved. Verify the resulting complete file against the intended final candidate.

**`0x148ef8` lies inside the first window in the preceding subsection.** The current CP already includes this increment. Rewriting the entire old name-fix window would overwrite it. On-disk verification is complete, but correct playback still requires newly captured photos after a restart. The presence of matching AP icon branches alone does not establish successful playback.

## 4. Invalidating the old boot snapshot: existing interfaces and functions

An on-disk change does not necessarily load the new component immediately: an old WBI may restore the old process. Use the existing sysfs interfaces in the vendor kernel, preserve the selected `wbi4` backend, and do not edit physical sectors manually.

| Interface | Value to write / check | Purpose |
| --- | --- | --- |
| `/sys/power/image_mode` | The initial read must be exactly `2\n` | Confirm the applicable starting state |
| `/sys/power/image_dev` | Read `*wbi4\n` before and after | Preserve the selected backend |
| `/sys/power/image_drop` | Write ASCII `1` once, 1B | Invalidate the old WBI on the current backend |
| `/sys/power/image_mode` | Immediately write ASCII `0` once, 1B | Skip the traced normal WBI-generation path when this kernel exits |

The vendor kernel `vmlinux.bin` function chain, in file offsets, is `0x63e28` (image_drop store) -> `0x63dc0` -> `0x63ce8` (`__wbi_drop`). The image_mode store is at `0x63150`. These locations identify functions; they are not kernel addresses to overwrite.

The required order is **drop1 -> mode0**. If mode0 is set first, drop may return success without invalidating the old WBI. Each write must return exactly 1. Then verify mode0, the original backend, all closes, and a fresh `dropped wbi` log entry from this operation. mode0 affects only the current kernel, and a log entry is not independent readback from the storage medium. A normal restart is still required to verify that the new components are in use.

## 5. Operation sequence

The following is the dependency order used in this research. Read the current state before repeating any completed write. Do not rerun every step merely because it appears here. After a timeout or an unknown result, establish the outcome through readback first.

1. **Pin the inputs and make backups.** Back up the target AP, CP, and local parameters. Verify the firmware versions, expected original bytes, file identity, and power stability. Existing access to the camera's internal system and a local USB service are prerequisites; this document does not explain how to obtain that access.
2. **Assemble the complete result offline.** Check each stage's expected original bytes, ranges, lengths, and the final complete-file contents. Ensure an older window does not overwrite a later increment. The short-value tables are not a complete patch.
3. **Complete the CP code and metadata changes.** The result must include processing dispatch, both name fixes, and the `0xb029` fix. After writing, check window readback, fsync, complete-file comparison, and EOF to establish that the on-disk file exactly matches the candidate.
4. **Merge the CP color parameters.** Write the two target units, read them back completely, and save them while preserving the camera's other records. Code selects the parameters; the parameters supply the underlying color processing. Both are required.
5. **Connect AP selection and saved bindings.** Write the fixed AP windows, verify the complete file, and restore `/usr` to read-only. Use the native mechanism to migrate ordinary slots16/17 to look10/11 and save. Firmware initial values do not replace migration of existing saved bindings, and factory defaults do not replace current adjustment values.
6. **Invalidate the old snapshot.** Execute and verify the single drop1 -> mode0 sequence from section 4 to prevent an old WBI from restoring the old runtime state. Success here does not by itself prove that the new components are running.
7. **Restart normally.** Exit the USB service, confirm no requests remain in flight, disconnect USB, and power the camera off and on normally. Then check saved values, active settings, and basic shooting.
8. **Validate the results.** Reset FL2 and FL3 separately, inspect all eight defaults and their markers, and take new photos with fixed white balance and exposure. Check the image, numeric look identifiers, names, and camera playback, then test existing looks for regressions.

## 6. Validation and troubleshooting

| Symptom | Check first | Passing condition |
| --- | --- | --- |
| Only ten looks appear | Menu product flags and whether the new AP is running | Twelve base looks are visible |
| FL2/FL3 appear but are unavailable | The two selection capabilities and restrictions imposed by the current shooting settings | The looks can be selected under the intended shooting conditions |
| Selection works, but the image resembles ST | Saved and active slot bindings, AP-to-CP conversion, CP parameter dispatch, and unit contents | Requests and parameters select the intended modes, with the corresponding effect in new photos |
| The image changes, but playback still shows ST | New photos' `0xb020`, Tag9416, and `0xb029`; trace playback reads further if necessary | Numeric values, strings, and camera playback each verify correctly |
| Reset values or default markers are wrong | Factory slots16/17 and the native reset source | Both looks restore `0,0,0,1,0,4,3,1`, with the Fade default marker at +1 |
| The file verifies correctly, but nothing changes after a restart | WBI invalidation, loading of the new component, and saved settings overriding initial values | Verify on-disk contents, loading of the new component, and actual photo results separately |

Start validation with ordinary single-shot photos, then extend it to RAW/JPEG/HEIF, bursts, and other combinations of settings. When comparing images, keep the scene, exposure, white balance, and other processing settings as consistent as possible. A visible difference establishes that processing changed; it does not by itself prove an exact match to the Alpha 1 II. Validate metadata fixes with photos captured after the fix. Existing photos are not updated automatically.

At the time of this record, evidence covers file writes, ordinary slot values in use after a restart, CP parameter retention, and basic shooting. Validation of the latest defaults, names, and `0xb029` playback behavior remains outstanding. Memory-card recovery after a broken CP has not been tested. Instructions for retrying a failed normal update must not be treated as a guarantee that arbitrary damage is recoverable.

## 7. References and acknowledgements

The following repositories informed firmware extraction, protocol research, and metadata analysis. The FL2/FL3 firmware data used in this port was extracted from Sony A12 firmware as described in section 1.

| Repository | Role in this research |
| --- | --- |
| [YuuLuo/a7c2-service-preload](https://github.com/YuuLuo/a7c2-service-preload) | Reference for service authentication, Jiritsu path handling, and the preload execution chain. The upstream study targets ILCE-7CM2 Ver.2.01; A7CR behavior was investigated separately. |
| [ma1co/Sony-PMCA-RE](https://github.com/ma1co/Sony-PMCA-RE) | Reference for Sony USB/Senser protocols, native Backup operations, and camera settings mechanisms. |
| [ma1co/fwtool.py](https://github.com/ma1co/fwtool.py) | Sony firmware container unpacking and extraction reference. |
| [Jannchie/llr](https://github.com/Jannchie/llr) | Early Sony Creative Look and RAW color-data analysis, including offline reference material. |
| [olkham/pysonycam](https://github.com/olkham/pysonycam) | Sony PTP property and enum documentation consulted when investigating Creative Look support. |
| [exiftool/exiftool](https://github.com/exiftool/exiftool) | Sony MakerNote decoding and verification of CreativeStyle, PictureProfile, and ColorMode fields in captured photos. |

Credit belongs to the respective authors and contributors. Each upstream project retains its own license; see the linked repositories for their terms. These links document research sources and do not imply upstream endorsement of this port.

## 8. Downstream reproducible Creative Look workflow

An independent [agent-neutral A7CR tool extension](https://github.com/littleben/a7cr-creative-look-tools)
builds on these A7CR findings, injectiver's research source and QAQLeonard's DCP/
native-worker foundations. It provides Python/JSON project commands, shared
AGENTS.md instructions and English/Chinese READMEs. Firmware inputs, original
profiles, per-body calibration, backups and photographs stay local.

The downstream implementation reports one A7CR 2.01 on Apple Silicon macOS with
native FL2/FL3, a single-slot Classic Chrome adaptation, then incremental PRO
Neg. Std and ASTIA slots while retaining the first look. Full AP/CP and 68-name
resource readback, new-boot loaded AP checks, saved/native ST and new JPEG custom
markers were verified. The recent portrait-look samples are still life, not a
skin-tone or original-Fujifilm color-matching test. Complete physical restoration
has not been exercised.

The portable source interface supports explicit checkpoints, local fitting and
asset checks, per-body preparation, installation, read-only verification and
exact inverses. It does not turn these notes into a universal installer or alter
the existing outstanding validation. An ordinary-USB exit timeout is distinct
from an unknown write: only the completely evidenced case can proceed to a new
cold-start read-only verification, preserving the original failure history.
