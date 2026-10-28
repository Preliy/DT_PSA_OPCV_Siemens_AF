# RFID read/write head — P+F IQT1 Expert Mode

The TIA program drives an RFID read/write head with Pepperl+Fuchs'
**`IQT1-xx-IO-V1_ExpertMode_Basic`** block (V2.1). The head is an IQT1-xx-IO-V1 behind an IO-Link
master, and its process data is a 32-byte Expert Mode telegram. In the emulation unit the head is
simulated by `FB_RFID_IQT1` in `TIA_1/EmulationUnit/SIM_1/Library/RFID_IQT1/`.

**Status: behaviour model only.** The block exists in the library and compiles, but no function
group uses it yet. There are no telegram addresses and no twin device for it.

## Where the behaviour comes from

The telegram is derived from the **master side**, the P+F block, not from the P+F manual. Every
byte the model writes is one the block reads or tests. Where the block leaves a value open, the
model uses a value the block reads correctly. Those choices are listed under
[Open points](#open-points).

Built the Open Commissioning way, like the [lift drives](lift-drives.md):

- **Process data:** `pControl` / `pStatus` point into the PLCSIM image. It is bytes only, so
  nothing is swapped; the address and count words in the telegram are high byte first.
- **The tag:** the twin's **DataReader** says which tag is in front of the head, the same device
  type the cameras use. `Control.1` triggers a read, `Status.1` echoes it, and `StatusData` is the
  64-bit tag UID. A UID of 0, or no echo because Unity is not connected, means **no tag**.
  `bSimulateTag` + `nSimulatedUid` bypass the twin for testing.
- **The tag's memory:** one file per tag, written and read with OC_Core **`FB_FileIO`**. The file
  is reloaded on every command and never cached. Every head that shares the folder sees the same
  tags, so data written at one station is read back at the next.

## Telegram layout

32 bytes from the PLC (`%QB`), 34 to the PLC (`%IB`): 32 data bytes plus the PQI at byte 32. The
Murr IO-Link master sends the PQI as a WORD, which is why the block reads 34.

| Byte | PLC → head (request) | Head → PLC (first response frame) |
|---:|---|---|
| 0 | handshake bits | handshake bits |
| 1 | frame length | frame length = last used index + 1 |
| 2 | fragment counter | frames still to come, 0 = last |
| 3–4 | telegram length | telegram length = 4 + data bytes |
| 5 | command | command echo |
| 6–7 | byte address (Change Tag: [6] = tag type) | [6] status, then data from [7] |
| 8–9 | byte count | data |
| 10–31 | write data, at most 22 bytes | data — 25 bytes in frame 1 |
| 32 | — | PQI, `stConfig.nPQI` (shown by the block, not evaluated) |

A read longer than 25 bytes is fragmented. Each further frame carries `[1]` frame length, `[2]`
counter, and 29 data bytes in `[3..31]`. That matches the block's `ReadData` offsets 0, 25, 54,
83 and so on.

### Handshake (byte 0)

| Bit | Name | Rule |
|---|---|---|
| `16#80` | Delete | The PLC toggles it in its init routine. The head drops whatever is in flight and mirrors it |
| `16#40` | UpdateMaster | **Equal** on both sides means a request is waiting. The head takes it and makes them differ, which permits the next request. While busy, the head leaves them equal and the block retries |
| `16#20` | UpdateSlave | The head makes them **equal** to publish a frame. The PLC writes the inverse back every scan, so **different** means consumed |

**The head publishes one frame per acknowledgement**, with the data and the toggle in the same
cycle. The emulation task runs at 1 ms and PLCSIM is far slower: a frame written before the PLC
has consumed the last one is lost.

## Commands

| Code | Command | Answer |
|---|---|---|
| `16#04` | Change Tag (the block's init) | `00`. Any tag type is accepted |
| `16#02` | Quit | `00` |
| `16#10` | Single Read 4-byte block | `00` + data, fragmented. `04` if the range leaves the tag memory or exceeds 192 bytes. `05` without a tag |
| `16#40` | Single Write 4-byte block | `00` once the tag file is written. `04` if the range leaves the tag memory or exceeds 22 bytes. `05` without a tag |
| `16#01` | Single Read Fixcode | `00` + the 8 UID bytes, most significant first. `05` without a tag |
| `16#19` `16#1A` `16#1D` | Enhanced Read / Write / Read Fixcode | `04`: not modelled |
| other | Special (tunnelled) commands | `04`: not modelled |

Status `00` = OK, `04` = error, `05` = no data carrier. These are the only codes the block
evaluates. A no-tag answer is exactly frame length 7, telegram length 4 and status `05`, which is
the pattern the block tests for.

## Tag files

| Parameter | Default |
|---|---|
| Folder `stConfig.sDirectory` | `C:\TwinCAT\RFID\`. Only the last level is created; the parent must exist |
| File | `Tag_<UID as 16 hex digits>.bin`, e.g. `Tag_E004015012345678.bin` |
| Content | the tag's raw user memory, `stConfig.nMemorySize` bytes (2048, at most 8192) |
| A tag seen for the first time | starts with zeroed memory. `FB_FileIO.Read` creates a missing file from the cleared buffer |

To give a pallet a preset memory, write its file before the run, or delete it to start blank.
A file error (`nFileErrorId`) answers `04`.

## Timing

| Parameter | Default | Meaning |
|---|---|---|
| `tAccessTime` | 30 ms | modelled air time per command, after the tag is known |
| `tSettle` | 20 ms | DataReader: hold after the trigger echo before sampling the UID |
| `tAcquireTimeout` | 500 ms | DataReader: no answer in this time means no tag |

These are modelled, not measured. They must stay well inside the block's `I_T_Timeout` (10 s by
default).

## Not modelled

Enhanced (continuous) commands, Special commands, tag-type behaviour, alignment of the address to
4-byte blocks, RSSI, and the timing statistics the block records (`Time_Status_00` and so on are
still filled by the block from its own clock).

## Open points

- **Unverified against the P+F manual:** the frame length of continuation frames, the direction of
  the fragment counter, and the status a real head gives for an out-of-range address. The block
  reads the model's choices correctly; a real head may differ.
- **Possible bug in the P+F block (V2.1):** frame 1 is copied with
  `MOVE_BLK(Resp_Telegram_1[7], COUNT := 26)` from a `0..31` array, which is one byte past its
  end. If TIA refuses that copy (`ENO = 0`), `ReadData[0..24]` stays empty. Check this in the first
  PLCSIM test.
- **A continuation frame of exactly 4 data bytes** has frame length 7. If its data bytes `[4]` and
  `[6]` happen to be `16#04` and `16#05`, the block mistakes it for a no-tag answer. This is
  harmless for any other length.
- **Next step:** instance the head in a function group, assign the telegram addresses from the TIA
  hardware configuration, and add a DataReader for it to the twin.
