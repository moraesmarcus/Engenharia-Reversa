# PES 6 Editor 7.0 - reverse notes

## Main finding

The player and team export formats used by the editor are not container formats.
They are raw copies of the internal 124-byte player record from the PES6 option
file after the editor has loaded/decrypted it.

## `.plrv` player export

- Size: exactly `124` bytes.
- Content: one raw player record.
- No header, no checksum, no player id, no file signature.
- Export path in the JAR: `editor.PlayerDialog$4.actionPerformed`.
- Import path in the JAR: `editor.PlayerDialog$5.actionPerformed`.

Import behavior:

```text
read 124 bytes from .plrv
if target player index >= 32768:
    copy to OptionFile.data[14288 + (index - 32768) * 124]
else:
    copy to OptionFile.data[37116 + index * 124]
```

The current `Morientes.plrv` file is one record:

```text
00 Morientes | shirt: MORIENTES
```

## `.tmrv` team export

- Size: `124 * player_count` bytes.
- Content: concatenated raw player records.
- No header, no team name, no team id, no squad layout, no checksum.
- Export path in the JAR: `editor.TransferPanel$2.actionPerformed`.
- Import path in the JAR: `editor.TransferPanel$3.actionPerformed`.

The current `Real Madrid.tmrv` is `1736` bytes:

```text
1736 / 124 = 14 player records
```

Records found:

```text
00 Casillas   | shirt: C A S I L L A S
01 Karembeu   | shirt: K A R E M B E U
02 Hierro     | shirt: H I E R R O
03 Salgado    | shirt: S A L G A D O
04 R. Carlos  | shirt: R. C A R L O S
05 Redondo    | shirt: R E D O N D O
06 Guti       | shirt: J. M. G U T I
07 Geremi     | shirt: G E R E M I
08 Savio      | shirt: S  A  V  I  O
09 Raul       | shirt: R  A  U  L
10 Anelka     | shirt: A N E L K A
11 Cuadrado   | shirt: C U A D R A D O
12 Helguera   | shirt: H E L G U E R A
13 Campo      | shirt: C  A  M  P  O
```

Team import behavior:

```text
record_count = file_length / 124
read record_count * 124 bytes
count existing non-empty player slots in the currently selected team

if imported record_count <= existing non-empty count:
    overwrite the first imported record_count current squad players
else:
    overwrite only existing non-empty count current squad players
```

Important: `.tmrv` import does not bring a team identity or roster slot table. It
imports into whichever team is selected in the editor and overwrites that team's
current players in order.

## Team slot limits used by the editor

The team export/import screen sets an upper slot limit from the selected team
number:

```text
team < 73   -> 23 slots
team == 73  -> 14 slots
team > 73   -> 32 slots
```

It counts non-empty players within that limit and writes only those player
records. This is why a `.tmrv` file can be much smaller than a full 32-player
club squad.

## Player record structure confirmed so far

Each player record is `124` bytes.

```text
0x00..0x1F  player display name, UTF-16LE, max 15 chars plus 00 00 terminator
0x20..0x37  shirt/commentary-style name field, single-byte text/padded bytes
0x38..0x7B  packed player data: abilities, positions, appearance, flags, etc.
```

The editor's `Player.setName()` confirms the display name field:

```text
UTF-16LE bytes are copied into the first 32 bytes.
Maximum accepted Java string length is 15.
```

The rest of the record is copied as opaque packed data by the `.plrv`/`.tmrv`
import and export paths.

## Option file offsets

The editor works against `OptionFile.data`, which is the loaded/decrypted option
file buffer.

Known player table offsets:

```text
normal players: 37116 + index * 124
edited players: 14288 + (index - 32768) * 124
```

Index handling observed in `editor.Player`:

```text
index == 0             -> <empty>
0 < index < 5000       -> normal player table
32768 <= index <= 32951 -> edited player table
```

The supplied `KONAMI-WIN32PES6OPT` file is `1191936` bytes, matching the editor's
internal option-file data length.

## Practical consequences

- To export one player manually, copy 124 bytes from the correct player record
  offset in the decrypted option data.
- To import one player manually, overwrite 124 bytes at the target player's
  record offset.
- To export a team manually, concatenate the 124-byte records of the players in
  the selected squad order.
- To import a team manually, split the `.tmrv` into 124-byte records and copy
  them over the selected team's current squad players in order.
- A `.tmrv` alone cannot recreate squad membership in an arbitrary empty team,
  because it does not contain player indices, team id, shirt numbers, formation,
  or roster slot metadata.

